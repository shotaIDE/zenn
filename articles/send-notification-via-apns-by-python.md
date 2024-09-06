---
title: "Pythonを使ってAPNs経由でiOS端末にプッシュ通知を送信する"
emoji: "📳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ios", "python"]
publication_name: "sun_asterisk"
published: false
---

<!-- cSpell:ignore asyncio, httpx -->

Python を使って APNs 経由で iOS 端末にプッシュ通知を送信する方法を調べていたのですが、意外と情報が少なかったので、試した方法をまとめてみました。

ポイントは、APNs への通信は **HTTP/2 のプロトコルを利用する必要がある**ことです。
そのため、HTTP/2 に対応した HTTP クライアントライブラリが必要になります。

# 前提

以下のように、何かしらの iOS アプリでプッシュ通知を受信する準備ができていることを前提として記事を進めます。

- Xcode 上でプッシュ通知受信のための Capability 追加が済んでいる
- 特定の iOS 端末向けのデバイストークンを 16 進数表現した値（以下のような値）が取得できている
  - 例: `3dcb4cb72e9abbdaa73a89d8fff449e0de4093cc2c0cbfa5ee9bed21c0a88a72`

また、保有している Apple Developer アカウントにて APNs 認証キー(`*.p8`ファイル)が取得できていることを前提とします。

:::message
上記の前提のやり方に関しては、以下を参照してください。

https://developer.apple.com/jp/help/account/configure-app-capabilities/communicate-with-apns-using-authentication-tokens/
:::

本記事では APNs サーバーとやりとりする際に Python を使用します。
動作確認した Python のバージョンは 3.12.5 です。

# 1. PyJWT をインストールする

APNs にリクエストを送信するには、認証キーを使って JWT を生成する必要があります。

今回は PyJWT を使用します。

https://pyjwt.readthedocs.io/en/stable/

以下のコマンドでインストールします。

```bash
pip install PyJWT
```

# 2. httpx をインストールする

APNs にリクエストを送信するためには、HTTP/2 に対応している HTTP クライアントが必要です。

そのため、今回は httpx を使用します。

https://www.python-httpx.org/

以下のコマンドでインストールします。

```bash
pip install httpx
```

# 3. JWT を生成するスクリプトを書く

APNs にリクエストを送信するためには、JWT トークンを生成する必要があります。
関連するドキュメントは以下です。

https://developer.apple.com/documentation/usernotifications/establishing-a-token-based-connection-to-apns

以下のスクリプトで JWT トークンを生成します。

```python
from time import time

import jwt


PRIVATE_KEY_PATH = 'apns-auth-key.p8' # APNs の認証キーのパス
APNS_KEY_ID = 'XXXXXXXXXX' # 作成した認証キーの Key ID
TEAM_ID = 'XXXXXXXXXX' # Apple Developer Program の Team ID

private_key = open(PRIVATE_KEY_PATH, 'r').read()
headers = {
    'alg': 'ES256',
    'kid': APNS_KEY_ID
}
payload = {
    'iss': TEAM_ID,
    'iat': time()
}

token = jwt.encode(
    payload,
    private_key,
    algorithm='ES256',
    headers=headers
)
```

`token` として得られた値を、リクエストヘッダーの `authorization` に `bearer {token}` の形式で設定します。

# 4. プッシュ通知のリクエストを送信するスクリプトを書く

APNs に対してプッシュ通知のリクエストを送信します。
関連するドキュメントは以下です。

https://developer.apple.com/documentation/usernotifications/sending-notification-requests-to-apns

以下のスクリプトでプッシュ通知のリクエストを送信します。

```python:send-push-notification.py
import httpx


BUNDLE_ID = 'com.example.PushTest' # iOS アプリの Bundle ID
URL_PREFIX = 'https://api.sandbox.push.apple.com/3/device/'

device_tokens = [
    # 1 台目の iOS デバイストークンの例
    '3dcb4cb72e9abbdaa73a89d8fff449e0de4093cc2c0cbfa5ee9bed21c0a88a72',
    # 2 台目の iOS デバイストークンの例
    '6f80ab76a6237ba9498a469e6f23c580d9572c0303706aa2c01a90dca3e795f6'
]

headers = {
    'authorization': f'bearer {token}', # `token` は前述の JWT トークン
    'apns-push-type': 'alert',
    'apns-topic': f'{BUNDLE_ID}'
}

payload = {
    'aps': {
        'alert': {
            'title': 'テストタイトル',
            'body': 'テスト本文'
        }
    }
}

async with httpx.AsyncClient(http2=True) as client:
    for index, device_token in enumerate(device_tokens):
        url = f'{URL_PREFIX}{device_token}'

        response = await client.post(
            url,
            headers=headers,
            json=payload
        )

        print(f'#{index} - device token: {device_token}')
        print(f'#{index} - response status code: {response.status_code}')
```

APNs サーバーは、開発用と本番用の 2 種類があり、それぞれ以下のドメインでアクセスできます。
本記事では開発用のサーバーを使用します。

- 開発用: `api.sandbox.push.apple.com`
- 本番用: `api.push.apple.com`

上記は、iOS アプリでエンタイトルメントファイル中の "APS Environment" に設定している値に応じて使い分ける必要があります。

| APS Environment | APNs サーバー |
| --------------- | ------------- |
| development     | 開発用        |
| production      | 本番用        |

以下は、APS Environment の設定画面の例です。

![](/images/send-notification-via-apns-by-python/aps-environment.png)

APS Entitlements の詳細については以下のドキュメントを参照してください。

https://developer.apple.com/documentation/bundleresources/entitlements/aps-environment

また、httpx で HTTP/2 の通信をするため、`AsyncClient` に `http2=True` を指定して通信クライアントを初期化しています。

```python
httpx.AsyncClient(http2=True)
```

さらに、すべてのリクエストが完了した際に HTTP 通信が適切に閉じられるようにするため、`async with` を使用しています。
非同期処理を含むため、`with` の代わりに `async with` を使用しています。

```python
async with httpx.AsyncClient(http2=True) as client:
    # ...
```

httpx で HTTP/2 の通信をする方法に関しては、以下のドキュメントが参考になります。

https://www.python-httpx.org/http2/

# 5. スクリプトを合わせて実行する

前述のスクリプトを組み合わせると、以下のようになります。

```python:main.py
# coding: utf-8

import asyncio
from time import time

import httpx
import jwt


def __generate_jwt_token():
    PRIVATE_KEY_PATH = 'apns-auth-key.p8' # APNs の認証キーのパス
    APNS_KEY_ID = 'XXXXXXXXXX' # 作成した認証キーの Key ID
    TEAM_ID = 'XXXXXXXXXX' # Apple Developer Program の Team ID

    private_key = open(PRIVATE_KEY_PATH, 'r').read()
    headers = {
        'alg': 'ES256',
        'kid': APNS_KEY_ID
    }
    payload = {
        'iss': TEAM_ID,
        'iat': time()
    }

    return jwt.encode(
        payload,
        private_key,
        algorithm='ES256',
        headers=headers
    )


async def send_notification():
    BUNDLE_ID = 'com.example.PushTest' # iOS アプリの Bundle ID
    URL_PREFIX = 'https://api.sandbox.push.apple.com/3/device/'

    device_tokens = [
        # 1 台目の iOS デバイストークンの例
        '3dcb4cb72e9abbdaa73a89d8fff449e0de4093cc2c0cbfa5ee9bed21c0a88a72',
        # 2 台目の iOS デバイストークンの例
        '6f80ab76a6237ba9498a469e6f23c580d9572c0303706aa2c01a90dca3e795f6'
    ]

    token = __generate_jwt_token()

    headers = {
        'authorization': f'bearer {token}',
        'apns-push-type': 'alert',
        'apns-topic': f'{BUNDLE_ID}'
    }

    payload = {
        'aps': {
            'alert': {
                'title': 'テストタイトル',
                'body': 'テスト本文'
            }
        }
    }

    async with httpx.AsyncClient(http2=True) as client:
        for index, device_token in enumerate(device_tokens):
            url = f'{URL_PREFIX}{device_token}'

            response = await client.post(
                url,
                headers=headers,
                json=payload
            )

            print(f'#{index} - device token: {device_token}')
            print(f'#{index} - response status code: {response.status_code}')


if __name__ == '__main__':
    asyncio.run(
        send_notification()
    )
```

上記を実行すると、以下のように各デバイストークンに対してプッシュ通知のリクエスト送信の成功が確認できます。

```log
#0 - device token: 3dcb4cb72e9abbdaa73a89d8fff449e0de4093cc2c0cbfa5ee9bed21c0a88a72
#0 - response status code: 200
#1 - device token: 6f80ab76a6237ba9498a469e6f23c580d9572c0303706aa2c01a90dca3e795f6
#1 - response status code: 200
```
