---
title: "fastlaneで利用するRubyのバージョンを固定する"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["fastlane", "ios", "android"]
published: false
---

ビルドツールで fastlane を利用しています。

https://fastlane.tools/

fastlane では Ruby を利用しています。
リポジトリに `.ruby-version` ファイルをコミットしておき、この Ruby のバージョンを利用するようにしました。

```text:.ruby-version
3.3.5
```

:::message
`.ruby-version` ファイルは Ruby のバージョン管理システムの rbenv で利用されるファイルです。
https://github.com/rbenv/rbenv#readme

fastlane のコマンドや CI/CD のワークフローは以下のようにして動作させています。

- rbenv を利用する
- 他の Ruby のバージョン管理システムにこのファイルの Ruby バージョンを用いて動作させる
  :::

これにより、開発者の環境によって動かないなどの差分が出ることや、CI/CD 上で突然動かなくなるなどのリスクを低減しました。

これにより、以下のようなことができるようになりました

> Ruby のバージョンを更新するメンテナンスを行うタイミングで、CI/CD のワークフローが動かなくなることが分かり、原因調査を行うことができた。
