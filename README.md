Automaintainer
==============

Automaintainer automatically maintains your GitHub projects. Just add the automaintainer user as a collaborator. It will add a new `automaintainer` branch with an `automaintainer.json` in it. You then need to set up some rules. I suggest:

```json
"rules": {
  "voter": {
    "pulls": 1
  },
  "accept_pull": {
    "votes": 2
  }
}
```

Because those are the only rules that are currently supported. That means anyone with 1 successful pull request becomes a voter, and any pull request with 2 votes is merged automatically. You can vote by commenting with `:+1:` on an issue.

If you want to mess around with it, try the [automaintainer-test](https://github.com/sgentle/automaintainer-test) repo.