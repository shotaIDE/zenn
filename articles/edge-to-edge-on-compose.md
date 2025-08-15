---
title: "Composeで没入感のあるエッジツーエッジのUIを実現する"
emoji: "🌟"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["android", "compose"]
published: false
---

## やりたいこと

エッジツーエッジで没入感のあるスクロール画面を実現したい。

| 画面の向き | Before                                                         | After                                                         |
| ---------- | -------------------------------------------------------------- | ------------------------------------------------------------- |
| 縦         | ![](/images/edge-to-edge-on-compose/01-a_portrait-before.gif)  | ![](/images/edge-to-edge-on-compose/01-b_portrait-after.gif)  |
| 横         | ![](/images/edge-to-edge-on-compose/02-a_landscape-before.gif) | ![](/images/edge-to-edge-on-compose/02-b_landscape-after.gif) |

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

残件

- [ ] Android 15 未満でエッジツーエッジを適用する方法を調査する

https://developer.android.com/develop/ui/compose/system/material-insets?hl=ja#override-default

https://developers-jp.googleblog.com/2019/10/gesture-navigation-handling-visual-overlaps.html

`modifier` におけるパディングは、`LazyColumn` のスクロール領域外に対して適用されます。
一方で、 `contentPadding` によるパディングは、`LazyColumn` のスクロール領域に対して適用されます。
そのため、`contentPadding` によるパディングを設定することが適切です。

`WindowInsets.safeDrawing` を利用することで、切り欠きを避けて描画できる。

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
