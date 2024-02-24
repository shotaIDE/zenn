---
title: "スケジュールで定期的に自動リリースの処理をキック"
free: true
---

GitHub Actions のスケジュールトリガーを利用します。

例えば、以下のように設定します。

```yaml:.github/workflows/regular-release.yml
name: Regular release

on:
  schedule:
    - cron: '0 0 * * 1' # 毎週月曜日の AM 9:00 (JST)

jobs:
  # ...
```

https://docs.github.com/ja/actions/using-workflows/events-that-trigger-workflows#schedule
