---
title: "E2Eテストを導入する"
emoji: "🕌"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["ios", "android"]
published: false
---

## 3-1. E2E テストの導入開始

- 開発メンバーによる自律的な E2E テストの追加とメンテナンス、精度向上の取り組み。開発メンバーが外部品質にオーナーシップをより持っていくことに繋げられそう

### E2E テストについて

#### E2E テストを運用することで、デグレを検知できる

プロダクトコード側で、選択肢ダイアログ表示中に別のダイアログが上に表示されてようになってしまい、選択肢ダイアログの操作がしづらくなりました。
つまり、操作性が悪くなるデグレでした。
E2E テストが失敗するようになったので、この事象を検知し修正できた。

#### E2E テストを実装することで間接的にプロダクトコードの品質向上に寄与している

テストの実行結果が不安定となる場合があり、それらはプロダクトコード側の品質が悪いことに原因があります。

- ネットワーク速度が遅いあるいは端末のスペックが低い場合に、正しい順番で処理が行われない
- プログラム中の準正常系、異常系のパターンが漏れている

#### 開発が立ち上がるまでのコストがそれなりに大きい

- CI 環境で、ネットワーク速度が遅かったり端末の動作が遅かったりする場合にもロバストに動くテストコードの書き方
- CI 環境で E2E テストを動かすための環境構築

#### メンテナンスのための開発者の作業コストが増える

- テストが失敗した際の調査
- 日々の機能追加や修正による、E2E テストコードの修正

#### 自動実行のための CI 環境の経済的コストがかかる

- マシンにある程度のスペックが求められるため、静的解析や単体テストを回すスペックよりも良いマシンスペックを選択したり、専用の SaaS を利用した方がいい
