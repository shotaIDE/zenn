---
title: "Jetpack Composeでエッジツーエッジの没入感があるスクロールUIを実現する"
emoji: "🌟"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["android", "compose"]
published: false
---

<!-- cspell:ignore jetpack -->

Jetpack Compose を利用して、システム UI の領域を超えてエッジツーエッジでスクロール UI を描画し、没入感を演出する方法について書きます。

| 向き | これを(Before)                                                                                                                | こうしたい(After)                                                                                                            |
| ---- | ----------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| 縦   | ![システムUIを避けてスクロールビューが描画されている・縦画面](/images/edge-to-edge-on-compose/goal_01-a_portrait-before.gif)  | ![システムUIの背景にスクロールビューが描画されている・縦画面](/images/edge-to-edge-on-compose/goal_01-b_portrait-after.gif)  |
| 横   | ![システムUIを避けてスクロールビューが描画されている・横画面](/images/edge-to-edge-on-compose/goal_02-a_landscape-before.gif) | ![システムUIの背景にスクロールビューが描画されている・横画面](/images/edge-to-edge-on-compose/goal_02-b_landscape-after.gif) |

公式ドキュメントなどでドンピシャなサンプルコードや説明がなく少し手間取ったので、メモしておきます。

## 結論

`Scaffold` の子 Compose に渡される `PaddingValues` には画面下部のナビゲーションバーの高さが含まれています。
これを `LazyColumn` の `contentPadding` にスクロール内部の余白として指定します。

また、`WindowInsets.safeDrawing` にはナビゲーションバー以外のシステム UI や切り欠きのサイズが含まれています。
これを、`TopAppBar` や `LazyColumn` の `windowInsetsPadding` に左右のサイズだけを取り出して指定します。

上記を行うことで、端末の画面いっぱいまでコンテンツが描画されつつ、視認性や操作感を損なわない UI が実現できます。

## やりたいことの補足説明

前提として、やりたいことをもう少し具体的に説明すると、以下のような感じです。

- システム UI（ナビゲーションバー） の下にスクロールビューのコンテンツが描画され、一番下までスクロールした際に最後のアイテムがシステム UI に重ならない
- 左右にシステム UI や切り欠きがあった場合、描画が重ならない

<!-- textlint-disable ja-technical-writing/sentence-length -->

![システムUIの背景にスクロールビューが描画されている・縦画面](/images/edge-to-edge-on-compose/03-a_portrait-behind-system-ui.png =300x)
![一番したまでスクロールした際に最後のアイテムがシステムUIに重ならない・縦画面](/images/edge-to-edge-on-compose/03-b_portrait-over-scroll.png =300x)
![システムUIの背景にスクロールビューが描画されていて、左右にシステムUIや切り欠きがあった場合描画されない・横画面](/images/edge-to-edge-on-compose/04-a_landscape-behind-system-ui.png =x300)
![一番したまでスクロールした際に最後のアイテムがシステムUIに重ならない・横画面](/images/edge-to-edge-on-compose/04-b_landscape-over-scroll.png =x300)

<!-- textlint-enable ja-technical-writing/sentence-length -->

## コードを用いた詳細な解説

### Before のコード

元々以下のようなコードを書いていました。

```kotlin:MainActivity.kt
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        enableEdgeToEdge()

        setContent {
            MaterialTheme {
                MyScaffold()
            }
        }
    }
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun MyScaffold() {
    Scaffold(
        modifier = Modifier.fillMaxSize(),
        topBar = {
            TopAppBar(
                title = {
                    Text(
                        text = "Edge to edge",
                    )
                }
            )
        }
    ) { innerPadding ->
        LazyColumn(
            modifier = Modifier
                .fillMaxSize()
                .padding(innerPadding),
            contentPadding = PaddingValues(16.dp),
            verticalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            items((1..14).toList()) { index ->
                Card(
                    modifier = Modifier.fillMaxWidth()
                ) {
                    Text(
                        text = "$index",
                        modifier = Modifier.padding(16.dp),
                        fontSize = 18.sp,
                    )
                }
            }
        }
    }
}
```

上記を実行すると、やりたいことの Before の状態となります。

- システム UI（ナビゲーションバー） の領域にスクロールビューのコンテンツが描画されていない
- 左右にシステム UI や切り欠きがあった場合、描画が重なってしまう

### After のコード

#### 1. `LazyColumn` の `contentPadding` を利用してシステム UI 分の余白を設定する

前提として、`Scaffold` の内部に渡される `innerPadding` の `bottom` は、**システム UI（ナビゲーションバー）の高さ分の余白を含みます**。
そのため、この値をスクロールビューの余白にうまく適用してやることで、やりたいことが実現できます。

![ScaffoldのinnerPaddingのサイズ](/images/edge-to-edge-on-compose/scaffold-inner-padding.gif)

Before のコードのように `LazyColumn` の `modifier` で余白を設定した場合、**スクロールコンテンツが描画される領域（ビューポート）の外側に余白が適用**されます。
そのため、スクロールコンテンツの描画領域がシステム UI に重ならないような状態となっていました。

![modifierで余白指定した場合のスクロールの様子](/images/edge-to-edge-on-compose/scroll-abstract-image_01-modifier.gif)

レイヤー構造としては、以下のようなイメージです。

![modifierで余白指定した場合のレイヤー構造](/images/edge-to-edge-on-compose/layers-abstract-image_01-modifier.gif)

一方で、`LazyColumn` の `contentPadding` を利用すると、**スクロールコンテンツの中身に余白が適用**されます。
これを利用することで、スクロールコンテンツの中身にシステム UI の高さ分だけ余白を設定できます。

![contentPaddingで余白指定した場合のスクロールの様子](/images/edge-to-edge-on-compose/scroll-abstract-image_02-contentPadding.gif)

レイヤー構造としては、以下のようなイメージです。

![contentPaddingで余白指定した場合のレイヤー構造](/images/edge-to-edge-on-compose/layers-abstract-image_02-contentPadding.gif)

これらを踏まえると、以下の方針とすることで、やりたいことが実現できます。

- `LazyColumn` の `modifier` における余白設定をしない。これにより、システム UI の領域にもスクロールコンテンツの描画領域が広がる。
- `LazyColumn` の `contentPadding` を利用してシステム UI の高さ分だけ余白を設定する。これにより、スクロールコンテンツの中身がシステム UI に重ならないところまでスクロールできるようになる。

```diff kotlin:MainActivity.kt
        LazyColumn(
            modifier = Modifier
                .fillMaxSize()
-                .padding(innerPadding),
                // ...
-            contentPadding = PaddingValues(16.dp),
+            contentPadding = PaddingValues(
+                start = 16.dp + innerPadding.calculateStartPadding(
+                    LocalLayoutDirection.current
+                ),
+                top = 16.dp + innerPadding.calculateTopPadding(),
+                end = 16.dp + innerPadding.calculateStartPadding(
+                    LocalLayoutDirection.current
+                ),
+                bottom = 16.dp + innerPadding.calculateBottomPadding(),
+            ),
```

#### 2. `WindowInsets.safeDrawing` により切り欠きを避けて描画する

`WindowInsets.safeDrawing` には、ノッチやパンチホールなどの「切り欠き」領域を避けて描画するための情報が含まれています。

![WindowInsets.safeDrawingのサイズ](/images/edge-to-edge-on-compose/window-insets-safe-drawing.gif)

これを利用し、上下左右のうち必要な要素だけを取り出して、`Modifier.windowInsetsPadding` に渡すことで、切り欠きを避けて描画できます。

```diff kotlin:MainActivity.kt
    Scaffold(
        modifier = Modifier.fillMaxSize(),
        topBar = {
            TopAppBar(
+                modifier = Modifier
+                    .windowInsetsPadding(
+                        WindowInsets.safeDrawing.only(WindowInsetsSides.Start)
+                    )
+                    .windowInsetsPadding(
+                        WindowInsets.safeDrawing.only(WindowInsetsSides.End)
+                    ),
                // ...
            )
        }
    ) { innerPadding ->
        LazyColumn(
            modifier = Modifier
                // ...
+                .windowInsetsPadding(
+                    WindowInsets.safeDrawing.only(WindowInsetsSides.Start)
+                )
+                .windowInsetsPadding(
+                    WindowInsets.safeDrawing.only(WindowInsetsSides.End)
+                ),
            // ...
```

### 完全なコード

最終的に以下のようなコードになりました。

```kotlin:MainActivity.kt
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        enableEdgeToEdge()

        setContent {
            MaterialTheme {
                MyScaffold()
            }
        }
    }
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun MyScaffold() {
    Scaffold(
        modifier = Modifier.fillMaxSize(),
        topBar = {
            TopAppBar(
                modifier = Modifier
                    .windowInsetsPadding(
                        WindowInsets.safeDrawing.only(WindowInsetsSides.Start)
                    )
                    .windowInsetsPadding(
                        WindowInsets.safeDrawing.only(WindowInsetsSides.End)
                    ),
                title = {
                    Text(
                        text = "Edge to edge",
                    )
                }
            )
        }
    ) { innerPadding ->
        LazyColumn(
            modifier = Modifier
                .fillMaxSize()
                .windowInsetsPadding(
                    WindowInsets.safeDrawing.only(WindowInsetsSides.Start)
                )
                .windowInsetsPadding(
                    WindowInsets.safeDrawing.only(WindowInsetsSides.End)
                ),
            contentPadding = PaddingValues(
                start = 16.dp + innerPadding.calculateStartPadding(
                    LocalLayoutDirection.current
                ),
                top = 16.dp + innerPadding.calculateTopPadding(),
                end = 16.dp + innerPadding.calculateStartPadding(
                    LocalLayoutDirection.current
                ),
                bottom = 16.dp + innerPadding.calculateBottomPadding(),
            ),
            verticalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            items((1..14).toList()) { index ->
                Card(
                    modifier = Modifier.fillMaxWidth()
                ) {
                    Text(
                        text = "$index",
                        modifier = Modifier.padding(16.dp),
                        fontSize = 18.sp,
                    )
                }
            }
        }
    }
}
```

## 参考

https://developer.android.com/design/ui/mobile/guides/layout-and-content/edge-to-edge?hl=ja

https://qiita.com/Nabe1216/items/6fd9e2293f7ae109150a

https://developers-jp.googleblog.com/2019/10/gesture-navigation-handling-visual-overlaps.html

https://developer.android.com/develop/ui/compose/modifiers?hl=ja

https://developer.android.com/develop/ui/views/layout/edge-to-edge?hl=ja#enable-edge-to-edge-display
