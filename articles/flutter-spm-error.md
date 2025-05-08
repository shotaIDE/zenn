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

上記のエラーが発生していた。

```
.package(name: "shared_preferences_foundation", path: "/Users/ide/.pub-cache/hosted/pub.dev/shared_preferences_foundation-2.5.4/darwin/shared_preferences_foundation"),
```

上記のように、`shared_preferences_foundation`のパスが指定されている。

```
.product(name: "shared-preferences-foundation", package: "shared_preferences_foundation"),
```

一度 `pubspec.yaml` を修正して、`shared_preferences` を削除して、`flutter pub get` を実行した。

そのあと、再度 `shared_preferences` を追加して、`flutter pub get` を実行した。
