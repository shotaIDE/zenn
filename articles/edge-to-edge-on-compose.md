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

`WindowInsets.safeContent` を利用することで、切り欠きを避けて描画できる。

| 項目          | Android 15                                       | Android 14                                           |
| ------------- | ------------------------------------------------ | ---------------------------------------------------- |
| `safeDrawing` | 端末を横にした際、ノッチを避けるようになっている | 端末を横にした際、ノッチを避けるようになっていない？ |
| `safeContent` | 端末を横にした際、ノッチを避けるようになっている | 端末を横にした際、ノッチを避けるようになっている？   |

以下のような方針で実装することで、いい感じにできそうです。

スクロール可能な方向における端は、セーフエリアを飛び出して画面いっぱいまで描画し、セーフエリア内にスクロールできるようにスクロール内部のパディングを設定する。
スクロール不能な方向における端は、セーフエリアを飛び出さないように描画する。

https://developer.android.com/develop/ui/compose/modifiers?hl=ja

https://qiita.com/Nabe1216/items/6fd9e2293f7ae109150a

https://developer.android.com/design/ui/mobile/guides/layout-and-content/edge-to-edge?hl=ja

https://developer.android.com/develop/ui/views/layout/edge-to-edge?hl=ja#enable-edge-to-edge-display
