---
title: "Flutter で iOS の Automatic Signing を使う"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

年次のプロビジョニングプロファイルの更新作業から解放されたい。

Flutter の iOS モジュールだけをビルドし、アーカイブとエクスポートはコマンドで実施する。

アーカイブの時にも Automatic Signing の設定を完全にしておかないと、色々と不具合が起きる。

プロビジョニングプロファイルがキャッシュされていないとアーカイブに失敗する。CD 環境上で失敗する。
Apple でサインインが失敗する。
