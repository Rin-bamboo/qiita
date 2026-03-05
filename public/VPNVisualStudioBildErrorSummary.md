---
title: VPN接続中にVisualStudioのNuGet復元が失敗した話
tags:
  - 'VPN'
  - 'VisualStudio2026'
  - 'nuget'
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# VisualStudioのビルドエラーが発生した


## 発生した問題

昨年11月にリリースされた **Visual Studio 2026** を使って、個人開発でWebアプリケーションを作っています。
久しぶりにソリューションを開いてビルドしたところ、急に下記のエラーが発生しました。

```txt
ソース https://api.nuget.org/v3/index.json のサービス インデックスを読み込めません。
  この要求の送信中にエラーが発生しました。
  接続が切断されました: 送信時に、予期しないエラーが発生しました。
  リモート パーティがトランスポート ストリームを終了したため、認証に失敗しました。
```

![errorImage.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1298767/62745013-d9cb-4e8d-96a8-e3c031f6e0cc.png)

エラーメッセージを見る限り、通信がうまくいっていない様子です。

### 発生環境

**使用IDE**：Microsoft Visual Studio Community 18.2.0
**Version**：4.8.09221

---

## 発生原因の推測

考えられる原因として、以下を想定しました。

1. Visual Studio 側（NuGet）に問題がある
2. リクエスト先URLのサーバーが落ちている
3. ファイアウォールやプロキシの設定で通信できない
4. リクエスト先サーバー側で通信がブロックされている

---

## 原因の切り分け

### 1. ブラウザで直接URLを開いてみる（NuGet 側に問題があるかの確認）

まずは、ブラウザでエラーに表示されているURL
`https://api.nuget.org/v3/index.json`
へ直接アクセスしてみました。

![WebPageErrorScreen.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1298767/4cff17af-5fd8-4503-9181-3b6c7f8bfb29.png)

すると、ブラウザでもアクセスできない状態でした。

つまり、**Visual Studio（NuGet）固有の問題ではなく、リクエスト自体に問題が発生している**ことが分かります。

---

### 2. リクエスト先URLのサーバーが落ちているか確認する

ブラウザでアクセスできないだけでは、サーバー側の問題なのか判断しにくいため、通信自体ができているかも確認しました。
（本当はトレースなどを取ったほうがよいかもしれません……）

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1298767/4a05998b-90b5-424a-8154-a9b18c9ea344.png)

サーバーからの応答自体はあるため、**サーバーが完全に落ちている可能性は低そう**です。

……と思ったのですが、IPアドレスが **`172.30.0.1`** となっており、これはプライベートIPアドレスです。
この時点で、何かがおかしい可能性がありました。

---

### 3. ファイアウォールやプロキシの設定で通信できない

ファイアウォールでアクセス拒否されていないことは確認済みです。
（セキュリティ上、どこまで情報を載せてよいか分からなかったため画像は省略します）

---

### 4. リクエスト先サーバー側で通信がブロックされている

ここは、どう確認すればよいか調べても情報が少なかったため、最近課金し始めたGPT先生にも相談してみました。

#### 相談結果

まず、443番ポートへ到達できるか（ブロックされていないか）を確認しました。

```powershell
Test-NetConnection api.nuget.org -Port 443
```

実行結果：

```txt
ComputerName     : api.nuget.org
RemoteAddress    : 172.30.0.1
RemotePort       : 443
InterfaceAlias   : Kaspersky VPN WG Nt Adapter
SourceAddress    : ***.***.***.***
TcpTestSucceeded : True
```

`TcpTestSucceeded` が `True` なので、指定ポートは開いており、疎通自体はできています。

ただし、やはり `RemoteAddress : 172.30.0.1` がかなり怪しいです。
先ほどの確認でも同じIPアドレスになっていました。

実際、`172.xxx.xxx.xxx` のようなアドレスは、社内ネットワークなどで使われる**プライベートIPアドレス**です。

さらに気になったのが、以下の点です。

```txt
InterfaceAlias   : Kaspersky VPN WG Nt Adapter
```

私は Kaspersky のVPNを使用していたので最初はあまり気にしていませんでしたが、通常であればここは

```txt
InterfaceAlias   : イーサネット
```

のような表示になります。

最近VPNを導入したばかりだったので、**VPN側でDNSの名前解決先が差し替えられている可能性**を疑いました。
そこで、いったんVPNを切って確認してみることにしました。

---

## 原因の特定

とりあえず、Kaspersky VPN を無効化してみました。

![KasperskyVPNScreen.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1298767/52e3f41a-399c-4da9-aba1-3afaa026b186.png)

その状態で再度、Visual Studio で **クリーン → リビルド** を実行すると……

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1298767/36630c05-6206-4503-9b7a-ec6ae2fb8d04.png)

**正常にビルド成功！**

---

## まとめ

最近変わったことといえば、**ルーターを変えたこと**と、**VPNを導入したこと**でした。
そのあたりが怪しいとは思っていましたが、まさか **VPN側が原因** だったとは思いませんでした……。

そもそも、なぜIPアドレスが書き換わっていたのかは、まだよく分かっていません。

ひとまず現状では、**ビルド時はVPNを切る**必要がありそうです。