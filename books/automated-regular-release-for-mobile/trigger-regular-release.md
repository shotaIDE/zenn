---
title: "スケジュールで定期的に自動リリースの処理をキック"
free: true
---

GitHub Actions のスケジュールトリガーを利用します。

以下のように設定します。

```yaml:.github/workflows/regular-release.yml
name: Regular release

on:
  schedule:
    - cron: '0 0 * * 1' # 毎週月曜日の AM 9:00 (JST)

jobs:
  # ...
```

スケジュールの頻度は、定期リリースは最大どの頻度で行われていいかという要件に合わせて設定します。

最大週 1 回に抑えたい場合は、上記のように週 1 回に設定します。
毎日リリースしても問題ない場合は、1 日 1 回に設定します。

https://docs.github.com/ja/actions/using-workflows/events-that-trigger-workflows#schedule
