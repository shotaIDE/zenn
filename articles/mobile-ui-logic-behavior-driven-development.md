---
title: "振る舞い駆動開発みたいなやつやってみよう"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ios", "android"]
published: false
---

## ふるまい駆動開発とは

## 調査が不足していること

- そもそも寄与するのかをもっと深ぼる？
  - バグが起きているファイルで頻出のものを確認する

## 背景

- エンジニアが仕様を確認していない
- リファクタリングが追いついていない

## メリット

- エンジニアが仕様の理解をする機会が増える
- 適切な設計につながる
- 最初からリファクタリングするハードルが減る
- iOS/Android、あるいは Web や API 側も相互でレビューできる可能性がある

## 導入にあたって

サンプルを実装し、できそうかを確認する。

## 設定した方針

- UI はテストしない
  - この内容が ViewModel・Presenter から出力されれば、UI は正常に表示されるだろう、という前提に立ち、ViewModel、Presenter の出力を確認する
- API とは通信しない。
  - スモールテストとする
- 画面仕様書をインプットとする
- 画面仕様書に書かれているすべての分岐をカバーする
  - ユーザーがやりたいことを達成するために必要なこと
- 書かれていない分岐のテストは書かない

### 概要

コード実装時、以下のように進めるようにする。

- 画面仕様書における条件に応じた表示やアクションに応じた UI の挙動に対し、画面ロジック部分のみをテストコードとして定義したのち、プロダクトコードを実装する。

これにより、品質向上を狙う。

### 目的

以下を目的として行う。

- 開発者による画面仕様書の理解度や、セルフテストの品質における属人性を排除する
- セルフテストの再実行を容易にし、リファクタリングの心理的ハードルを下げる
- View や UI ロジック周りでテストコードを実装することで、リファクタリングすべき箇所の発見機会を増やす
- 包括的な自動テストが実装されることに繋げ、開発者・QA が品質に対して根拠ある自信を持つ

### やり方

以下の前提のもと、テストコードを実装する。

#### テスト対象

実装する際に参照する画面仕様書における、条件に応じた表示やアクションに応じた UI の挙動。

これらの条件**全てに対し、条件 1 つに対して 1 つのテストコードが対応する**ように書く。

#### 実装方法

- UI ロジックに関する INPUT と OUT を確認する
- UI そのものが動くかどうかはテストしない
  - → 安定性や速度を高めるため。
- API や Web ソケット通信、DB、キーバリューストアはモックを使う
  - → 安定性や速度を高めるため。
- API や Web ソケット通信、DB、キーバリューストア以外はモックを使わずに本物を使う
  - → UI ロジックに近い場所をモックしすぎると、UI ロジックに紐づくアプリの動作のテストにならないため。また、仕様変更やリファクタリング時にテストコードが壊れにくくするため。

いわゆる一般的な振る舞い駆動開発とは異なり、UI そのものはテストしない。

実装するの難易度高いからやめるという結論もありうる。