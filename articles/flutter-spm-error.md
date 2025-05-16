---
title: "FlutterでSPMを利用している際に `Module not found` エラーが出た時の対処法メモ"
emoji: "🕌"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["ios", "android"]
published: false
---

# はじめに

Flutter の iOS ビルド時に "Module shared_preferences_foundation not found" という旨のエラーが出たので、その際の対処法を簡単にメモしておきます。

# 結論

依存関係の再構築（`pubspec.yaml` から `shared_preferences` を一度削除し、再度追加して `flutter pub get` を実行）でビルドエラーが解消しました。

# 発生したエラー

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

# 状況メモ

- 上記のようなエラーが発生
- `Package.swift` には以下のような記述があった

```swift
.package(name: "shared_preferences_foundation", path: "/Users/ide/.pub-cache/hosted/pub.dev/shared_preferences_foundation-2.5.4/darwin/shared_preferences_foundation"),
.product(name: "shared-preferences-foundation", package: "shared_preferences_foundation"),
```

:::message
パスや依存関係が壊れている可能性がありそう、という印象でした。
:::

# 試したこと

- `pubspec.yaml` から `shared_preferences` を一度削除して `flutter pub get`
- その後、再度 `shared_preferences` を追加して `flutter pub get`

これでビルドが通るようになりました。

# まとめ

根本原因は深掘りしていませんが、依存関係の再構築で解決したので、同じようなエラーが出た場合は一度試してみてください。
