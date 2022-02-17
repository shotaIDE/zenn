---
title: "Flutterの開発ポリシー"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter"]
published: false
---

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

### StatefulWidget または ConsumerStatefulWidget では、プライベートメソッドに context を渡さないようにする

**理由**

これらの中ではいつでも context を取得できるため

## 画面の構築

### プライベートの切り出しはウィジェットとして切り出す

**理由**

Flutter のフレームワーク側でリビルドを必要最小限に抑えやすくなるため

### 一時変数は、格納されるウィジェットの型を接尾語として優先的に利用する

**理由**

中身が想像しやすくするため

### State データの一時変数を作成する際の名前は、 `state` とする

**理由**

型に名前を合わせて、中身を想像しやすくするため

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

### スクロール可能なウィジェットはセーフエリア外までスクロール内容が表示されるようにする

この際、スクロールの下部にセーフエリア外のサイズを余白としてとる。スクロールを最下部まで行った際に部品がセーフエリア外のホームバーなどと被ってタップできなくなるのを防ぐため

**理由**

画面サイズ最大までスクロール表示が見えた方がユーザビリティが上がるため

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

## コンポーネント

### コンポーネントには外側の余白を持たせない

**理由**

再利用性を高めるため

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

### コンポーネントでは UI の表示のみを実装し、画面遷移などのイベントは外部から渡せるようにしておく

**理由**

責務を整理し、再利用性を高めるため

# テストコード

## 実装とセットでテストコードを書く

- 実装者が対象コードのロジックを冷静に見直すことで、実装時点での間違いや考慮もれを発見するため
- デグレ発生時に早期発見するため

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
