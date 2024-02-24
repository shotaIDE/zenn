---
title: "ライブラリ更新の PR が自動で作成されるようにする"
---

dependabot を使います。

https://docs.github.com/ja/code-security/dependabot/dependabot-version-updates/about-dependabot-version-updates

以下のように設定しておきます。

```yaml:.github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "gradle"
    directory: "/android/"
    open-pull-requests-limit: 1
    schedule:
      interval: "daily"
      time: "06:00"
      timezone: "Asia/Tokyo"
  - package-ecosystem: "pub"
    directory: "/"
    open-pull-requests-limit: 1
    schedule:
      interval: "daily"
      time: "07:00"
      timezone: "Asia/Tokyo"
    # By default, Flutter repository is determined as a library and only version
    # constraint updates are made, so change the versioning strategy.
    # See https://github.com/dependabot/dependabot-core/issues/4979
    versioning-strategy: "increase"
```
