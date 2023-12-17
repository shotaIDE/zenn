---
title: "個人アプリ「うちのコメロディー」をサービスデザインする際に考えたことすべて"
emoji: "🐈"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["個人開発", "uiuxdesign", "materialdesign"]
published: false
---

先日、「うちのコメロディー」というアプリをリリースしました。

https://apps.apple.com/us/app/%E3%81%86%E3%81%A1%E3%81%AE%E3%82%B3%E3%83%A1%E3%83%AD%E3%83%87%E3%82%A3%E3%83%BC/id6450181110

https://play.google.com/store/apps/details?id=ide.shota.colomney.MyPetMelody

まだ必要最小限の機能しかない価値検証のアプリ風のものですが、ファーストリリースまでにやったことと考えたことなどをまとめてみます。

# 着想

私は趣味で作曲をしています。

そのことを知っていた知り合いの猫好きな方から、「ウチで飼ってる猫の鳴き声使って、オリジナルの音楽作れないか？」と話を持ちかけられました。

知り合いは元々、以下のような猫の鳴き声と音楽を組み合わせた作品を YouTube などで見たことがあったらしく、そこから着想を得たようです。

https://youtu.be/kG2JvYSqGR4

そうして、知り合いの猫の動画を共有してもらい、それを元にしたオリジナル音楽を何点か作りました。

https://youtu.be/wjyNe1lGZfU

そうすると、知り合いはすごく喜んでくれました。
私の予想を遥かに上回り喜んでいる感覚を受けたので、喜んでいる理由をそれとなく聞いてみました。

そうすると、自分のコだけのオリジナル作品というところに特に感動を覚えるということでした。

この際に、自分のペットだけのオリジナル音楽を作れるサービスは、面白いんじゃないかと着想を得ました。

# サービス構築に着手するまで

既存のサービスで同様のものがないかをまず調査しました。
1 つだけほぼ想定しているサービスと同じようなものを見つけましたが、すでにサービス終了していました。

https://www.itmedia.co.jp/dc/articles/1311/26/news148.html

価値のありそうな体験をミニマムに提供するサービスを、ゼロから設計してリリースしたいというモチベーションがあったので、この時点で作ってみようと決意しました。
そのため、本当に需要がありそうかの検証とかはしていません。

# 最重要のユーザー体験

サービスを設計するにあたり、ユーザーにとって価値ある体験を以下のように仮定しました。

```
自分が飼っている猫の動画を用意することで、オリジナル曲が作れる。
```

これは先述の知り合いとの会話から得た体験を元に設定しました。

要件定義や UI/UX 設計の際には、ユーザーが「最重要の体験」を得るまでに離脱しないことを最優先の目標として組み立てていきました。

# ペルソナ設定

最重要のユーザー体験に加えて、ターゲットとするユーザーのペルソナを以下に設定しました。

```
「猫」を飼って愛でていて、かつ、家族の一員として大切に育てている。
```

こちらも先述の知り合いとの会話や、知り合いの考え方を元に設定しました。

用件定義や UI/UX 設計の際には、ペルソナにとって価値があるかどうかを意識して組み立てていきました。

# 要件定義

スマホで猫の動画を撮影した後にオリジナル曲を作る、ということを繰り返しできるようにするため、モバイルアプリとして提供することにしました。

また、我が子だけのオリジナル作品を自慢したいという欲求が考えられるので、SNS へのシェア機能は必須と考えました。

一方で、作品を作るためにあらためて猫の動画を撮影する必要がある場合、猫にストレスを与えてしまう可能性があるので、すでに撮影している動画を使ってできるように全体の流れや UX を設計しました。

また、最重要なユーザー体験を得るために必要なことのみに最小限絞り、その他はバックログに積んでファーストリリースからは外すようにしました。
作っているうちに色々追加機能のアイデアは湧いてくるのですが、そうしたものに手をつけているとファーストリリースがいつまで経ってもできないです。
まずは最重要なユーザー体験を得るために必要最小限なものが揃った状態でリリースすることを目標に進めました。

# デザインシステムの設計

モバイルアプリの技術として Flutter を使う前提だったので、Flutter で iOS/Android 両方ともに適用しやすい Material Design を踏襲することにしました。

https://m2.material.io/

動物の温かみを感じられるよう、コンセプトカラーを茶色に設定しました。

![](/images/my-pet-melody-service-design/color-palette.png)

オリジナル曲を作るフローの画面のタイトルバーは透明とし、動画選択などのアクションボタンが相対的に目立つようにしました。

![](/images/my-pet-melody-service-design/title-bar-of-design-system.png)

これらの画面では、迷わず先に進めることができるかという観点では、タイトルバーに表示されているテキストや戻るボタンは、動画選択などのアクションボタンよりも重要度が低いです。
そのため、ユーザーが迷う可能性を下げることで、最重要なユーザー体験に辿り着く可能性をあげることに寄与できると考えたため、上記のデザインルールを導入しました。

# イラスト

ペルソナを踏まえて可愛い雰囲気にしたいと考えたので、猫のイラストをデザイナーに依頼しました。
イラストは描けないので、他の人の力を借りました。

あとは、ただの思いつきですが猫がアニメーションして動くと可愛いなと思い、パラパラ漫画風にアニメーションできるように体勢や角度違いのイラストを複数枚描いてもらいました。

![](/images/my-pet-melody-service-design/flicking-image.gif)

# ユーザーテスト

プロトタイプが完成した直後にユーザーテストを実施し、ユーザーが操作している様子を横で観察することで、機能や UI/UX の課題を洗い出しました。

プロトタイプ完成後、ユーザーテスト実施、というサイクルを 3 回程度繰り返しました。

オリジナル作品を作るにあたり、猫の動画の中から鳴き声が入っている部分をトリミングする必要があるのですが、プロトタイプ時点では、アプリの外で動画の鳴き声をトリミングする想定でした。
ただし、ユーザーテストを行うと、その方法はかなり複雑な操作になることが分かり、最重要の体験前に脱落する原因となる可能性が高いと感じました。
そのため、アプリ内でトリミングする機能をつけました。

また、アプリ内でトリミングする際にも、ユーザーが自分の手で鳴き声部分を確認してトリミングするのは、かなり難易度が高いと感じました。
そのため、鳴き声を検知して自動でトリミングする機能をつけました。

:::message
鳴き声を検知して自動でトリミングする機能は技術的には難易度が高く、工数もかかるものでした。
ただ、ユーザーが自分の手でトリミングするパターンと自動でトリミングされるパターンにおける、UI/UX の検討難易度の高さと最重要の体験へ到達できそうかという観点を天秤にかけ、実施することにしました。
脳死でポチポチタップするだけで最重要の体験に辿り着けるというのは、メリットが高いかなと思いました。
:::

さらに、プロトタイプ時点では、元々 2 つの動画を選んで 1 つの作品を作る、という機能を実装していました。
ところが、こちらも操作の流れが複雑になってしまい、最重要の体験の脱落の原因となりうると感じました。
そのため、動画は 1 つのみを選択するようにして、全体の流れをシンプルにしました。

# サービス名

前述している通り、「うちのコメロディー」、英語名では「My Pet Melody」という名前に決定しました。

最重要のユーザー体験における、我が子のためだけのオリジナル音楽を作れるということが名前から簡単に察することができることを意図しています。

また、その他にも以下を考慮しています。

- 一瞬でぱっと読めるか
- 類似サービス名がないか
- すでに商標登録されていないか
- 英語名のドメインがとられていないか

# キャッシュポイント

アプリでは、無料プランと有料プランを用意し、それぞれで機能差をつけています。

キャッシュポイントを作ったのは、キャッシュポイントをつけることで要件定義や UI/UX 設計の複雑さが上がるため、その部分を勉強したいと考えたからです。
サービスとしての事情ではなく、個人のモチベーションが理由です。

無料プランでアハ・モーメントを体験して、もっとたくさん使いたいと感じてもらった場合に有料プランへ誘導できるように、機能を制限しました。

価格設定については、特に理由はなくテキトーです。

# 法務的視点

以下のサイトで、利用規約とプライバシーポリシーを作成し、そこからサービス特有の事情に合わせて少しカスタマイズしました。

https://www.cloudlegal.jp/contracts

また、オリジナル作品の BGM に使う予定の音楽に関して、著作権の状態を確認し、パブリックドメインとなっているものだけを利用するようにしました。

# マーケティング

作れる作品のイメージが湧きやすいような動画を作り、宣伝やストアの素材として利用しました。

https://youtu.be/uTSQPfuDNyo

# 展望

以上が、個人アプリ「うちのコメロディー」をリリースするまでにしたことすべてです。

現時点ではユーザー数はそこまでいなくて無風状態という感じですが、今後、以下あたりをぼちぼち試していければなと考えています。

- 宣伝時の謳い文句を変える
- できることの想像がつきやすいように宣伝方法を工夫する
- ターゲットのペルソナを変える（猫を飼っている人から、他の動物を飼っている人、育児中の家庭など）
- そもそも需要がなかったと分かったら、サービスを終了する