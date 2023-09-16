---
title: "Fiddlerを利用してAPIのレスポンス中におけるJSONの一部のバリューを書き換える"
emoji: "💂‍♂️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["fiddler", "api"]
published: true
---

クライアント開発において、サーバーからのレスポンスを細かく書き換えて動作確認したいことがあります。

本記事では、Fiddler を利用し、レスポンス中の JSON における特定のキーに対するバリューを書き換える方法をメモしておきます。

# 対象の読者

Fiddler をある程度利用したことがある方を対象としています。

https://www.telerik.com/fiddler

また、Fiddler を利用した **HTTPS 通信のリクエスト・レスポンスを復号して閲覧する方法を知っている**前提で手順を記載しています。

https://blog.14nigo.net/2022/11/fiddlerclassicpcandsmartphone.html

# 具体的な手順

Fiddler にはプロキシーとしての動作をカスタムできるスクリプトが用意されているので、そのスクリプトにレスポンスを書き換える処理を追記することで実現します。

https://docs.telerik.com/fiddler/extend-fiddler/addrules

## 解説する書き換え内容

API `https://your.domain/v1/environment` のレスポンスにおける特定のキー `planCode` に対応するバリューを、`2` に書き換えたい、という状況を想定します。

```json:Before
{
  "errorCode": 0,
  "result": {
    "featureAEnabled": true,
    "planCode": 1
  }
}
```

```diff json:Before
{
  "errorCode": 0,
  "result": {
    "featureAEnabled": true,
-     "planCode": 1
+     "planCode": 2
  }
}
```

## スクリプトを修正する

Fiddler を起動し、メニューの "Rules" > "Customize Rules..." をクリックします。

![](/images/replace-response-with-fiddler/01_fiddler-customize-rules-menu.png)

以下の通りに書き換え、上書き保存します。上書き保存すると、すぐに書き換え機能が動作します。

```diff javascript
// ...

static function OnBeforeResponse(oSession: Session) {
  if (m_Hide304s && oSession.responseCode == 304) {
      oSession["ui-hide"] = "true";
  }

+  if (oSession.host == "your.domain" && oSession.PathAndQuery == "/v1/environment") {
+    var responseBodyString = oSession.GetResponseBodyAsString();
+    var responseBodyJson = Fiddler.WebFormats.JSON.JsonDecode(responseBodyString);
+
+    responseBodyJson.JSONObject["result"]["planCode"] = 2;
+
+    var fixedResponseBodyString = Fiddler.WebFormats.JSON.JsonEncode(responseBodyJson.JSONObject);
+    oSession.utilSetResponseBody(fixedResponseBodyString);
+  }
}

// ...
```

![](/images/replace-response-with-fiddler/02_fiddler-script-editor.png)

## スクリプトをデバッグする

もし、スクリプト書き換え後にうまく動作しない場合は、"Log"タブを確認すると解決のヒントが得られます。

![](/images/replace-response-with-fiddler/03_fiddler-log.png)
