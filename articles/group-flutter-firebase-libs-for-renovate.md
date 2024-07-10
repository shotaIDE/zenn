---
title: "Flutterで利用しているFirebaseのライブラリを、Renovateでうまく自動更新する"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "firebase", "renovate", "ios", "android"]
published: false
---

# 背景

Renovate とは利用しているライブラリのバージョンアップを効率化するためのツールです。

Flutter で Firebase のライブラリを導入している際、Firebase の各種ライブラリを複数導入していることがあります。

これらのライブラリは、同時に新しいバージョンがリリースされることも多く、かつ、依存関係もあることから、同時に新しいバージョンに乗り換える必要が出てきます。

Renovate では、初期状態だと一つ一つのライブラリしか更新しないため、PR で自動更新が失敗する場合があります。

今回はその解決策を紹介します。

# やり方の概要

Renovate の group を利用し、利用している Firebase のライブラリを指定します。

```json:renovate.json

```

# やり方の詳細

# 補足
