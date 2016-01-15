Automaintainer
==============

    fs = require 'fs'
    fetch = require 'node-fetch'

Config
------

    USER = 'automaintainer'
    PASS = fs.readFileSync('TOKEN').toString('utf-8').trim()
    AUTH = new Buffer('automaintainer:'+PASS).toString('base64')
    DEFAULTCONFIG = require './default.automaintainer.json'

    HEADERS =
      'User-Agent': 'Automaintainer'
      Authorization: "Basic #{AUTH}"

Helpers
-------

A mini caching wrapper around fetch so we don't hit GitHub too hard.

    cache = {}
    EXPIRY = 1000*60*60*24
    cacheable = (url, opts) -> !opts.method or opts.method is 'GET'
    cachedfetch = (url, opts) ->
      if cacheable(url, opts) and cache[url]
        opts.headers ||= {}
        opts.headers["If-None-Match"] = cache[url].etag

      fetch url, opts
      .then (result) ->
        if result.status is 304
          # console.log "CACHE HIT", url
          return cache[url].data
        resdata = if result.headers.get('content-type')?.match /^application\/json/ then result.json() else result.text()
        resdata.then (data) ->
          ok = (result.status / 100 | 0) is 2
          if ok and etag = result.headers.get('etag')
            # console.log "CACHE SAVED", url
            clearTimeout cache[url].timeout if cache[url]
            timeout = setTimeout (-> delete cache[url]), EXPIRY
            cache[url] = {etag, data, timeout}
          else if cache[url]
            # console.log "CACHE PURGED", url
            clearTimeout cache[url].timeout
            delete cache[url]

          if ok then data else throw new Error(data.message or data)


Mini API client. Takes an endpoint, prefixes it and does some JSON stuff.

    makeAPI = (base="https://api.github.com") ->
      api = (endpoint, opts={}) ->
        endpoint = "#{base}/#{endpoint}" unless endpoint.match /^https:/
        opts.headers ||= {}
        opts.headers[k] = v for k, v of HEADERS
        cachedfetch endpoint, opts

      api.get = (endpoint, opts={}) ->
        api endpoints, opts

      api.post = (endpoint, body, opts={}) ->
        opts.method ||= 'POST'
        opts.body ||= JSON.stringify(body)
        opts.headers ||= {}
        opts.headers['Content-Type'] ||= 'application/json'
        api endpoint, opts

      api.put = (endpoint, body, opts={}) ->
        opts.method ||= 'PUT'
        api.post endpoint, body, opts

      api.patch = (endpoint, body, opts={}) ->
        opts.method ||= 'PATCH'
        api.post endpoint, body, opts

      api.with = (newbase) ->
        makeAPI "#{base}/#{newbase}"

      api

    api = makeAPI()


Config
------

We keep all our config in an automaintainer.json in the automaintainer branch
of each repo. We don't really do git things properly, so this is a bit racey.
Hopefully it shouldn't get updated often enough to matter in practice.

    getConfigSlow = (repo) ->
      api 'repos/#{repo}/git/refs/heads/automaintainer'
      .then (ref) ->
        throw new Error "No matching ref" unless ref.ref
        api ref.object.url
      .then (commit) -> api commit.tree.url
      .then (tree) ->
        for entry in tree.tree when entry.path is 'README.md'
          console.log "ENTRY", entry
          return api entry.url, headers: Accept: 'application/vnd.github.v3.raw'
        throw new Error "No matching tree entry"
      .then (data) ->
        console.log "DATA", data
        data

    getConfig = (repo) ->
      console.log "#{repo}: loading config"
      api "https://raw.githubusercontent.com/#{repo}/automaintainer/automaintainer.json"
      .then (data) ->
        config = JSON.parse data
        console.log "config", config
        config

    updateConfig = (repo, message, conf) ->
      data = JSON.stringify conf, null, 2
      gitAPI = api.with "repos/#{repo}/git"
      tree =
        path: "automaintainer.json"
        mode: "100644"
        type: "blob"
        content: data

      gitAPI "refs/heads/automaintainer"
      .catch -> null
      .then (ref) ->
        # console.log "ref", ref
        head = ref?.object?.sha

        gitAPI.post "trees", tree: [tree]
        .then (tree) ->
          gitAPI.post "commits", {message, tree: tree.sha, parents: if head then [head]}

        .then (commit) ->
          if head
            gitAPI.patch "refs/heads/automaintainer", sha: commit.sha
          else
            gitAPI.post "refs", ref: "refs/heads/automaintainer", sha: commit.sha
        .then (result) ->
          console.log "#{repo}: committed new config"
          conf

    initConfig = (repo) ->
      console.log "#{repo}: initialising config"
      updateConfig repo, "Initial automaintainer config", DEFAULTCONFIG


Pull requests
-------------

We merge pull requests after config-determined (`rules.accept_pull.votes`)
number of votes. Votes are made with a `:+1` in a comment by any collaborator
or voter.

    acceptPull = (repo, pull, voted) ->
      console.log "#{repo}: accepting #{pull.number} with #{voted.length} votes"

      api.put pull.url + '/merge',
        sha: pull.head.sha
        commit_message: "merged by votes from #{voted.join ', '}"


    updatePull = (repo, pull, config) ->
      canVote = {}
      canVote[k] = true for k in config.voters or []
      canVote[k] = true for k in config.collaborators or []

      api pull.comments_url #"repos/#{repo}/issues/#{pull.number}/comments"
      .then (comments) ->
        voted = []
        #console.log "comments", comments, canVote[comments[0].user.login], comments[0].body.match ':+1:'
        for comment in comments when canVote[comment.user.login] and comment.body.indexOf ':+1:' > -1
          voted.push comment.user.login
          if voted.length >= config.rules.accept_pull.votes
            return acceptPull repo, pull, voted
        return

    updatePulls = (repo, config) ->
      console.log "#{repo}: updating pull requests"
      return unless config.rules?.accept_pull
      api "repos/#{repo}/pulls"
      .then (pulls) ->
        Promise.all (updatePull repo, pull, config for pull in pulls)

Members
-------

Update collaborators from GitHub's internal collaborators list and update
voters based the contents of `rules.voter.pulls`, which is a number indicating
how many merged pull requests are required to get vote access.

    updateCollaborators = (repo, config) ->
      console.log "#{repo}: updating collaborators"
      repoAPI = api.with "repos/#{repo}"
      repoAPI 'collaborators'
      .then (collaborators) ->
        collabs = {}
        collabs[k] = true for k in config.collaborators or []
        newcollabs = []
        for {login} in collaborators
          newcollabs.push login if !collabs[login]
          collabs[login] = true

        return if newcollabs.length is 0

        config.collaborators = (k for k of collabs).sort()
        updateConfig repo, "New collaborators: #{newcollabs.join ', '}", config

    getMergedPullsByUser = (repo) ->
      api "repos/#{repo}/pulls?state=closed"
      .then (pulls) ->
        # console.log "pulls", pulls
        users = {}
        for pull in pulls when pull.merged_at
          users[pull.user.login] ||= 0
          users[pull.user.login]++
        users

    updateVoters = (repo, config) ->
      return unless config.rules?.voter?.pulls
      console.log "#{repo}: updating voters"
      getMergedPullsByUser repo
      .then (userPulls) ->
        voters = {}
        voters[k] = true for k in config.voters or []
        newvoters = []
        for user, pulls of userPulls when pulls >= config.rules.voter.pulls
          newvoters.push user if !voters[user]
          voters[user] = true

        return if newvoters.length is 0

        config.voters = (k for k of voters).sort()
        updateConfig repo, "New voters: #{newvoters.join ', '}", config


High-level update
-----------------

Just loop over all the repos we have commit access to and run our individual helpers.

    updateRepo = (repo) ->
      getConfig repo
      .catch (err) -> if err.message is 'Not Found' then initConfig repo else throw err
      .then (config) ->
        console.log "#{repo}: updating"
        Promise.resolve()
        .then -> updateVoters repo, config
        .then -> updateCollaborators repo, config
        .then -> updatePulls repo, config

    updateRepos = ->
      console.log "starting update run"
      api "user/repos"
      .then (repos) ->
        Promise.all (updateRepo repo.full_name for repo in repos when repo.permissions.push)


Ghetto cron
-----------

We run our main loop every 60 seconds, but exponentially backoff if things go
bad.

    goodRepeat = 1000*60
    badRepeat = 1000*60

    reUpdate = ->
      updateRepos()
      .then ->
        badRepeat = 1000*60
        console.log "timer: retrying normally after", goodRepeat
        setTimeout reUpdate, goodRepeat
      .catch (err) ->
        console.error "something screwed up", err
        badRepeat *= 2
        console.log "timer: backing off to", badRepeat
        setTimeout reUpdate, badRepeat
    reUpdate()