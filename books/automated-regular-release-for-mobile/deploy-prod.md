---
title: "商用アプリをデプロイ＆審査提出"
free: true
---

以下のように、Fastlane でスクリプトを作成します。

```ruby:Fastfile
lane :deploy_and_submit_for_review do
  # ...
  # ここでビルドとアプリ審査提出を行う
  # ...
end
```
