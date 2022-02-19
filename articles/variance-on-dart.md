---
title: "共変性と反変性の簡単な説明とDartにおける例"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dart", "flutter"]
published: false
---

# 一言でまとめると

リスコフの置換原則は前提とすると、

基底型でできることは派生型もでき、そこから導き出される定理。

派生型の引数を持つ関数型やクラスの型に、基底型の引数を持つ関数やクラスのインスタンスが代入できる
→ 引数の派生型が基底型に置き換えられる。
→ 引数に**反変性**がある

基底型の戻り値を持つ関数型やクラスの型に、派生型の戻り値を持つ関数やクラスのインスタンスが代入できる
→ 戻り値の基底型が派生型に置き換えられる
→ 戻り値に**共変性**がある

個人的な感想として、定義方法を変えることで反変の関係だったものを共変の関係に変えることもでき、また、共変の関係の方が設計しやすいと感じるので、あまり反変の関係は意識することないかもしれない。

基底型には派生型を代入できる互換性があり、

これと同じ方向に、派生型に対して互換性を持つことを「共変」
反対の方向に、基底型に対して互換性を持つことを「反変」
どちらの互換もない場合は「不変」

## 共変性

基底クラスに派生クラスを代入しても成り立つ性質。

外部に基底クラスしか返さないため、成り立つ。

```dart:repository.dart
class Repository {
  const Repository(Api api);
  final Api api;
}
```

```dart:api.dart
// 基底クラス
abstract class Api {
  Future<String> fetch();
}

// 派生クラスその1
class ApiProd implements Api {
  @override
  Future<String> fetch() async {
    final response = await http.get('https://www.example.com/data');
    return response.response.body;
  }
}

// 派生クラスその2
class ApiMock implements Api {
  @override
  Future<String> fetch() async {
    return '{"code":"OK"}';
  }
}
```

```dart:main.dart
final Repository repository;
if (F.flavor == Flavor.mock) {
  repository = Repository(
    // 共変: 基底クラス [Api] の引数に派生クラス [ApiMock] を指定している
    ApiMock(),
  );
} else {
  repository = Repository(
    // 共変: 基底クラス [Api] の引数に派生クラス [ApiProd] を指定している
    ApiProd(),
  );
}
```

## 反変性

派生クラスに基底クラスを代入しても成り立つ性質。

内部で基底クラスのメソッドしか利用していないため、成り立つ。
基底クラスが持つメソッドは派生クラスも持つから、成り立つ。
派生クラスが基底クラスの振る舞いをできるから、成り立つ。
リスコフの置換原則が成り立つ。

```dart:article.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'article.freezed.dart';

/// 記事の内容を格納するクラス
@freezed
class Article with _$Article {
  /// テキストのみの記事
  const factory Article.text({
    required String id,
    required String content,
  }) = TextArticle;

  /// 写真のみの記事
  const factory Article.picture({
    required String id,
    required Uri url,
  }) = PictureArticle;
}

/// IDとタイトルを合わせてBase64化し、ローカルにキャッシュするファイル名を生成
String generateFileNameFromArticle(Article article) {
  final seed = '${article.id}:${article.title}';
  final bytes = utf8.encode(seed);
  return base64.encode(bytes);
}
```

```dart:main.dart
/// 派生クラス [TextArticle] をフェッチし、ローカルにキャッシュして返すメソッド
Future<TextArticle> _fetchTextArticle({
  required String id,
  required String Function(TextArticle) generateFileName,
}) async {
  final article = await _remoteDataSource.fetchTextArticle(id: id);

  final fileName = generateFileName(article);
  await _localDataSource.save(article: article, fileName: fileName);

  return article;
}

await _fetchTextArticle(
  id: id,
  // 反変: 引数が派生クラス [TextArticle] のコールバック引数に、
  //  引数が基底クラス [Article]のコールバックを指定している
  generateFileName: generateFileNameFromArticle,
);
```
