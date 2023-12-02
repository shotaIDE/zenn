---
title: "多国籍チームでタッグを組み、スピード感ある継続的な価値提供を実現する"
emoji: "🕌"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["ios", "android"]
published: false
---

こんにちは。[Sun Asterisk](https://sun-asterisk.com/)でモバイルアプリエンジニアをやっている、井手 将太です。
本記事は、[Sun\* Advent Calendar 2023](https://adventar.org/calendars/9043) の n 日目の記事です。

私は、Sun Asterisk でモバイルアプリエンジニアとして iOS と Android のアプリ開発プロジェクトに参画しています。
より魅力的なアプリを素早く世の中に届けるためエンジニアの領域から何ができるか、ということをミッションとして業務に取り組んでいます。

# 想定する読者

プロジェクトマネージャーや、エンジニアを想定しています。

# 現在取り組んでいるプロジェクトとその背景

Sun\*は、クライアントさんとさまざまなプロダクトを世に届けるあらゆるフェーズで、一緒にチームを組んで伴走することを実施しています。

私が取り組んでいるプロジェクトでは、元々クライアントさんがリリースしていたサービスが、開発の技術的負債が蓄積していて、開発スピードの鈍化が顕著な問題となっていました。
コードの内部品質が悪化していて、バグを修正するのに非常に時間がかかる、修正時の影響が不明で検証に時間がかかる、などの問題が発生していました。

そうした課題を解決するために、開発チームの体制立て直しを行うことになり、その際に Sun\*がジョインして開発チームの体制立て直しを進めさせていただくことになりました。

そうした中で、開発チームが以下に開発スピードを落とさずに価値提供していけるかという点にフォーカスし、取り組んでいる内容を紹介します。

# 多国籍チームでタッグを組む際の課題

Sun\*はベトナム側の拠点に 1000 名以上のエンジニアが在籍しており、潤沢な開発力があります。

彼らは、そもそも自立して動ける開発のプロフェッショナルです。

そのため、特に開発のスピードを落とさずに取り組むということにも自立的に取り組んでくれます。

一方で、ベトナム語を母国語とするメンバーですので、日本語でコミュニケーションを取るクライアントと言語の壁があります。
ベトナムチームとはブリッジ SE がいるので一応口頭のコミュニケーションは取れるものの、直接話せるわけではないので、やはり壁が生じてしまいます。

開発スピードを落とさない、という命題は開発チーム内だけで完結する問題ではなく、状況に応じてデザイナー、ビジネスサイド、サポートチームなどを巻き込んでいく必要がある、複雑な問題です。

また、ベトナムチームと日本チームでオンラインミーティングなどで口頭のコミュニケーションを密にとることもできます。
一方で、これは技術的な内容を含む通訳を介したコミュニケーションとなるため、フロー効率が悪く、時間を多く消費してしまいます。
そのため、あまりにこの時間をとってしまうと、時間が効率的に使えていない感があったり、単純に長時間ミーティングで疲弊してしまいます。

そのため、コミュニケーションパスを制限し、取り決めによりお互い干渉せずに各々業務を進められるようにしておくという動きも必要です。

# 業務ドメインの分担

そうした動きを行っていく中で、業務ドメインの効率的な境界線が見えてきました。

現在の分担の仕方が以下となっています。

（ここに、ドメイン分担の様子を示した図を挿入します。）

## ベトナムチームのドメイン担当

- 設計
- 実装
- リグレッションテスト
- 継続的なリファクタリング
- バグ分析
- モバイルチーム内の振り返り
- 依存ライブラリの継続的改善
- 自動テスト
- 自動の E2E テストと監視
- 継続的インテグレーション
- 外部からの仕様改善要求
- ストーリーポイントの見積もり
- コードレビュー

## 日本チームのドメイン担当

- スプリントゴールの設定
- 要求仕様のすり合わせ
- スプリントレビュー
- 画面仕様の策定
- API 仕様の決定
- 各クライアントとの足並みを揃える
- コードの内部品質の監視
- 継続的デプロイ
- リリース
- 運用監視
- 開発チームの定性的評価
- チーム設計
- 複雑な仕様のドキュメンテーション
- サポート対応
- 開発プロセスの全体的な改善
- 技術トレンドに応じた、仕様の単純化

# ドメイン担当の理由など

各種ドメイン担当に関していくつか試行錯誤や理由などはありつつも、全てをここに記載するのは情報が多くなりすぎるので、この記事ではいくつかピックアップして解説していきます。

## 要求仕様のすり合わせ

要件定義は、基本的に日本側で行います。
一方で、ベトナムメンバーからフィードバックをもらった際にすぐに軌道修正できるよう、仔細に決めすぎないようしています。

ベトナムチームは日本語を用いた口頭のコミュニケーションが取れないという制約がありますので、クライアント側にいるプロダクトマネージャーと距離感が生じてしまいます。

その点がそのまま放置されていると、ビジネス側・エンジニア側の視点が極端に偏ってしまった実装になる懸念があります。例えば、プロダクトにとってはそこまで重要でない機能に過剰に工数がかけられてしまう、技術的なメリットだけが考えられてプロダクトにとって重要な機能がなくなってしまう、などです。

その差を解消するために、プロダクトの背景知識や仕様の背景などを、明文化して伝えるということに取り組んでいます。
背景知識まで伝えることで、ベトナムチームのメンバーも提示された仕様がプロダクトにとって妥当かという点まで考えることができ、より良い仕様の提案などが実施してくれます。

（開発視点での FB をデザイナーやビジネスのメンバーに伝えるという図を掲載する。）

## ストーリーポイントの見積もり

ストーリーポイントの見積もりは、ベトナムチームと日本チームが一緒にオンラインミーティングへ参加して実施しています。

元々見積もりは、仕様を日本チームが明文化し、それを見てベトナムチームがストーリーポイントを割り振るという形式を取っていました。

ただ、進行するにつれて、ベトナムチームの見積りと日本チームの想定の間でしばしば乖離が見られ、ストーリーポイントに基づく結果の違いが顕著になり始めました。
これには、タスクのゴールの認識が合っていなかった、見積もりの相対的なサイズ感によるストーリーポイントを見積もるという考え方が合っていなかった、という理由がありました。
正確な見積もりができないことに対する不安により反発がありました。
しかし、見積もりに時間を過剰にかけて正確な見積もりを出しても、プロジェクトとしての費用対効果が低いです。
また、見積もりに応じてスプリント内に完了させるというコミットメントは、デイリーの確認で調整すれば良いです。
これらのことをベトナムチームに伝え、現状の形になりました。

（仕様を伝えて、ストーリーポイントを見積もるという図を掲載する。）

## リリース作業

デプロイ作業については、ビジネスとの調整が必要になってくるので、日本語のコミュニケーションを取れるメンバーが分担することにしています。

トランクベース開発をし、トランクブランチはどのコミットでもリリース可能な状況となるよう、コミットの粒度やゴールを認識合わせしていきます。
場合によって、フィーチャートグルを導入し、本番では稼働しないコードをサイレントリリースできるようにしておきます。
それにより、ベトナムの開発メンバーは、トランクブランチにマージすること集中し、日本側はその時にトランクブランチの最新コミットをリリースすれば良いのです。
こうすることで、密なコミュニケーションが不要となります。

実際には、特定の地点ではデグレが発生しているので、追加コミットが必要など、微調整が必要ではあります。

（トランクベース開発の図を掲載する。）

## 開発チームの定性的評価

開発メンバーの定性的な自己評価や、スプリントごとの振り返りを共有してもらうことで、開発プロセスにおける改善の妥当性や問題点の洗い出しを行っています。

定性的な自己評価では、Google フォームによるアンケートで 2 ヶ月に 1 回回答してもらっています。
内容としては、開発チームのコード、各種開発プロセスや、チームに対する満足度や自由記述の意見などを回答してもらっています。

（Google フォームのアンケートの図を載せる。）

こうした背景には、[DevOps の能力](https://cloud.google.com/architecture/devops?hl=ja)や、SPACE 指標を参考にしています。
測定したいキャパシティや指標を選択し、それが測定できるように質問を構築しています。

スプリントごとの振り返りでは、「感謝したいこと」「課題」の 2 分類で意見を出す単純なフレームワークなどを利用し、ベトナムチーム内で実施してもらいます。

「感謝したいこと」を組み込んでいる意図としては、メンバーが「良い」とか「楽しい」と感じる基準を間接的に読み取ることができると考えたからです。
これにより、開発プロセスの改善やドメイン分担などにおいて、メンバーのモチベーションにつながる取り組みへ繋げることができます。

振り返りは Miro で実施しているので、意見などが書かれたベトナム語の付箋を日本語に翻訳してもらい、ブリッジ SE の方に口頭の補足踏まえて内容を共有してもらいます。

（Miro のスクリーンショットを載せる。）

その結果、ベトナムチーム内で問題に対する改善案が出た場合は、そのまま実施してもらうようにしています。

新しい技術を習得することをポジティブに捉えていました。

# まとめ

上記で記載した分担は、あくまで現時点での状況のスナップショットにすぎません。
今後も進めていく中でブラッシュアップが必要と考えています。

同じような課題やチャレンジをしている方の参考になれば幸いです。