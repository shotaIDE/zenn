---
title: "ライブラリ更新の PR が自動で作成されるようにする"
free: true
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

アプリでは、依存ライブラリのバージョン制約は、最新版のみに追従していく方針になることが多いです。

一方で、Flutter に対する dependabot は現在、そうしたバージョン制約が取りづらくなっているため、独自でパッチを当てています。

また、dependabot による PR の作成時刻をある程度ずらしています。
これにより、片方がマージされた際に他の PR がマージ対象のブランチの最新コミットから遅れているためにマージできない状況を防ぎます。
