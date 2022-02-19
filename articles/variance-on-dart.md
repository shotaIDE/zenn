---
title: "共変性と反変性の簡単な説明とDartにおける例"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dart", "flutter"]
published: true
---

# 共変性と反変性の簡単な説明

以下、基本原則（リスコフの置換原則）を前提とし、

:::message

基底型でできることは派生型でもでき、基底型変数は中身を派生型に置き換えても動作する

:::

原則と同じく、基底型変数を派生型に置き換えられることを**共変**(きょうへん)と呼ぶ。

一方で、呼び出し構造により、見かけ上、互換性の向きが反対になっていて、派生型変数に基底型に置き換えられることを**反変**(はんぺん)と呼ぶ。

## 具体的な用例

基底型の戻り値を持つ関数型やクラスの型に、派生型の戻り値を持つ関数やクラスのインスタンスが代入できる
→ 戻り値の基底型が派生型に置き換えられる
→ 戻り値に**共変性**がある

派生型の引数を持つ関数型やクラスの型に、基底型の引数を持つ関数やクラスのインスタンスが代入できる
→ 引数の派生型が基底型に置き換えられる。
→ 引数に**反変性**がある

# Dart における例

## 共変

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

## 反変

:::message

Dart に Union class を導入する言語パッチライブラリ[freezed](https://pub.dev/packages/freezed)の利用を前提としています。

:::

```dart:article.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'article.freezed.dart';

/// 記事の内容を格納するUnionクラス
///
/// 基底クラス
@freezed
class Article with _$Article {
  /// テキストのみの記事
  ///
  /// 派生クラスその1
  const factory Article.text({
    required String id,
    required String content,
  }) = TextArticle;

  /// 写真のみの記事
  ///
  /// 派生クラスその2
  const factory Article.picture({
    required String id,
    required Uri url,
  }) = PictureArticle;
}

/// IDとタイトルを合わせてBase64化し、ローカルにキャッシュするファイル名を生成
///
/// 派生クラス間で生成方法は変わらないため、基底クラス [Article] を引数に取る
String generateFileNameFromArticle(Article article) {
  final seed = '${article.id}:${article.title}';
  final bytes = utf8.encode(seed);
  return base64.encode(bytes);
}
```

```dart:main.dart
/// 派生クラス [TextArticle] をフェッチし、ローカルにキャッシュして返すメソッド
///
/// ファイル名生成の関数の引数は、派生クラス [TextArticle] を引数に取る
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

_個人的な感想として、定義方法を変えることで反変の関係だったものを共変の関係に変えることもでき、また、共変の関係の方が直感的で設計しやすいと感じるので、あまり反変の関係で設計することはないように思う。_

# 参考

https://medium.com/dartlang/dart-declaration-site-variance-5c0e9c5f18a5

https://qiita.com/CodeOne/items/730480791c2e98b40d66#%E5%8F%8D%E5%A4%89%E6%80%A7%E3%81%8C%E3%82%B5%E3%83%9D%E3%83%BC%E3%83%88%E3%81%95%E3%82%8C%E3%82%8B%E3%81%A8

https://zenn.dev/suin/scraps/adf781330c9888

https://qiita.com/mkosuke/items/42c19d7edbf111f7fb71#futureort%E3%81%AF%E6%88%BB%E3%82%8A%E5%80%A4%E3%81%AB%E4%BD%BF%E7%94%A8%E3%81%97%E3%81%AA%E3%81%84
