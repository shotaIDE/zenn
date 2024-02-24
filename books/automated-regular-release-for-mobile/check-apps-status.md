---
title: "データストアを見て前のリリースの審査中ではないかを判定する"
---

Google スプレッドシートの特定のセルを読み取り、前回のリリースが審査中でないかを判定します。

事前準備として、以下のドキュメントを元に GCP のプロジェクトでサービスアカウントを作成して、必要な権限を付与した上で、キーを JSON 形式でダウンロードしておきます。

https://github.com/gimite/google-drive-ruby/blob/master/doc/authorization.md#service-account

Google スプレッドシートを用意し、Google スプレッドシートの編集画面にてサービスアカウントのメールアドレスに編集権限を付与します。

以下のように、Fastlane で Google スプレッドシートを操作するためのライブラリを依存関係として追加します。

```ruby:Gemfile
source 'https://rubygems.org'

gem 'google_drive'
```

サービスアカウントのキー JSON ファイルを `fastlane/spreadsheet-service-account-key.json` に格納しておきます。

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

以下のように、Fastlane のスクリプトを GitHub Actions で実行します。

```yaml
# ...

jobs:
  check-apps-status:
    name: Check apps status
    runs-on: ubuntu-latest
    outputs:
      is-release-available: ${{ steps.check-apps-currently-released.outputs.is_released == 'true' }}
    steps:
      - uses: actions/checkout@v4
      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - name: Check apps currently released
        id: check-apps-currently-released
        run: |
          if bundle exec fastlane check_mobile_apps_are_currently_released; then
            # Fastlaneのスクリプトが成功した場合
            echo "is_released=true" >> $GITHUB_OUTPUT
          else
            # Fastlaneのスクリプトが失敗した場合
            echo "is_released=false" >> $GITHUB_OUTPUT
          fi
  next-job:
    needs: check-apps-status
    if: needs.check-apps-status.outputs.is-release-available
    # ...
```

リリース済みであることが確認できたら GitHub Actions の次のジョブを実行するようにしています。
