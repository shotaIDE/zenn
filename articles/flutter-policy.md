---
title: "Flutterの開発ポリシー"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter"]
published: true
---

Flutter でのアプリ開発におけるポリシーをメモしてみます。
静的解析ではチェックできず、人の目でチェックする必要がある内容を対象としています。

# デザインシステム

## 1 つ以上の要素を囲むウィジェットのサイズは、内部ウィジェットのマージンや親ウィジェット内の相対関係により決定されるようにする

固定値での指定は可能な限り避ける

**理由**

各端末の画面サイズや、端末の文字サイズ設定、データによるレイアウト崩れを避けるため

## 色は必ず `Theme.of(context).primaryColor` のように定義されたものを利用する

**理由**

- デザインシステムで定義された色のみが適用されるようにするため
- ダークモードのように実行時に動的に色が変わる挙動に対応するため

## テキストスタイルは必ず `Theme.of(context).textThemes.caption1` のように定義されたものを利用する

**理由**

デザインシステムで定義された Typography 定義のスタイルのみが適用されるようにするため

## 画面に表示する文字列は `*.arb` ファイルに定義して利用する

- 変数する含む文字列も `*.arb` ファイルに定義して利用することを優先する
- 部分的に太字などフォーマットが異なる場合は、分けて定義するしかない

**理由**

- 表記ゆれに気付きやすくするため
- 多言語対応するため

# UI 設計

## 画面と状態管理

### State は画面生成に必要最小限のデータを持つようにする

### State と ViewModel のメンバー変数は重複がないようにする

## 画面の構築

### UI 要素をファイル内で切り出す際は、プライベートウィジェットとして切り出す

**理由**

Flutter のリビルドを必要最小限に抑える仕組みの恩恵が受けやすいため

### ウィジェットのローカル変数への格納 → ウィジェットを余白をつけて組み合わせる、の２ステップで段落分けして記載する

**理由**

可読性を向上させるため

### ローカル変数名は、格納されるウィジェットの型を接尾語にする

**理由**

中身が想像しやすくするため

:::details コードサンプル

**BAD**

```dart
const nameLabel = Text('Alice');
```

**GOOD**

```dart
const nameText = Text('Alice');
```

:::

### ウィジェット間の余白は、ウィジェットの上または左に付与する

**理由**

上から順に読んでいく際に、構造が理解しやすいため

:::details コードサンプル

**BAD**

```dart
const firstText = Text('1st');
const secondText = Text('2nd');
const thirdText = Text('3rd');

final body = Column(
  children: [
    const Padding(
      padding: EdgeInsets.only(bottom: 16), // BAD
      child: firstText,
    ),
    Row(
      children: const [
        Padding(
          padding: EdgeInsets.only(right: 16), // BAD
          child: secondText,
        ),
        thirdText,
      ],
    ),
  ],
);
```

**GOOD**

```dart
const firstText = Text('1st');
const secondText = Text('2nd');
const thirdText = Text('3rd');

final body = Column(
  children: [
    firstText,
    Padding(
      padding: const EdgeInsets.only(top: 16), // GOOD
      child: Row(
        children: const [
          secondText,
          Padding(
            padding: EdgeInsets.only(left: 16), // GOOD
            child: thirdText,
          ),
        ],
      ),
    ),
  ],
);
```

:::

### スクロール可能なウィジェットはセーフエリア外までスクロール内容が表示されるようにする

この際、スクロールの下部にセーフエリア外のサイズを余白としてとる。スクロールを最下部まで行った際に部品がセーフエリア外のホームバーなどと被ってタップできなくなるのを防ぐため

**理由**

画面サイズ最大までスクロール表示が見えた方がユーザビリティが高いため

:::details コードサンプル

**BAD**

```dart
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(),
      body: SafeArea( // BAD
        child: SingleChildScrollView(
          child: Column(
            children: List.generate(
              20,
              (index) => Center(
                child: SizedBox(
                  width: MediaQuery.of(context).size.width * 0.8,
                  child: OutlinedButton(
                    onPressed: () {},
                    child: Text('Button ${index + 1}'),
                  ),
                ),
              ),
            ).toList(),
          ),
        ),
      ),
    );
  }
```

![](/images/flutter-policy/01_scroll-view-with-safe-area.gif)

**GOOD**

```dart
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(),
      body: SafeArea(
        bottom: false, // GOOD
        child: SingleChildScrollView(
          padding: EdgeInsets.only(
            bottom: MediaQuery.of(context).viewPadding.bottom, // GOOD
          ),
          child: Column(
            children: List.generate(
              20,
              (index) => Center(
                child: SizedBox(
                  width: MediaQuery.of(context).size.width * 0.8,
                  child: OutlinedButton(
                    onPressed: () {},
                    child: Text('Button ${index + 1}'),
                  ),
                ),
              ),
            ).toList(),
          ),
        ),
      ),
    );
  }
```

![](/images/flutter-policy/02_scroll-view-without-safe-area.gif)

:::

### `StatefulWidget` または `ConsumerStatefulWidget` 内では、プライベートメソッドに `context` を渡さないようにする

**理由**

プライベートゲッターで `context` を取得できるため

### State データのローカル変数を作成する際の名前は、 `state` とする

**理由**

型に名前を合わせて、中身を想像しやすくするため

:::details コードサンプル

**BAD**

```dart
class MyScreen extends ConsumerWidget {
  // ...

  final AutoDisposeStateNotifierProvider<MyViewModel, MyState> viewModel;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final viewModel = ref.watch(viewModel); // BAD
```

**GOOD**

```dart
class MyScreen extends ConsumerWidget {
  // ...

  final AutoDisposeStateNotifierProvider<MyViewModel, MyState> viewModel;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(viewModel); // GOOD
```

:::

## コンポーネント

### 外側に余白を持たせない

**理由**

再利用性を高めるため

:::details コードサンプル

**BAD**

```dart
class MyButton extends StatelessWidget {
  // ...

  @override
  Widget build(BuildContext context) {
    return Padding( // BAD
      padding: const EdgeInsets.symmetric(horizontal: 16),
      child: ElevatedButton(
        onPressed: onPressed,
        child: child,
      ),
    );
  }
}
```

**GOOD**

```dart
class MyButton extends StatelessWidget {
  // ...

  @override
  Widget build(BuildContext context) {
    return ElevatedButton(
      onPressed: onPressed,
      child: child,
    );
  }
}

// ...

// コンポーネントの利用箇所
Center(
  child = Padding( // GOOD
    padding: const EdgeInsets.symmetric(horizontal: 16),
    child: MyButton(
      onPressed: () => Navigator.pop(context),
      child: const Text('pop'),
    ),
  ),
);
```

:::

### UI 表示のみを実装し、画面遷移などのイベントは外部から渡せるようにする

**理由**

責務を UI 表示のみにすることで、再利用性を高めるため

# テストコード

## 実装とセットでテストコードを書く

- 実装者が対象コードのロジックを冷静に見直すことで、実装時点での間違いや考慮もれを発見するため
- デグレを早期発見するため

# 一般

## 命名

### メソッド名から容易に想像がつく引数以外は、名前付きにしておく

**理由**

- 可読性を高めるため
- 利用者が引数の内容を明確に認知し、利用ミスを防ぐため

### 無名関数における利用していない引数は `_` で無効化する

**理由**

利用しないことを最初に明記し、可読性を高めるため

## “info” という単語は利用は避け、より具体的な語彙を利用する

**理由**

全てのクラスやメンバー変数は何かしらの情報を持つという前提があり、”info”という単語がその中身を説明することに寄与しないため

## コメント

### コードを見ても分からない背景や意図をマストで優先的に書く

### コードを見る際に補足となる説明事項は書いてもいい

### コードを見るとすぐに分かることは書かない

## その他

### インデントが深くならない書き方を優先する

**理由**

可読性を高めるため
