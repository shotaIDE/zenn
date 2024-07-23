---
title: "Flutterで利用しているFirebaseのライブラリを、Renovateでうまく自動更新する"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "firebase", "renovate", "ios", "android"]
published: false
---

# 背景

Renovate とは利用しているライブラリのバージョンアップを効率化するためのツールです。

https://docs.renovatebot.com/

Flutter で Firebase のライブラリを導入している際、Firebase の各種ライブラリを複数導入していることがあります。

これらのライブラリは、同時に新しいバージョンがリリースされることも多く、かつ、依存関係もあります。
そのため、同時に新しいバージョンに乗り換える必要が出てきます。

Renovate では、初期状態だと 1 つ 1 つのライブラリしか更新しません。
そのため、上記のライブラリ間の依存関係の制約により自動更新に失敗する場合があります。

今回はその解決策を紹介します。

# やり方の概要

Renovate の group を利用し、利用している Firebase のライブラリを指定します。

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

# やり方の詳細

# 補足

Android ネイティブで Firebase を導入している場合、Firebase のライブラリ群の依存関係を解決するために、Firebase Android BoM を利用できます。

https://firebase.google.com/docs/android/learn-more?hl=ja#bom

これは、1 つのバージョンを指定することで、Firebase のライブラリ群の依存関係を解決するための仕組みです。

Flutter ではこのような仕組みが提供されていないため、上記のような工夫が必要になります。
