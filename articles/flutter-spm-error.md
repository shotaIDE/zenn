---
title: "FlutterでSPMを利用している際に `Module not found` エラーが出た時の対処法メモ"
emoji: "🫧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "spm", "ios"]
published: true
---

# はじめに

Flutter プロジェクトで iOS 向けに Swift Package Manager(以下、SPM と記載)を利用している際、以下のようなエラーが出たので、その対処法を簡単にメモしておきます。

```log
Module `shared_preferences_foundation` not found
```

:::message
根本原因の深掘りはしていないので、同じような事象が起きた際の参考程度にしてください 🙏
:::

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

1. `pubspec.yaml` から `shared_preferences` を一度削除し、iOS のビルドを実行
2. 再度 `shared_preferences` を追加し、iOS のビルドを実行

# 詳細な対応ログ

まず、以下のようなエラーが発生しました。

```log
Launching lib/main.dart on iOS 17 iPhone in debug mode...
Automatically signing iOS for device deployment using specified development team in Xcode project: xxxxxxxxxx
Xcode build done.                                           52.5s
Failed to build iOS app
Could not build the precompiled application for the device.
Parse Issue (Xcode): Module 'shared_preferences_foundation' not found
/Users/xxx/localWorks/house-worker/client/ios/Runner/GeneratedPluginRegistrant.m:53:8
2

Error launching application on iOS 17 iPhone.
```

ここで、`shared_preferences_foundation` でプロジェクト内を検索してみると、以下のような記述がありました。

```swift:Package.swift
.package(name: "shared_preferences_foundation", path: "/Users/xxx/.pub-cache/hosted/pub.dev/shared_preferences_foundation-2.5.4/darwin/shared_preferences_foundation"),
.product(name: "shared-preferences-foundation", package: "shared_preferences_foundation"),
```

ローカルマシンにキャッシュされた依存関係のファイルがリンク切れを起こしていそうという印象でした。

そこで、依存関係を一旦削除して、再度追加してみました。

1. `pubspec.yaml` から `shared_preferences` を一度削除する。

```diff yaml:pubspec.yaml
# ...
dependencies:
-  shared_preferences: ^2.5.3
```

iOS のビルドを実行する。

その後、再度 `shared_preferences` を追加する。

```diff yaml:pubspec.yaml
# ...
dependencies:
+  shared_preferences: ^2.5.3
```

iOS のビルドを実行する。

これで iOS ビルドが通るようになりました。
