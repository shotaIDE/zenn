---
title: "PR のマージ必須条件が満たされた場合、自動でマージされるようにする"
free: true
---

Mergify を利用して dependabot が提出した PR は、必須条件が満たされたら自動でマージされるようにします。

https://mergify.com/

例えば、以下のように Mergify の設定を定義します。

```yaml:.github/mergify.yml
pull_request_rules:
  - name: Automatic merge PRs from dependabot
    conditions:
      - "author = dependabot[bot]"
    actions:
      merge:
        method: merge
```

`author = dependabot[bot]"` と指定しておくことで dependabot による PR のみが自動マージの対象になります。
