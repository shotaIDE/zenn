---
title: "Composeで没入感のあるエッジツーエッジのUIを実現する"
emoji: "🌟"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["android", "compose"]
published: false
---

残件

- [ ] Android 15 未満でエッジツーエッジを適用する方法を調査する

https://developer.android.com/develop/ui/compose/system/material-insets?hl=ja#override-default

https://developers-jp.googleblog.com/2019/10/gesture-navigation-handling-visual-overlaps.html

`modifier` におけるパディングは、`LazyColumn` のスクロール領域外に対して適用されます。
一方で、 `contentPadding` によるパディングは、`LazyColumn` のスクロール領域に対して適用されます。
そのため、`contentPadding` によるパディングを設定することが適切です。

https://developer.android.com/develop/ui/compose/modifiers?hl=ja
