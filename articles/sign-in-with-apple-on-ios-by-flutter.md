---
title: "FlutterによるiOSアプリでTestFlightにアップロードしたアプリのみ「Appleでサインイン」が失敗する問題の解消法メモ"
emoji: "🍎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "ios", "xcode", "apple"]
published: false
---

Flutter プロジェクトで「Apple でサインイン」を実装している際に、以下の事象に遭遇しました。

- **TestFlight にアップロードしたアプリで「Apple でサインイン」**すると、内部で何かしらのエラーが発生し、**サインインモーダルが表示されない**
- **ローカルでビルドしたアプリ**では問題なく「Apple でサインイン」が**動作する**

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
これにより、TestFlight にアップロードしたアプリでも「Apple でサインイン」機能が正常に動作するようになりました。

## まとめ

Flutter プロジェクトで Sign in with Apple 機能を実装する際は、以下の点が重要と考えています：

- Automatic Signing は便利ですが、Sign in with Apple 機能との組み合わせで問題の発生する可能性がある
- Manual Signing を使用することで、より予測可能で安定したビルド結果を得られる
- CI/CD パイプラインでの運用を考慮した署名設定の選択が重要

同様の問題に遭遇した方の参考になれば幸いです。
