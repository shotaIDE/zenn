---
title: "商用アプリをデプロイ＆審査提出"
free: true
---

以下のように、Fastlane でスクリプトを作成します。

```diff ruby:ios/fastlane/Fastfile
default_platform(:ios)

platform :ios do
  # ...

+  lane :bump_patch_version do
+    Dir.chdir('../') do
+      sh('dart run cider bump patch --bump-build')
+    end
+  end
+
+  lane :add_release_candidate_tag do
+    full_version_name = get_full_version_name
+
+    add_git_tag(
+      grouping: 'rc',
+      includes_lane: false,
+      build_number: full_version_name
+    )
+  end
+
+  lane :set_full_version_name_from_latest_tag do
+    latest_tag_name = Dir.chdir('../') do
+      sh("git describe --tags --abbrev=0 --match='rc/*'")
+    end
+
+    matched = %r{^.*/(\d+\.\d+\.\d+\+\d+)$}.match(latest_tag_name)
+
+    full_version_name = matched[1]
+
+    Dir.chdir('../') do
+      sh("dart run cider version #{full_version_name}")
+    end
+  end
end
```

```diff ruby:ios/fastlane/Fastfile
default_platform(:ios)

platform :ios do
  # ...
+  lane :build_prod do
+    export_options_plist_path = 'ios/Runner/ExportOptions.plist'
+
+    Dir.chdir('../') do
+      sh(
+        'flutter build ipa '\
+        "--export-options-plist=#{export_options_plist_path}"
+      )
+    end
+
+    lane_context[SharedValues::IPA_OUTPUT_PATH] = 'build/ios/ipa/アプリ名.ipa'
+  end
+
+  lane :deploy_prod_without_build do
+    apple_api_key_id = 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
+    apple_api_issuer_id = 'xxxxxxxxxx'
+
+    app_store_connect_api_key_path = 'fastlane/app-store-connect-api-key.p8'
+
+    lane_context[SharedValues::IPA_OUTPUT_PATH] = 'build/ios/ipa/アプリ名.ipa'
+
+    api_key = app_store_connect_api_key(
+      key_id: apple_api_key_id,
+      issuer_id: apple_api_issuer_id,
+      key_filepath: app_store_connect_api_key_path
+    )
+
+    upload_to_app_store(
+      api_key: api_key,
+      metadata_path: 'fastlane/metadata/ios',
+      skip_metadata: false,
+      skip_screenshots: true,
+      force: true,
+      submit_for_review: true,
+      reject_if_possible: true,
+      automatic_release: true,
+      submission_information: { add_id_info_uses_idfa: false },
+      precheck_include_in_app_purchases: false
+    )
+  end
end
```

```diff ruby:android/fastlane/Fastfile
default_platform(:android)

platform :android do
  # ...

+  lane :deploy_prod do
+    package_name = 'your.app.package.id'
+    metadata_path = 'fastlane/metadata/android'
+    google_play_json_key_path = 'fastlane/google-play-service-account-key.json'
+
+    Dir.chdir('../') do
+      sh('flutter build appbundle')
+    end
+
+    lane_context[SharedValues::GRADLE_AAB_OUTPUT_PATH] = 'build/app/outputs/bundle/release/app-release.aab'
+
+    upload_to_play_store(
+      package_name: package_name,
+      release_status: 'completed',
+      track: 'production',
+      metadata_path: metadata_path,
+      json_key: google_play_json_key_path
+    )
+  end
end
```

```diff yaml:.github/workflows/regular-release.yml
name: Regular release

# ...

jobs:
  check-apps-status:
    # ...
  check-unreleased-diff:
    # ...
  e2e-test-ios:
    # ...
  e2e-test-android:
    # ...
  update-apps-status:
    # ...
+  trigger-cd-production:
+    name: Trigger CD production
+    runs-on: ubuntu-latest
+    needs:
+      - e2e-test-ios
+      - e2e-test-android
+    steps:
+      - name: Generate token
+        id: generate-token
+        uses: tibdex/github-app-token@v2
+        with:
+          app_id: ${{ secrets.MY_WORKFLOW_APP_ID }}
+          private_key: ${{ secrets.MY_WORKFLOW_APP_PRIVATE_KEY_PEM }}
+      - uses: actions/checkout@v4
+        with:
+          # Use App tokens instead of default token `GITHUB_TOKEN`
+          # to ensure that GitHub Actions will be triggered again after push back.
+          token: ${{ steps.generate-token.outputs.token }}
+          fetch-depth: 0
+      - name: Setup Git
+        uses: ./.github/actions/setup-git
+      - name: Setup Flutter
+        uses: ./.github/actions/setup-flutter
+      - name: Setup Ruby
+        uses: ./.github/actions/setup-ruby
+      - name: Set full version name from tag
+        run: bundle exec fastlane ios set_full_version_name_from_latest_tag
+      - name: Bump patch version
+        run: bundle exec fastlane ios bump_patch_version
+      - name: Add tag
+        run: bundle exec fastlane ios add_release_candidate_tag
+      - name: Push back tag
+        run: |
+          latest_tag_name="$(git describe --tags --abbrev=0)"
+          git push origin "${latest_tag_name}"
```

```yaml:.github/workflows/ios-production-deploy.yml
name: iOS Production Deploy

on:
  push:
    tags:
      - "rc/*"

jobs:
  deploy:
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
      - name: Set full version name from tag
        run: bundle exec fastlane set_full_version_name_from_latest_tag
      - name: Generate App Store Connect API key file
        run: echo "${{ secrets.APP_STORE_CONNECT_API_KEY_P8_BASE64 }}" | base64 -d > fastlane/app-store-connect-api-key.p8
      - name: Import Code-Signing Certificates
        uses: Apple-Actions/import-codesign-certs@v2
        with:
          p12-file-base64: ${{ secrets.APP_DISTRIBUTION_P12_BASE64 }}
          p12-password: ${{ secrets.APP_DISTRIBUTION_P12_PASSWORD }}
      - name: Import Provisioning Profile
        run: |
          mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
          echo "${{ secrets.PROVISIONING_PROFILE_PROD_APP_STORE_BASE64 }}" | base64 -d > ~/Library/MobileDevice/Provisioning\ Profiles/prod_app-store.mobileprovision
      - name: Build prod app
        run: bundle exec fastlane ios build_prod
      # Uploading to App Store may fail for temporary reasons, so we retry.
      - name: Deploy prod app to App Store
        uses: nick-fields/retry@v3
        with:
          timeout_minutes: 360
          max_attempts: 3
          retry_on: error
          command: bundle exec fastlane ios deploy_prod_without_build
```

```yaml:.github/workflows/android-production-deploy.yml
name: Android Production Deploy

on:
  push:
    tags:
      - "rc/*"

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup Flutter
        uses: ./.github/actions/setup-flutter
      - name: Setup Gradle
        uses: ./.github/actions/setup-gradle
      - name: Setup Ruby
        uses:
      - name: Set full version name from tag
        run: bundle exec fastlane set_full_version_name_from_latest_tag
      - name: Generate Google Play service account key file
        run: echo "${{ secrets.GOOGLE_PLAY_SERVICE_ACCOUNT_KEY_JSON_BASE64 }}" | base64 -d > fastlane/google-play-service-account-key.json
      - name: Deploy prod app to Google Play
        run: bundle exec fastlane android deploy_prod
```
