---
title: "Flutterの開発ポリシー"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter"]
published: false
---

# デザインシステム

- 1 つ以上の要素を囲むウィジェットのサイズ指定は可能な限り避け、内部ウィジェットの余白や親ウィジェットによる相対配置の結果自動でサイズが決定されるようにする
  - 各端末の画面サイズや、端末の文字サイズ設定、データによるレイアウト崩れを避けるため
- 色は必ず `Theme.of(context).primaryColor` のように定義されたものを利用する
  - デザインシステムに則った色のみが利用されるようにするため
- テキストスタイルは必ず `Theme.of(context).textThemes.caption1` のように定義されたものを利用する
  - デザインシステムに則った色のみが利用されるようにするため
- 画面に表示する文字列は `*.arb` ファイルに定義して利用する
  - 表記ゆれに気付きやすくするため
- 変動する値を含む文字列も `*.arb` ファイルに定義して利用することを優先する
  - 表記ゆれに気付きやすくするため
  - 部分的に太字などフォーマットが異なる場合は、分けて定義するしかない

# UI と状態管理

- State は画面生成に必要最小限のデータを持つようにする
- State と ViewModel のメンバー変数は重複がないようにする
- StatefulWidget または ConsumerStatefulWidget では、プライベートメソッドに context を渡さないようにする
  - これらの中ではいつでも context を取得できるため

# UI の構築

## プライベートの切り出しはウィジェットとして切り出す

Flutter のフレームワーク側でリビルドを必要最小限に抑えやすくなるため

### 一時変数は、格納されるウィジェットの型を接尾語として優先的に利用する

中身が想像しやすくするため

### Screen で State データの一時変数を作成する際の名前は、 state とする

型に名前を合わせて、中身を想像しやすくするため

**BAD**

```dart
class _HogeScreen extends ConsumerWidget {
  // ...

  final AutoDisposeStateNotifierProvider<WorkspaceViewModel, WorkspaceState>
      viewModel;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final viewModel = ref.watch(viewModel); // BAD
```

**GOOD**

```dart
class _HogeScreen extends ConsumerWidget {
  // ...

  final AutoDisposeStateNotifierProvider<WorkspaceViewModel, WorkspaceState>
      viewModel;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(viewModel); // GOOD
```

## コンポーネントには外側の余白を持たせない

再利用性を高めるため

**BAD**

```dart
class HogeButton extends StatelessWidget {
  // ...

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 16), // BAD
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
class HogeButton extends StatelessWidget {
  // ...

  @override
  Widget build(BuildContext context) {
    return ElevatedButton(
      onPressed: onPressed,
      child: child,
    );
  }
}

  // コンポーネントの利用箇所
  Center(
    child = Padding(
      padding: const EdgeInsets.symmetric(horizontal: 16), // GOOD
      child: HogeButton(
        onPressed: () => Navigator.pop(context),
        child: const Text('pop'),
      ),
    ),
  );
```

## スクロール可能なウィジェットが画面下側まで表示されている場合は、セーフエリア外までスクロール内容が表示されるようにし、スクロールの下部にセーフエリア外のサイズを余白としてとる

- 画面サイズ最大までスクロール表示が見えた方がユーザビリティが上がるため
- 余白を取るのは、スクロールを最下部まで行った際に部品がセーフエリア外のホームバーなどと被ってタップできなくなるのを防ぐため

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

# コンポーネント設計

- コンポーネントでは UI の表示のみを実装し、画面遷移などのイベントは外部から渡せるようにしておく
  - 責務を整理し、再利用性を高めるため

# テストコード

- 実装とセットでテストコードを書く
  - 実装者が対象コードのロジックを冷静に見直すことで、実装時点での間違いや考慮もれを発見するため
  - デグレ発生時に早期発見するため

# 命名

- メソッド名から容易に想像がつく引数以外は、名前付きにしておく
  - 可読性を高めるため
- 利用者が引数の内容を明確に認知し、利用ミスを防ぐため
- 無名関数における利用していない引数は `_` で無効化する
  - 利用しないことを最初に明記し、可読性を高めるため
- “info” という単語は利用は避け、より具体的な語彙を利用する
  - 全てのクラスやメンバー変数は何かしらの情報を持つという前提があり、”info”という単語がその中身を説明することに寄与しないため

# コメント

- コードを見ても分からない背景や意図をマストで優先的に書く
- コードを見る際に補足となる説明事項は書いてもいい
- コードを見るとすぐに分かることは書かない

# 一般

- インデントが深くならない書き方を優先する
  - 可読性を高めるため
