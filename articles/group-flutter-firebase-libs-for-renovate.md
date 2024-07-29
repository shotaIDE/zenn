---
title: "Flutterで利用している複数のFirebaseライブラリを、Renovateでうまく自動更新する"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "firebase", "renovate", "ios", "android"]
published: false
---

# 背景

Renovate は、依存しているライブラリのバージョンアップを効率化するためのツールです。

https://docs.renovatebot.com/

Flutter で Firebase を使っていると、多くの場合 Firebase のライブラリを複数導入することになります。

これらのライブラリは、同時に新しいバージョンがリリースされることも多く、また、**同時にバージョンアップしないといけない制約が設定されている**こともあります。

一方で **Renovate は初期設定の状態だと、 1 回につき 1 つのライブラリしか更新しません**。

そのため、Flutter における Firebase のライブラリを Renovate でアップデートしようとすると、バージョン制約の解決により失敗する場合があります。

今回はその解決策を紹介します。

# やり方の概要

Renovate の `matchPackageNames` を利用し、利用している Firebase のライブラリを 1 つの PR で更新するようにします。

https://docs.renovatebot.com/configuration-options/#matchpackagenames

# やり方の詳細

以下のように Renovate の設定ファイルを構成します。

```json:renovate.json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "enabledManagers": [
    "pub"
  ],
  "extends": ["config:recommended"],
  "packageRules": [
    {
      "matchPackageNames": [
        "cloud_firestore",
        "firebase_analytics",
        "firebase_auth",
        "firebase_core",
        "firebase_crashlytics",
        "firebase_messaging",
        "firebase_remote_config",
        "firebase_storage"
      ],
      "groupName": "firebase"
    }
  ]
}
```

`matchPackageNames` は一例です。
ご自身の管理する `pubspec.yaml` を参考に、利用している Firebase のライブラリを指定してください。

上記の設定をメインブランチにマージすると、以下のように **Firebase のライブラリ群を同時に更新する PR が 1 つ作成されます**。

![](/images/group-flutter-firebase-libs-for-renovate/pr-updating-firebase-libraries.png)

# 補足 1: Renovate におけるライブラリグループ化のプリセット

本記事で紹介したライブラリ群のグループ化に関して、Renovate 自体にいくつかのプリセットが用意されています。
しかし、記事執筆時点では Firebase のライブラリ群をグループ化するようなプリセットは見つかりませんでした。

https://docs.renovatebot.com/presets-group/

# 補足 2: Android ネイティブにおける Firebase のライブラリ群のバージョン制約解決

また、Android ネイティブで Firebase を導入している場合、Firebase Android BoM が利用できます。

https://firebase.google.com/docs/android/learn-more?hl=ja#bom

これは、Firebase のライブラリ群のバージョン制約解決を効率的に行うための仕組みです。
BoM に対するバージョンを 1 つだけ指定すれば、適切な Firebase のライブラリ群のバージョンを解決してくれます。

Flutter ではこのような仕組みが提供されていないため、本記事で紹介したような工夫が必要になります。
