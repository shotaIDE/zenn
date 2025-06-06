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

サインインモーダルは以下を指しています。

![Appleでサインインのモーダル](/images/sign-in-with-apple-on-ios-by-flutter/sign-in-with-apple-modal.png =300x)

解決策をメモしておきます。

## 前提

以下のプロジェクトです。

- Flutter プロジェクトにおける iOS アプリ
- Apple でサインイン機能を実装済み

また、Xcode プロジェクトファイルの設定で、**Automatic Signing を有効に**しています。

![Automatic Signingが有効](/images/sign-in-with-apple-on-ios-by-flutter/automatic-signing.png)

一方で、Test Flight にアップロードするアプリは、**Manual Signing によりビルド**したものとしていました。
具体的には、以下のような一連のコマンドでビルドしたものをアップロードしていました。

```shell
# Flutter部分をビルド
flutter build ios --no-codesign

# iOSネイティブをアーカイブ
xcodebuild archive \
    CODE_SIGNING_ALLOWED=NO \
    -workspace ./ios/Runner.xcworkspace \
    -scheme Runner \
    -configuration Release \
    -archivePath ./build/ios/Runner.xcarchive \
    -authenticationKeyIssuerID "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" \
    -authenticationKeyID "XXXXXXXXXX" \
    -authenticationKeyPath ./ios/fastlane/app-store-connect-api-key.p8

# iOSネイティブをipaファイルにエクスポート
xcodebuild -exportArchive \
    -archivePath ./build/ios/Runner.xcarchive \
    -exportPath ./build/ios/ipa \
    -exportOptionsPlist "${EXPORT_OPTIONS_PLIST_RELATIVE_PATH}" \
    -allowProvisioningUpdates \
    -authenticationKeyIssuerID "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" \
    -authenticationKeyID "XXXXXXXXXX" \
    -authenticationKeyPath ./ios/fastlane/app-store-connect-api-key.p8
```

:::message
このような段階を踏んだビルド方法を取っていたのは、ローカルマシンの開発時には Automatic Signing を有効にしておきたかったためです。
:::

`-exportOptionsPlist` オプションのファイルには、以下のような内容を指定していました。
**Manual Signing の設定が含まれた**ものになっています。

```xml:./ios/ExportOptions.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>destination</key>
	<string>export</string>
	<key>method</key>
	<string>app-store-connect</string>
	<key>provisioningProfiles</key>
	<dict>
		<key>my.application.bundle.identifier</key>
		<string>My Application for App Store</string>
	</dict>
	<key>signingCertificate</key>
	<string>Apple Distribution</string>
	<key>signingStyle</key>
	<string>manual</string>
	<key>teamID</key>
	<string>XXXXXXXXXX</string>
	<!-- ... -->
</dict>
</plist>
```

## 結論

Xcode プロジェクトの構成を Manual Signing に変更し、ビルドコマンド時に署名周りの設定を付け替える必要がないようにしました。
この設定によりビルドしたアプリを Test Flight にアップロードすると、「Apple でサインイン」機能が正常に動作しました。

## 詳細

具体的には以下の通りの手順で設定しています。

**Xcode プロジェクトファイルの構成そのものを Manual Signing に変更**しました。
Test Flight にアップロードする用の構成(`Release-prod`)のみ変更を適用しています。

![Manual Signingに変更](/images/sign-in-with-apple-on-ios-by-flutter/manual-signing.png)

また、署名周りの設定の付け替えが必要なくなったため、Flutter のビルドコマンド 1 つで TestFlight へのアップロードに使うアプリが生成できるようになりました。

```shell
flutter build ipa --export-options-plist "${EXPORT_OPTIONS_PLIST_RELATIVE_PATH}"
```
