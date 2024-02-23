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

## 前回のリリースから差分があるかを判定する

リリース時に付与される Git のタグを元に、最新コミットまでの差分を確認し、差分がある場合はリリース作業を進めます。

これには、GitHub Actions の Changed Files というアクションを利用しています。

https://github.com/marketplace/actions/changed-files

```yaml
# ...

check-unreleased-diff:
  name: Check some diffs exist related to app
  runs-on: ubuntu-latest
  outputs:
    has-diff-related-to-app: ${{ steps.check-related-files.outputs.any_changed == 'true' }}
  steps:
    - uses: actions/checkout@v4
      with:
        ref: ${{ github.head_ref }}
        fetch-depth: 0
    - name: Get latest RC tag name
      id: get-latest-rc-tag-name
      run: |
        latest_rc_tag_name="$(git describe --tags --abbrev=0 --match 'rc/*')"
        echo "Latest RC tag name: $latest_rc_tag_name"
        echo "tag-name=$latest_rc_tag_name" >> $GITHUB_OUTPUT
    - name: Check latest release is not based on latest commit
      id: check-latest-release-is-not-based-on-latest-commit
      run: |
        latest_release_commit_sha="$(git rev-list -n 1 HEAD)"
        latest_rc_tag_commit_sha="$(git rev-list -n 1 ${{ steps.get-latest-rc-tag-name.outputs.tag-name }})"
        if [ "$latest_release_commit_sha" = "$latest_rc_tag_commit_sha" ]; then
          echo "Latest release is based on the latest commit."
          echo "result=false" >> $GITHUB_OUTPUT
        else
          echo "Latest release is not based on latest commit."
          echo "result=true" >> $GITHUB_OUTPUT
        fi
    - name: Check related files
      id: check-related-files
      uses: tj-actions/changed-files@v42
      if: ${{ steps.check-latest-release-is-not-based-on-latest-commit.outputs.result == 'true' }}
      with:
        base_sha: ${{ steps.get-latest-rc-tag-name.outputs.tag-name }}
        files: |
          .maestro/**
          android/**
          ios/**
          lib/**
          test/**
          .tool-versions
          .xcode-version
          pubspec.lock
          pubspec.yaml
        sha: "HEAD"
```

## E2E テスト実行

Maestro Cloud とその GitHub Actions を利用しています。

https://cloud.mobile.dev/

GitHub Actions の Secrets における `MAESTRO_CLOUD_API_KEY` に、Maestro Cloud の API キーを登録しておきます。

```yaml
# ...

e2e-test-ios:
  name: E2E test iOS app
  runs-on: macos-14
  steps:
    - uses: actions/checkout@v4
    - name: Setup Flutter
      uses: ./.github/actions/setup-flutter
    - name: Setup Xcode
      uses: ./.github/actions/setup-xcode
    - name: Setup CocoaPods
      uses: ./.github/actions/setup-cocoapods
    - name: Setup Ruby
      uses: ./.github/actions/setup-ruby
    - name: Build iOS dev app
      run: bundle exec fastlane ios build_dev_for_simulator
    - uses: mobile-dev-inc/action-maestro-cloud@v1.8.1
      with:
        api-key: ${{ secrets.MAESTRO_CLOUD_API_KEY }}
        app-file: "build/ios/iphonesimulator/Runner.app"
        ios-version: 17
        device-locale: ja_JP
```

## データストアにリリースの審査中とマーク

Google スプレッドシートに、リリース作業開始を記録します。

以下のように、Fastlane でスクリプトを作成します。

```ruby:Fastfile
lane :update_mobile_apps_status_to_in_release_process do
  Dotenv.load '.env'

  target_spreadsheet_id = 'xxxx' # スプレッドシートのURLの https://docs.google.com/spreadsheets/d/xxxx/edit における xxxx の部分
  sheet_name = '審査ステータス'
  target_range = 'A1'
  in_release_process_value = 'iOS と Android 両方ともリリース進行中'

  service_account_key_json = File.read('spreadsheet-service-account-key.json')
  service_account_key = StringIO.new(service_account_key_json)
  session = GoogleDrive::Session.from_service_account_key(service_account_key)

  spreadsheet = session.spreadsheet_by_key(target_spreadsheet_id)
  sheet = spreadsheet.worksheet_by_title(sheet_name)
  sheet[target_range] = in_release_process_value
  sheet.save
end
```

`spreadsheet-service-account-key.json` は前述の通り、サービスアカウントのキー JSON ファイルです。

さらに、以下のように Fastlane のスクリプトを GitHub Actions で実行します。

GitHub Actions の Secrets に `SPREADSHEET_SERVICE_ACCOUNT_KEY_JSON_BASE64` を登録します。
値にはサービスアカウントのキー JSON ファイルを base64 エンコードしたものを登録しておきます。

```yaml
update-apps-status:
  name: Update apps status
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - name: Setup Ruby
      uses: ./.github/actions/setup-ruby
    - name: Generate service account key file
      run: echo ${{ secrets.SPREADSHEET_SERVICE_ACCOUNT_KEY_JSON_BASE64 }} | base64 -d > fastlane/spreadsheet-service-account-key.json
    - name: Update apps status
      run: bundle exec fastlane update_mobile_apps_status_to_in_release_process
```

## 商用アプリをデプロイ＆審査提出

以下のように、Fastlane でスクリプトを作成します。

```ruby:Fastfile
lane :deploy_and_submit_for_review do
  # ...
  # ここでビルドとアプリ審査提出を行う
  # ...
end
```

## 審査が通り公開された際に、データストアにリリース済みとマーク

リリース完了のメール通知が来た際に、Google スプレッドシートを更新します。

Zapier を利用しています。

Google スプレッドシートの変更時に発動する、Apps Script を利用しています。

iOS と Android 両方ともリリース通知が揃った場合に、スプレッドシートを更新します。

```javascript:script.as
const summarySheetName = 'Summary';
const statusRange = 'A2';
const statusValueBothInReleaseProcess = 'iOS と Android の両方ともリリース進行中';
const statusValueOnlyOneSideInReleaseProcess = 'iOS と Android のいずれかがリリース進行中';
const statusValueBothReleased = 'iOS と Android の両方ともリリース済み';

function checkLatestReleaseHasCompleted() {
  const status = getValueInRangeForSheetByName(summarySheetName, statusRange);

  const iosLastRow = getLastRowForSheetByName('iOS');

  const androidLastRow = getLastRowForSheetByName('Android');

  if (iosLastRow !== androidLastRow) {
    console.log(`Last row number is different: iOS = ${iosLastRow}, Android = ${androidLastRow}.`);
    console.log(`Release status is "${status}".`);

    if (status == statusValueBothInReleaseProcess) {
      console.log('The above means only one of either iOS or Android is in release process.');

      setValueInRangeForSheetByName(summarySheetName, statusRange, statusValueOnlyOneSideInReleaseProcess);
    }

    return;
  }

  console.log(`Last row number is same: iOS = ${iosLastRow}, Android = ${androidLastRow}.`);
  console.log(`Release status is "${status}".`);

  if (status == statusValueBothInReleaseProcess) {
    console.log('The above means that both of iOS and Android are in release process.');
    return;
  }

  console.log('The above means that both iOS and Android releases have been completed.');

  const spreadsheet = SpreadsheetApp.getActiveSpreadsheet();
  const iosSheet = spreadsheet.getSheetByName('iOS');
  const iosLastRange = iosSheet.getRange(iosLastRow, 1);
  const iosLastReleaseMetaData = iosLastRange.getValue();
  const iosVersionNameRegex = /App Version Number: ([0-9]+\.[0-9]+\.[0-9]+)/;
  const matched = iosLastReleaseMetaData.match(iosVersionNameRegex);

  if (!matched && matched.length <= 1) {
    console.log(`Failed to extract version name from meta data: "${iosLastReleaseMetaData}".`);
  }

  const versionName = matched[1];
  console.log(`Found "${versionName}" release.`);

  setValueInRangeForSheetByName(summarySheetName, statusRange, statusValueBothReleased);
}

function getLastRowForSheetByName(sheetName) {
  const spreadsheet = SpreadsheetApp.getActiveSpreadsheet();
  const sheet = spreadsheet.getSheetByName(sheetName);
  return sheet.getLastRow();
}

function getValueInRangeForSheetByName(sheetName, range) {
  const spreadsheet = SpreadsheetApp.getActiveSpreadsheet();
  const sheet = spreadsheet.getSheetByName(sheetName);
  const targetRange = sheet.getRange(range);
  return targetRange.getValue();
}

function setValueInRangeForSheetByName(sheetName, range, value) {
  const spreadsheet = SpreadsheetApp.getActiveSpreadsheet();
  const sheet = spreadsheet.getSheetByName(sheetName);
  const targetRange = sheet.getRange(range);
  targetRange.setValue(value);
}
```
