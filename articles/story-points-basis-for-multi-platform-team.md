---
title: "モバイル、Web、API での開発チームにおけるストーリーポイントの基準"
emoji: "🤖"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["アジャイル"]
published: false
---

# 前提

- モバイル、Web、API での開発チーム

理想的には、モバイル、Web、API での開発チームにおいて、ストーリーポイントの基準を統一していることが望ましい。

ある程度抽象的なレベルで実装に何が必要かということを考える。

# 時間見積もりではなく、相対見積もりがいいのか？

一説によると、開発者のパフォーマンスは 10 倍違うと言われています。
そのため、同じタスクでも、開発者によってかかる時間が異なります。

時間は人に依存した単位。チーム全体のパフォーマンスを上げていきたい。

# モバイルと Web、API で揃えた方がいいのか？

理想的には、モバイル、Web、API の垣根なく、チーム一丸となって開発を進めることが望ましいです。

そのため、例えばモバイルの開発が遅れている場合、Web、API の開発者がフォローに入ったりして、チーム全体でスプリントゴールに向かいやすい状況を作ることができます。

そうした理想状態を見据えて、ストーリーポイントの基準を揃えることにしました。

## フィボナッチ数列を使うか？

大きくなるほど、不透明な部分が大きくなるため、フィボナッチ数列を使うことが望ましいと考えています。

# 具体的な基準

## 1

- 極度に少ない労力で完了できる
- コード修正時の懸念がない（ほぼ言い切れる）
- 具体的な実装内容の見通しが完全に立っている

## 2

- 少ない労力で完了できる
- コード修正時の懸念がほとんどない。
- 具体的な実装内容の見通しがほぼ立っている

## 3

- ある程度少ない労力で完了できる
- コード修正時の懸念はあるものの、大きく懸念は増えなさそう
- 具体的な実装内容の見通しは半分くらいは立っている。細かい部分はやりながら考える

## 5

- 普通に労力をかければ完了できる。ハードな部分もあるかも
- コード修正時の懸念はあるものの、大きく懸念は増えなさそう
- 具体的な実装内容の見通しは半分くらいは立っている。細かい部分はやりながら考える

## 8

- 多くの労力をかければ完了できる。ハード
- コード修正時の懸念がある。懸念の内容によってはもっと上ぶれる可能性もある
- 具体的な実装内容の見通しはあまり立っていない。大部分をやりながら考える

# 参考

https://asana.com/ja/resources/story-points

https://qiita.com/keitakn/items/c30b7071ebfbf4b1b3e0