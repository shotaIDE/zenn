---
title: "実装時の前提とセットアップ"
---

以降のチャプターでは、実際に実装した際のコードを紹介していきます。
その際の前提とセットアップについて以下に記載します。

# アプリ

Flutter を利用して iOS と Android のアプリを実装する前提です。
そのため、iOS と Android の共通リポジトリが 1 つあります。

# CI/CD

前提として、GitHub 上で開発し、CI/CD の基盤として GitHub Actions を利用しています。

# スクリプト

各種自動スクリプトは Fastlane を利用して組んでいます。

https://fastlane.tools/

# データストア

データストアとしては Google スプレッドシートを利用しています。

以下のドキュメントを元に GCP のプロジェクトでサービスアカウントを作成して、必要な権限を付与した上で、キーを JSON 形式でダウンロードしておきます。

https://github.com/gimite/google-drive-ruby/blob/master/doc/authorization.md#service-account

Google スプレッドシートを用意し、Google スプレッドシートの編集画面にてサービスアカウントのメールアドレスに編集権限を付与します。

以下のように、Fastlane で Google スプレッドシートを操作するためのライブラリを依存関係として追加します。

```ruby:Gemfile
source 'https://rubygems.org'

gem 'google_drive'
```

サービスアカウントのキー JSON ファイルをダウンロードしておきます。
