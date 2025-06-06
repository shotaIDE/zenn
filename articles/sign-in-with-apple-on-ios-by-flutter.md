---
title: "FlutterによるiOSアプリでTestFlightにアップロードしたアプリのみ「Appleでサインイン」が失敗する"
emoji: "🍎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "ios", "xcode", "apple"]
published: false
---

<!-- cSpell:ignore textlint, xcarchive -->

Flutter プロジェクトで「Apple でサインイン」を実装している際に、以下の事象に遭遇しました。

- **Test Flight にアップロードしたアプリで「Apple でサインイン**」すると、内部で何かしらのエラーが発生し、**サインインモーダルが表示されない**
- **ローカルでビルドしたアプリでは問題なく「Apple でサインイン」が動作する**

解決策をメモしておきます。

## 前提

- Flutter プロジェクトにおける iOS アプリ
- Apple でサインイン機能を実装済み

また、Xcode プロジェクトファイルの設定で、Automatic Signing が有効になっています。

![Automatic Signingが有効]()

一方で、Test Flight にアップロードするアプリは、**Manual Signing によりビルド**したものとしていました。
具体的には、以下のようなコマンドでビルドしていました。

```shell
# Flutter部分をビルド
flutter build ios --no-codesign

# iOSネイティブをアーカイブ
xcodebuild archive \
    CODE_SIGNING_ALLOWED=NO
   -workspace ./ios/Runner.xcworkspace \
   -scheme Runner \
   -configuration Release \
    -archivePath ./build/ios/Runner.xcarchive
    -archivePath ./build/ios/Runner.xcarchive \
    -authenticationKeyIssuerID "${APPLE_API_ISSUER_ID}" \
    -authenticationKeyID "${APPLE_API_KEY_ID}" \
    -authenticationKeyPath "${APP_STORE_CONNECT_API_KEY_ABSOLUTE_PATH}"

# iOSネイティブをipaファイルにエクスポート
xcodebuild -exportArchive \
   -archivePath ./build/ios/Runner.xcarchive \
   -exportPath ./build/ios/ipa \
   -exportOptionsPlist "${EXPORT_OPTIONS_PLIST_RELATIVE_PATH}" \
   -allowProvisioningUpdates \
   -authenticationKeyIssuerID "${APPLE_API_ISSUER_ID}" \
   -authenticationKeyID "${APPLE_API_KEY_ID}" \
   -authenticationKeyPath "${APP_STORE_CONNECT_API_KEY_ABSOLUTE_PATH}"
```

## 結論

**Xcode プロジェクトファイルの構成そのものを Manual Signing に変更**しました。
これに伴い、Flutter のビルドコマンド 1 つで TestFlight へのアップロードに使うアプリが生成できるようになりました。

```shell
flutter build ipa --export-options-plist "${EXPORT_OPTIONS_PLIST_RELATIVE_PATH}"
```

このアプリを Test Flight にアップロードすると、「Apple でサインイン」機能が正常に動作するようになりました。
