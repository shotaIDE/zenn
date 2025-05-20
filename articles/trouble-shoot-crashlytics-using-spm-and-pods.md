---
title: "FlutterでSPMとCocoaPodsを併用しつつ、FirebaseCrashlyticsを使っている際のビルドエラーの対処法メモ"
emoji: "🫧"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "spm", "ios"]
published: false
---

```log
Error output from Xcode build:
↳
    ** BUILD FAILED **
2

Xcode's output:
↳
    Writing result bundle at path:
    	/var/folders/jm/nqt10pys5fx0b2w6llztpnz40000gp/T/flutter_tools.odBRmY/flutter_ios_build_temp_dirvV1jC2/temporary_xcresult_bundle

    Unhandled exception:
    ProcessException: No such file or directory
      Command: /Users/ide/works/pochi-trim/client/ios/Pods/FirebaseCrashlytics/run --validate --flutter-project /Users/ide/works/pochi-trim/client/.dart_tool/flutterfire/platforms/ios/buildConfigurations/Debug-emulator/colomney-house-worker-dev-tf1/app_id_file.json
```

Crashlytics の dSYM アップロードのスクリプトは以下のようになっている。自動生成。

```shell
#!/bin/bash
PATH="${PATH}:$FLUTTER_ROOT/bin:$HOME/.pub-cache/bin"

if [ -z "$PODS_ROOT" ]; then
  # Cannot use "BUILD_DIR%/Build/*" as per Firebase documentation, it points to "flutter-project/build/ios/*" path which doesn't have run script
  DERIVED_DATA_PATH=$(echo "$BUILD_ROOT" | sed -E 's|(.*DerivedData/[^/]+).*|\1|')
  PATH_TO_CRASHLYTICS_UPLOAD_SCRIPT="${DERIVED_DATA_PATH}/SourcePackages/checkouts/firebase-ios-sdk/Crashlytics/run"
else
  PATH_TO_CRASHLYTICS_UPLOAD_SCRIPT="$PODS_ROOT/FirebaseCrashlytics/run"
fi

# Command to upload symbols script used to upload symbols to Firebase server
flutterfire upload-crashlytics-symbols --upload-symbols-script-path="$PATH_TO_CRASHLYTICS_UPLOAD_SCRIPT" --platform=ios --apple-project-path="${SRCROOT}" --env-platform-name="${PLATFORM_NAME}" --env-configuration="${CONFIGURATION}" --env-project-dir="${PROJECT_DIR}" --env-built-products-dir="${BUILT_PRODUCTS_DIR}" --env-dwarf-dsym-folder-path="${DWARF_DSYM_FOLDER_PATH}" --env-dwarf-dsym-file-name="${DWARF_DSYM_FILE_NAME}" --env-infoplist-path="${INFOPLIST_PATH}" --build-configuration=${CONFIGURATION}
```

`if [ -z "$PODS_ROOT" ]; then` で、CocoaPods を使っていない判定をしている。

一方で、Crashlytics は SPM 対応しているので、CocoaPods を使わずに SPM で導入している場合は、`PODS_ROOT` は空になる。
