---
title: "FlutterでSPMを利用している際に `Module not found` エラーが出た時の対処法メモ"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ios", "flutter"]
published: false
---

# はじめに

Flutter プロジェクトで iOS 向けに SPM（Swift Package Manager）を利用している際、"Module shared_preferences_foundation not found" という旨のエラーが出たので、その対処法を簡単にメモしておきます。

# 前提

以下の環境で発生したエラーです。

- Flutter 3.29.3 stable
- Xcode 16.3

対象のプロジェクトでは、iOS プラットフォームで SPM を利用しています。

Flutter で SPM はまだ正式サポートされていないため、以下のコマンドを実行した状態で、ビルドなどを行っています。

```bash
flutter config --enable-swift-package-manager
```

詳しくは公式のドキュメントを参考にしてください。

https://docs.flutter.dev/packages-and-plugins/swift-package-manager/for-app-developers

# 結論

エラーが発生しているモジュールの大元となっている Flutter の依存関係を一旦削除し、再度追加することで、ビルドエラーが解決しました。

1. `pubspec.yaml` から `shared_preferences` を一度削除し、`flutter pub get --no-example` を実行
2. 再度 `shared_preferences` を追加し、`flutter pub get --no-example` を実行
3. iOS のビルドを実行

# 詳細

## 発生したエラー

```log
Launching lib/main.dart on iOS 17 iPhone in debug mode...
Automatically signing iOS for device deployment using specified development team in Xcode project: 4UGYN353AH
Xcode build done.                                           52.5s
Failed to build iOS app
Could not build the precompiled application for the device.
Parse Issue (Xcode): Module 'shared_preferences_foundation' not found
/Users/ide/localWorks/house-worker/client/ios/Runner/GeneratedPluginRegistrant.m:53:8
2

Error launching application on iOS 17 iPhone.
```

## 状況メモ

- 上記のようなエラーが発生
- `Package.swift` には以下のような記述があった

```swift
.package(name: "shared_preferences_foundation", path: "/Users/ide/.pub-cache/hosted/pub.dev/shared_preferences_foundation-2.5.4/darwin/shared_preferences_foundation"),
.product(name: "shared-preferences-foundation", package: "shared_preferences_foundation"),
```

パスや依存関係が壊れている可能性がありそう、という印象だった。

## 試したこと

- `pubspec.yaml` から `shared_preferences` を一度削除して `flutter pub get --no-example` を実行
- その後、再度 `shared_preferences` を追加して `flutter pub get --no-example` を実行

これで iOS ビルドが通るようになりました。

# まとめ

根本原因は深掘りしていませんが、依存関係の再構築で解決したので、同じようなエラーが出た場合は参考にしてみてください。
