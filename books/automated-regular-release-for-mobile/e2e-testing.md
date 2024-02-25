---
title: "E2E テスト実行"
free: true
---

Maestro Cloud とその GitHub Actions を利用しています。

https://cloud.mobile.dev/

以下を元に、GitHub Actions の Secrets における `MAESTRO_CLOUD_API_KEY` に、Maestro Cloud の API キーを登録しておきます。
また、`.maestro/` フォルダーに E2E テストの実行内容を定義しておきます。

https://cloud.mobile.dev/ci-integration/github-actions

:::message
ここでは、Maestro を用いた E2E テストの詳しい書き方は取り扱いません。
以下を参考にしてください。
https://maestro.mobile.dev/
:::

```diff yaml:.github/workflows/regular-release.yml
name: Regular release

# ...

jobs:
  check-apps-status:
    # ...
  check-unreleased-diff:
    # ...
+  e2e-test-ios:
+    name: E2E test iOS app
+    needs: check-unreleased-diff
+    if: ${{ needs.check-unreleased-diff.outputs.has-diff-related-to-app == 'true' }}
+    runs-on: macos-14
+    steps:
+      - uses: actions/checkout@v4
+      - name: Setup Flutter
+        uses: ./.github/actions/setup-flutter
+      - name: Setup Xcode
+        uses: ./.github/actions/setup-xcode
+      - name: Setup CocoaPods
+        uses: ./.github/actions/setup-cocoapods
+      - name: Setup Ruby
+        uses: ./.github/actions/setup-ruby
+      - name: Build iOS dev app
+        run: flutter build ios --debug --simulator
+      - uses: mobile-dev-inc/action-maestro-cloud@v1.8.1
+        with:
+          api-key: ${{ secrets.MAESTRO_CLOUD_API_KEY }}
+          app-file: 'build/ios/iphonesimulator/Runner.app'
+          ios-version: 17
+          device-locale: ja_JP
+  e2e-test-android:
+    name: E2E test Android app
+    needs: check-unreleased-diff
+    if: ${{ needs.check-unreleased-diff.outputs.has-diff-related-to-app == 'true' }}
+    runs-on: ubuntu-latest
+    steps:
+      - uses: actions/checkout@v4
+      - name: Setup Flutter
+        uses: ./.github/actions/setup-flutter
+      - name: Setup Gradle
+        uses: ./.github/actions/setup-gradle
+      - name: Setup Ruby
+        uses: ./.github/actions/setup-ruby
+      - name: Build Android dev app
+        run: flutter build apk --dart-define-from-file 'dart-defines_dev.json'
+      - uses: mobile-dev-inc/action-maestro-cloud@v1.8.1
+        with:
+          api-key: ${{ secrets.MAESTRO_CLOUD_API_KEY }}
+          app-file: 'build/app/outputs/flutter-apk/app-release.apk'
+          android-api-level: 34
+          device-locale: ja_JP
```

これにより、iOS と Android の E2E テストが実行されます。
