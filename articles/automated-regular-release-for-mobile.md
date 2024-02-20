---
title: "完全に自動でメンテナンスリリースされていくモバイルアプリを作ってみる実験"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ios", "android"]
published: false
---

# はじめに

個人アプリ開発では、確保できるプライベートの時間の不安定さにより開発できない期間が発生しやすいです。
開発できない期間が空くと、再開した時に利用しているライブラリの多くが最新ではなくなっていて、辛いライブラリのアップデート作業が発生します。

開発できない期間でも、ライブラリ更新とそのアップデートリリースを自動化できれば、再開した際の手間を減らすことができます。
また、ユーザーに対してメンテナンスされているアプリであるという印象を与えることができます。

本記事では、完全に自動でライブラリ更新のアップデートがリリースされていくモバイルアプリを作ってみる方法を紹介します。

# 概要

ライブラリ更新を PR 作成からマージまで自動化しておき、メインのブランチにライブラリ更新が定期的にマージされるようにしておきます。

上記とは別サイクルの自ら定義した頻度で、自動リリースの処理を動かします。
これには、ビルドとアプリ審査提出、審査が通ったら公開という一連の流れが含まれます。

この際、各所で静的解析や自動テストのチェックを入れることで、品質を担保しておきます。

また、アプリ審査状況を保持するデータストアを用意し、アプリ審査中に新規でアプリ審査を提出してしまわないようにします。

# 前提

アプリ公開時のリリースノートは以下のような固定の文言を定義しておき、それを全てのリリースで適用するという前提です。

```text
細かいバグ修正とパフォーマンスの向上を行いました。
```

# 詳細の方針

概要で示した通り、大きく分けて以下の 2 つのサイクルがあります。

- メインブランチにライブラリ更新が定期的にマージされるようにしておく
- 自ら定義した頻度で自動リリースの処理を動かす

## メインブランチにライブラリ更新が定期的にマージされるようにしておく

以下の要素を組み合わせて、ライブラリ更新が定期的にマージされるようにしておきます。

- ライブラリ更新の PR が自動で作成されるようにする
- 静的解析と単体テストがパスすることを PR のマージに必須の条件とする
- PR のマージ必須条件が満たされた場合、自動でマージされるようにする

## 自ら定義した頻度で自動リリースの処理を動かす

以下のようにフローを組みます。

![](/images/automated-regular-release-for-mobile/automated-release-flow.png)

以下の要素を組み合わせています。

- スケジュールで定期的に自動リリースの処理をキック
- データストアを見て前のリリースの審査中ではないかを判定する
- 前回のリリースから差分があるかを判定する
- E2E テスト実行
- データストアにリリースの審査中とマーク
- 商用アプリをデプロイ＆審査提出
- 審査が通り公開された際に、データストアにリリース済みとマーク

トリガーの頻度は変数なので、どの程度の頻度でリリースしてもいいかを元に決定します。

# 各要素の利用技術

以下で、各要素の具体的な利用技術を簡単に紹介します。

前提として、GitHub 上で開発し、CI/CD の基盤として GitHub Actions を利用しています。

各種自動スクリプトは Fastlane を利用して組んでいます。

https://fastlane.tools/

また、データストアとしては Google スプレッドシートを利用しています。

## ライブラリ更新の PR が自動で作成されるようにする

dependabot を使います。

https://docs.github.com/ja/code-security/dependabot/dependabot-version-updates/about-dependabot-version-updates

## 静的解析と単体テストがパスすることを PR のマージに必須の条件とする

PR が提出されたら GitHub Actions で静的解析や単体テストが実行されるようにしておきます。

GitHub のブランチ保護ルールにて、これらの静的解析や単体テストをマージの必須条件としておきます。

https://docs.github.com/ja/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/managing-a-branch-protection-rule

ライブラリのアップデートで、アプリ内で利用しているメソッドが利用不可になったり deprecated になった場合は、ここで自動サイクルが止まります。

## PR のマージ必須条件が満たされた場合、自動でマージされるようにする

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

## スケジュールで定期的に自動リリースの処理をキック

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

## データストアを見て前のリリースの審査中ではないかを判定する

Google スプレッドシートの特定のセルを読み取り、前回のリリースが審査中でないかを判定します。

事前準備として、以下のドキュメントを元に GCP のプロジェクトでサービスアカウントを作成して、必要な権限を付与した上で、キーを JSON 形式でダウンロードしておきます。

https://github.com/gimite/google-drive-ruby/blob/master/doc/authorization.md#service-account

Google スプレッドシートを用意し、Google スプレッドシートの編集画面にてサービスアカウントのメールアドレスに編集権限を付与します。

以下のように、Fastlane で Google スプレッドシートを操作するためのライブラリを依存関係として追加します。

```ruby:Gemfile
source 'https://rubygems.org'

gem 'google_drive'
```

以下のように Fastlane でスクリプトを作成します。

```ruby:Fastfile
require 'google_drive'

lane :check_mobile_apps_are_currently_released do
  target_spreadsheet_id = 'xxxx' # スプレッドシートのURLの https://docs.google.com/spreadsheets/d/xxxx/edit における xxxx の部分
  sheet_name = '審査ステータス'
  target_range = 'A1'
  released_value = 'iOS と Android 両方ともリリース済み'

  service_account_key_json = File.read('spreadsheet-service-account-key.json')
  service_account_key = StringIO.new(service_account_key_json)
  session = GoogleDrive::Session.from_service_account_key(service_account_key)

  spreadsheet = session.spreadsheet_by_key(target_spreadsheet_id)
  sheet = spreadsheet.worksheet_by_title(sheet_name)
  latest_status = sheet[target_range]

  raise '現在のステータスがリリース済みではありません！' unless latest_status == released_value

  UI.success '現在のステータスはリリース済みです！'
end
```

## 前回のリリースから差分があるかを判定する

リリース時に付与される Git のタグを元に、最新コミットまでの差分を確認し、差分がある場合はリリース作業を進めます。

これには、GitHub Actions の Changed Files というアクションを利用しています。

https://github.com/marketplace/actions/changed-files

## リリース作業開始時にデータストアを更新

Google スプレッドシートに、リリース作業開始を記録します。

## E2E 自動テスト

Maestro Cloud とその GitHub Actions を利用しています。

https://cloud.mobile.dev/

## 審査完了後のリリース通知による、データストア更新

リリース完了のメール通知が来た際に、Google スプレッドシートを更新します。

Zapier を利用しています。

## データストア更新時に、iOS / Android の両方ともリリースされたかを判定

Google スプレッドシートの変更時に発動する、Apps Script を利用しています。

iOS と Android 両方ともリリース通知が揃った場合に、スプレッドシートを更新します。
