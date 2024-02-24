---
title: "E2E テスト実行"
---

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
