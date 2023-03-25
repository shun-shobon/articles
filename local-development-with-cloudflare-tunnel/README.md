---
emoji: "🚇"
tags:
  - Tips
---

# Cloudflare Tunnel でローカルの開発環境を公開する

## はじめに

LINE Messaging API や Slack/Discord の Slash Command など、Webhook ベースの API を利用する場合、
ローカルに建てた Web サーバーを外部からアクセスできるようにする必要があります。

今までは、[ngrok](https://ngrok.com/) などのツールを利用していましたが、
Cloudflare Tunnel でも同様のことができるということで、
今回はそれを試してみました。

## Cloudflare Tunnel とは

Cloudflare Tunnel は、プライベートなネットワークと Cloudflare の Edge Network を接続するためのツールです。
プライベートなネットワークと Cloudflare の Edge Network を接続することで、プライベートなネットワークを公開せずに、Cloudflare を通して外部からアクセスできるようになります。
もちろん接続は暗号化されているので、セキュリティ的にも安心です。
また Cloudflare Tunnel は Cloudflare Zero Trust の一部なので、単に Cloudflare を通すだけでなく、間に認証などのセキュリティ機能を挟むこともできます。

今回はこれのトンネル機能だけを利用します。

## 使い方

先に Cloudflare のアカウントを作成しておいてください。

### cloudflared のインストール

Cloudflare Tunnel では Cloudflare の Edge Network に接続するために cloudflared というツールを利用します。
cloudflared は各 OS のパッケージマネージャーからインストールできる他、Docker イメージも公開されています。
詳しくは公式のリポジトリを参照してください。

https://github.com/cloudflare/cloudflared

### とりあえずトンネルを作成してみる

まずはとりあえずトンネルを作成してみます。トンネルを作成するだけならログインをしなくても可能です。
以下のコマンドは `localhost:3000` を外部に公開するトンネルを作成します。

```bash
cloudflared tunnel --url localhost:3000
```

コマンドを実行すると、トンネルの URL が表示されます。URL は `https://<UUID>.cfargotunnel.com` の形式になっています。
この URL にアクセスすると、`localhost:3000` にパケットが転送されます。終了する場合は Ctrl + C で終了してください。
この UUID は一時的なものなので、次回実行すると異なるものになります。
UUID を固定したい場合は後述する手順に従って名前付きのトンネルを作成することができます。

### ログイン

cloudflared から Cloudflare にログインします。
コマンドを実行すると、URL が表示されるので、ブラウザで開いてログインしてください。
もし 1 アカウントに複数のドメイン名を登録している場合は、どのドメイン名を利用するか選択する画面が表示されます。

```bash
cloudflared login
```

### トンネルの作成

トンネルを作成します。このコマンドではあくまでトンネルの作成を行うだけで、トンネルに接続することはありません。
トンネルには名前を付けることができるので、次回以降の接続時にトンネル名を指定することで、同じ UUID を持ったトンネルに接続することができます。

```bash
cloudflared tunnel create <トンネル名>
```

コマンドを実行すると UUID が表示されます。この UUID がトンネルの URL になります。

### トンネルの接続

作成したトンネルへの接続は先程のコマンドで作成したトンネル名を指定して実行します。

```bash
cloudflared tunnel --url localhost:3000 <トンネル名|UUID>
```

### 独自のドメイン名を利用する

UUID でのトンネルの URL は長くて覚えにくいですが、独自のドメイン名を利用することもできます。
やり方は簡単で、DNS に CNAME レコードを追加するだけです。

CNAME レコードに `<UUID>.cfargotunnel.com` を追加することで、そのドメイン名でトンネルにアクセスできるようになります。

### トンネルの一覧表示と削除

トンネルの一覧は `cloudflared tunnel list` で閲覧する他、Cloudflare Zero Trust のダッシュボードからも確認できます。

トンネルの削除は `cloudflared tunnel delete <トンネル名|UUID>` で行うことができます。もちろんダッシュボード上からも可能です。

## まとめ

いままでこういった場面では ngrok を利用するのが一般的だったと思いますが、Cloudflare Tunnel でも同様のことができることがわかりました。
なんとなく Cloudflare の方が安心感があるので、今後は Cloudflare Tunnel を利用していこうと思っています。

他にも Cloudflare Tunnel では認証をかけたり SSH 接続をトンネル経由で行うこともできるようです。
そのあたりはまだ試せていませんが、興味がある方は公式のドキュメントを参照してみてください。
