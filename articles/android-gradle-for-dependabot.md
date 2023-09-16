---
title: "AndroidのGradleファイルにおけるバージョンの共通化をdependabotが反応するように構築する"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dependabot", "gradle", "android"]
published: true
---

# 背景

Android の Gradle ファイルにおいて、複数のライブラリのバージョンを固定させるためにバージョンの変数を用いることがあります。

```gradle:app/build.gradle
mockk_version = '1.12.5' // バージョンの変数

testImplementation "io.mockk:mockk:$mockk_version"
testImplementation "io.mockk:mockk-android:$mockk_version"
```

しかし、上記の定義方法だと、**[dependabot](https://docs.github.com/ja/code-security/dependabot/dependabot-version-updates) が反応せず、ライブラリ更新の自動化を導入できません**。

本記事では、複数のライブラリバージョンの共通化を強制しつつ、dependabot を利用した自動ライブラリ更新を実現する方法を紹介します。

# 結論

**バージョンカタログという仕組みを利用**して定義します。

https://developer.android.com/build/migrate-to-catalogs

[Gradle 7.0 以上が必要](https://docs.gradle.org/7.0/release-notes.html)です。

# 手順

まず、Gradle 8.0 未満を利用している場合は、`settings.gradle` に以下の行を追加して機能を有効化します。

```gradle:settings.gradle
enableFeaturePreview("VERSION_CATALOGS")
```

次に、`gradle/libs.versions.toml` ファイルを作成し、以下のように定義します。

```gradle:gradle/libs.versions.toml
[versions]
mockk_version = "1.12.5"

[libraries]
mockk = { group = "io.mockk", name = "mockk", version.ref = "mockk_version" }
mockkAndroid = { group = "io.mockk", name = "mockk-android", version.ref = "mockk_version" }

[plugins]
```

さらに、`app/build.gradle` にて、以下のように修正します。

```diff gradle:app/build.gradle
- mockk_version = '1.12.5' // バージョンの変数
-
- testImplementation "io.mockk:mockk:$mockk_version"
+ testImplementation libs.mockk
- testImplementation "io.mockk:mockk-android:$mockk_version"
+ testImplementation libs.mockkAndroid
```

これをマージすると、dependabot が反応し、以下のような PR が自動で作成されるようになります。

![](/images/android-gradle-for-dependabot/01_pr-summary.png)
![](/images/android-gradle-for-dependabot/02_pr-diff.png)
