---
title: "データストアにリリースの審査中とマーク"
free: true
---

Google スプレッドシートに、リリース作業開始を記録します。

以下のように、Fastlane でスクリプトを作成します。

```diff ruby:ios/fastlane/Fastfile
default_platform(:ios)

platform :ios do
  # ...

+  lane :update_mobile_apps_status_to_in_release_process do
+    Dotenv.load '.env'
+
+    target_spreadsheet_id = 'xxxx' # スプレッドシートのURLの https://docs.google.com/spreadsheets/d/xxxx/edit における xxxx の部分
+    sheet_name = '審査ステータス'
+    target_range = 'A1'
+    in_release_process_value = 'iOS と Android 両方ともリリース進行中'
+
+    service_account_key_json = File.read('spreadsheet-service-account-key.json')
+    service_account_key = StringIO.new(service_account_key_json)
+    session = GoogleDrive::Session.from_service_account_key(service_account_key)
+
+    spreadsheet = session.spreadsheet_by_key(target_spreadsheet_id)
+    sheet = spreadsheet.worksheet_by_title(sheet_name)
+    sheet[target_range] = in_release_process_value
+    sheet.save
+  end
end
```

`spreadsheet-service-account-key.json` は前述の通り、サービスアカウントのキー JSON ファイルです。

さらに、以下のように Fastlane のスクリプトを GitHub Actions で実行します。

GitHub Actions の Secrets に `SPREADSHEET_SERVICE_ACCOUNT_KEY_JSON_BASE64` を登録します。
値にはサービスアカウントのキー JSON ファイルを base64 エンコードしたものを登録しておきます。

```diff yaml:.github/workflows/regular-release.yml
name: Regular release

# ...

jobs:
  check-apps-status:
    # ...
  check-unreleased-diff:
    # ...
  e2e-test-ios:
    # ...
  e2e-test-android:
    # ...
-  next-job:
+  update-apps-status:
+    name: Update apps status
+    runs-on: ubuntu-latest
    needs:
      - e2e-test-ios
      - e2e-test-android
+    steps:
+      - uses: actions/checkout@v4
+      - name: Setup Ruby
+        uses: ./.github/actions/setup-ruby
+      - name: Generate service account key file
+        run: echo ${{ secrets.SPREADSHEET_SERVICE_ACCOUNT_KEY_JSON_BASE64 }} | base64 -d > fastlane/spreadsheet-service-account-key.json
+      - name: Update apps status
+        run: bundle exec fastlane ios update_mobile_apps_status_to_in_release_process
```
