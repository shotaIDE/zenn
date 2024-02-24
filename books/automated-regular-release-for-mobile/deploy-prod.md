---
title: "商用アプリをデプロイ＆審査提出"
free: true
---

以下のように、Fastlane でスクリプトを作成します。

```ruby:fastlane/Fastfile
lane :bump_patch_version do
  Dir.chdir('../') do
    sh('dart run cider bump patch --bump-build')
  end
end

lane :add_release_candidate_tag do
  full_version_name = get_full_version_name

  add_git_tag(
    grouping: 'rc',
    includes_lane: false,
    build_number: full_version_name
  )
end

lane :set_full_version_name_from_latest_tag do
  latest_tag_name = Dir.chdir('../') do
    sh("git describe --tags --abbrev=0 --match='rc/*'")
  end

  matched = %r{^.*/(\d+\.\d+\.\d+\+\d+)$}.match(latest_tag_name)

  full_version_name = matched[1]

  Dir.chdir('../') do
    sh("dart run cider version #{full_version_name}")
  end
end
```

```ruby:ios/fastlane/Fastfile
  desc 'Build production app'
  lane :build_prod do
    install_dependencies

    generate

    build(
      dart_defines_path: 'dart-defines_prod.json',
      export_options_plist_path: './ios/ExportOptions_AppStore.plist'
    )
  end
```

```yaml:.github/workflows/automated-release.yml
trigger-cd-production:
  name: Trigger CD production
  runs-on: ubuntu-latest
  needs:
    - e2e-test-ios
    - e2e-test-android
  steps:
    - name: Generate token
      id: generate-token
      uses: tibdex/github-app-token@v2
      with:
        app_id: ${{ secrets.MY_WORKFLOW_APP_ID }}
        private_key: ${{ secrets.MY_WORKFLOW_APP_PRIVATE_KEY_PEM }}
    - uses: actions/checkout@v4
      with:
        # Use App tokens instead of default token `GITHUB_TOKEN`
        # to ensure that GitHub Actions will be triggered again after push back.
        token: ${{ steps.generate-token.outputs.token }}
        fetch-depth: 0
    - name: Setup Git
      uses: ./.github/actions/setup-git
    - name: Setup Flutter
      uses: ./.github/actions/setup-flutter
    - name: Setup Ruby
      uses: ./.github/actions/setup-ruby
    - name: Set full version name from tag
      run: bundle exec fastlane set_full_version_name_from_latest_tag
    - name: Bump patch version
      run: bundle exec fastlane bump_patch_version
    - name: Add tag
      run: bundle exec fastlane add_release_candidate_tag
    - name: Push back tag
      run: |
        latest_tag_name="$(git describe --tags --abbrev=0)"
        git push origin "${latest_tag_name}"
```

```yaml:.github/workflows/ios-production-deploy.yml
name: CD / iOS Production

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
