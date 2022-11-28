---
title: "私のFlutterのアプリ開発ポリシー"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter"]
published: true
---

Flutter でのアプリ開発におけるポリシーをメモしてみます。
静的解析ではチェックできず、人の目でチェックする必要がある内容を対象としています。

# UI 設計

## デザインシステム

### 1 つ以上の要素を囲むウィジェットのサイズは、内部ウィジェットのマージンや親ウィジェット内の相対関係により決定されるようにする

固定値での指定は可能な限り避ける

_理由_

各端末の画面サイズや、端末の文字サイズ設定、データ長によるレイアウト崩れを避けるため

:::details コードサンプル

**BAD**

```dart
return Scaffold(
  appBar: AppBar(),
  body: const SizedBox(
    width: 320, // BAD
    child: ListTile(
      title: Text('Alice'),
    ),
  ),
);
```

**GOOD**

```dart
return Scaffold(
  appBar: AppBar(),
  body: const Padding(
    padding: EdgeInsets.symmetric(horizontal: 16), // GOOD
    child: ListTile(
      title: Text('Alice'),
    ),
  ),
);
```

:::

### 色は `Theme` に定義されているものを利用する

_理由_

- デザインシステムで定義された色のみが適用されるようにするため
- ダークモードのように実行時に動的に色が変わる挙動に対応するため

:::details コードサンプル

**BAD**

```dart
final colorContainer = Container(
  color: Colors.blue, // BAD
);
```

**GOOD**

```dart
final colorContainer = Container(
  color: Theme.of(context).primaryColor, // GOOD
);
```

:::

### テキストスタイルは `Theme` に定義されているものを利用する

_理由_

デザインシステムで定義された Typography のスタイルのみが適用されるようにするため

:::details コードサンプル

**BAD**

```dart
const nameText = Text(
  'Alice',
  style: TextStyle(fontSize: 14), // BAD
);
```

**GOOD**

```dart
final nameText = Text(
  'Alice',
  style: Theme.of(context).textTheme.caption, // GOOD
);
```

:::

### 画面に表示する文字列は `*.arb` ファイルに定義して利用する

- 変数する含む文字列も `*.arb` ファイルに定義して利用することを優先する
- 部分的に太字などフォーマットが異なる場合は、分けて定義するしかない

_理由_

- 表記ゆれに気付きやすくするため
- 多言語対応するため

:::details コードサンプル

**BAD**

```dart
const appInfoText = Text('アプリ情報'); // BAD
final nameText = Text('名前: $name'); // BAD
```

**GOOD**

```arb:app_ja.arb
"appInfo": "アプリ情報",
"nameFormat": "名前: {name}",
"@nameFormat": {
    "placeholders": {
        "name": {
            "type": "String"
        }
    }
},
```

```dart
import 'package:flutter_gen/gen_l10n/app_localizations.dart';

final appInfoText = Text(S.of(context)!.appInfo); // GOOD
final nameText = Text(S.of(context)!.nameFormat(name)); // GOOD
```

:::

## 画面の構築

### データ数が多いものやデータ長が長いもの、複数のパターンが存在するデータの全てに対して、レイアウト崩れが発生しないことを確認する

以下に注意する

- enum クラスの全てのケース
- Union class の全てのサブクラス

_理由_

画面設計時の考慮もれに気づくため

### 複数のウィジェットをグルーピングしてファイル内で切り出す際は、プライベートウィジェットとして切り出す

_理由_

Flutter のリビルドを必要最小限に抑える仕組みの恩恵を最大限受けるため

:::details コードサンプル

**BAD**

```dart:member_screen.dart
class MemberScreen extends StatelessWidget {
  const MemberScreen({
    Key? key,
    required this.member,
  }) : super(key: key);

  final String member;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(),
      body: Column(
        children: [
          _headerTile(context), // BAD
          Container(
            alignment: Alignment.center,
            child: Text(member),
          )
        ],
      ),
    );
  }

   // BAD
  Widget _headerTile(BuildContext context) {
    return ListTile(
      title: const Text('Header'),
      onTap: () => Navigator.push(context, MemberScreen.route()),
    );
  }
}
```

**GOOD**

```dart:member_screen.dart
class MemberScreen extends StatelessWidget {
  const MemberScreen({
    Key? key,
    required this.member,
  }) : super(key: key);

  final String member;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(),
      body: Column(
        children: [
          const _HeaderTile(), // GOOD
          Container(
            alignment: Alignment.center,
            child: Text(member),
          )
        ],
      ),
    );
  }
}

 // GOOD
class _HeaderTile extends StatelessWidget {
  const _HeaderTile({
    Key? key,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return ListTile(
      title: const Text('Header'),
      onTap: () => Navigator.push(context, MemberScreen.route()),
    );
  }
}
```

:::

### 複雑な画面では、ウィジェットのローカル変数への格納 → 余白をつけて組み立て、の 2 ステップで処理を分ける

_理由_

処理のかたまりごとに一種類の処理のみを記述することで、可読性を向上させるため

:::details コードサンプル

**BAD**

```dart
final body = Column(
  children: const [
    Text('1st'),
    Padding(
      padding: EdgeInsets.only(top: 16),
      child: Text('2nd'),
    ),
  ],
);
```

**GOOD**

```dart
// ウィジェットのローカル変数への格納
const firstText = Text('1st');
const secondText = Text('2nd');

// 余白をつけて組み立て
final body = Column(
  children: const [
    firstText,
    Padding(
      padding: EdgeInsets.only(top: 16),
      child: secondText,
    ),
  ],
);
```

:::

### ローカル変数名は、格納されるウィジェットの型を接尾語にする

_理由_

中身を想像しやすくし、可読性を向上させるため

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

_理由_

- 上から順に読んでいく際に構造が理解しやすくなり、可読性の向上が見込めるため
- 余白の付与する位置を統一することで、可読性の向上が見込めるため

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

### スクロール可能なウィジェットではセーフエリア外までスクロール内容が表示されるようにする

この際、スクロールウィジェットの内側の下部にセーフエリア外の下部領域と一致する領域を余白としてとる。スクロールを最下部までスクロールした際にスクロール内部のウィジェットがセーフエリア外のホームバーなどと被ってしまうのを防ぐため

_理由_

画面サイズ最大までスクロール領域を広げてユーザビリティを高めるため

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

_理由_

プライベートゲッターで `context` を取得できるため

:::details コードサンプル

**BAD**

```dart
class MyScreen extends StatefulWidget {
  // ...

  Future<void> _launchSettingsScreen(
    BuildContext context, // BAD
  ) async {
    await Navigator.push<void>(
      context,
      SettingsScreen.route(),
    );
  }
```

**GOOD**

```dart
class MyScreen extends StatefulWidget {
  // ...

  Future<void> _launchSettingsScreen() async {
    await Navigator.push<void>(
      context, // GOOD
      SettingsScreen.route(),
    );
  }
```

:::

### State データのローカル変数を作成する際の名前は、 `state` とする

_理由_

中身を想像しやすくし、可読性を向上させるため

:::details コードサンプル

**BAD**

```dart
class MyScreen extends ConsumerWidget {
  // ...

  final AutoDisposeStateNotifierProvider<MyViewModel, MyState> _viewModel;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final viewModel = ref.watch(_viewModel); // BAD
```

**GOOD**

```dart
class MyScreen extends ConsumerWidget {
  // ...

  final AutoDisposeStateNotifierProvider<MyViewModel, MyState> _viewModel;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(_viewModel); // GOOD
```

:::

## コンポーネント

### 外側の余白を持たせない

_理由_

外側の余白は利用側に委ねることで、再利用性を高めるため

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

_理由_

画面遷移のイベントは利用側に委ねることで、再利用性を高めるため

## 画面と状態管理

### State は画面生成に必要最小限のデータを持つようにする

_理由_

State を見るだけで画面のパターンを想像できるようにして、可読性を高めるため

### State と ViewModel のメンバー変数は重複がないようにする

_理由_

変数の二重管理の発生により処理が不必要に複雑になるのを防ぐため

# 一般

## 命名

### メソッド名から容易に想像がつく引数以外は、名前付きにしておく

_理由_

- 可読性を高めるため
- 利用者が引数の内容を明確に認知し、利用ミスを防ぐため

### 無名関数における利用していない引数は `_` で無効化する

_理由_

利用しないことを最初に明記し、可読性を高めるため

### “info” という単語は利用は避け、より具体的な語彙を利用する

_理由_

全てのクラスやメンバー変数は何かしらの情報を持つという前提があり、”info”という単語がその中身を説明することに寄与しないため

## コメント

### コメントはコードを見ても分からないことを書き、分かることは書かないようにする

- 背景や意図をマストで優先的に書く
- 補足となる説明事項は書いてもいい
- コードを日本語訳しただけのようなすぐに分かることは書かない

_理由_

コードの読みやすさの向上を図りつつ、コメントのメンテナンスコストを不必要に上げないため

## その他

### メソッドやメンバー変数の公開範囲や可変性、null 許容性は最大限縛る

- プライベートの方がパブリックより望ましい
- `final` → `late final` → `var`(変数)の順で望ましい
- not-null の方が nullable より望ましい

_理由_

参照される範囲や可変性を狭めた前提でコードを読むことができ、可読性の向上が図れるため

### インデントが深くならない書き方を優先する

_理由_

インデントの視認性が低くならないようにして可読性を高めるため

### デバッグ用のコードやコメントアウトしたコードは残さない

PR レビューはこれらのコードは削除しておく

_理由_

動作していないコードは、読む際ノイズとなり、削除しておくことで可読性が高まるため
