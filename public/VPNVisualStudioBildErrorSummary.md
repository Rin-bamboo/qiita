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
昨年11月にリリースされた、VisualStudio2026を使用してWebアプリケーションの個人開発を行っており、
久しぶりにソリューションを触って構築を行っていたところ、下記のエラーが急に発生しました

```
ソース https://api.nuget.org/v3/index.json のサービス インデックスを読み込めません。
  この要求の送信中にエラーが発生しました。
  接続が切断されました: 送信時に、予期しないエラーが発生しました。。
  リモート パーティがトランスポート ストリームを終了したため、認証に失敗しました。
```

![errorImage.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1298767/62745013-d9cb-4e8d-96a8-e3c031f6e0cc.png)

エラー的には、通信がうまくいっていない様子

### 発生環境
仕様IDE:Microsoft Visual Studio Community 18.2.0 version 4.8.09221

## 発生原因の推測
考えられる原因として、
1. VisualStudio側(nuget)に問題がある
2. リクエスト先URLのサーバーが死んでいる
3. ファイヤーウォールやプロキシの設定で通信ができない
4. リクエスト先のサーバーで通信がブロックされている

### 原因の切り分け
1. ブラウザで直接URLをたたいてみる(nugetに問題があるかどうか)
   ブラウザのURLでエラーのURL[https://api.nuget.org/v3/index.json]を叩いてみると。。。
   
   ![WebPageErrorScreen.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1298767/4cff17af-5fd8-4503-9181-3b6c7f8bfb29.png)
   ブラウザでもアクセスができない状態。
   つまり、VisualStudio(nuget)ではなリクエストで問題が発生している

2. リクエスト先URLのサーバーが死んでいる
   ブラウザURLでのアクセスだと、サーバー側の問題なのかわからないので通信できているかを確認
   多分、トレースとかの方がほんとは良い・・・
   ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1298767/4a05998b-90b5-424a-8154-a9b18c9ea344.png)
   
   サーバーからの応答はあるためサーバーが死んでいることはなさそう・・・
   と思いましたが、IPアドレスが「172.30.0.1」とプライベートIPアドレスになっており何かがおかしい可能性がすでにあります。

3. ファイヤーフォールやプロキシの設定で通信ができない
   ファイやウォールでアクセス拒否されていないことは確認済み（セキュリティ上どこまで情報の載せて大丈夫か分からなかったので画像は不記載・・・）

4. リクエスト先のサーバーで通信がブロックされている
   ここは、どう確認すべきか調べても情報が少なかったので・・・最近課金し始めたGPT先生に相談・・・
   #### 相談結果
   1. 443 が到達できるかを確認（ブロック判定）
      ```PowerShell:PowerShell
        Test-NetConnection api.nuget.org -Port 443
      ```
      実行結果
      ```
      ComputerName     : api.nuget.org
      RemoteAddress    : 172.30.0.1
      RemotePort       : 443
      InterfaceAlias   : Kaspersky VPN WG Nt Adapter
      SourceAddress    : ***.***.***.***
      TcpTestSucceeded : True
      ```

      TcpTestSucceeded が Trueとなっており、指定ポートは空いており疎通はできている状態ですが、
      ``` RemoteAddress:172.30.0.1 ```が何やら怪しそう・・・　
      ``` 2.リクエスト先URLのサーバーが死んでいる ```で確認した時にも同じIPになっていました
      実際、「172.xxx.xxx.xxx」のようなIPアドレスは社内IPなどのプライベートアドレスで使用されます。
      ここでもう一つ気になる情報として、
      ``` InterfaceAliasがKaspersky VPN WG Nt Adapter ```となっております。
      実際私はKasperskyのVPNを使用していたのであまり気にしていませんでしたが、通常ここは
      ``` InterfaceAlias   : イーサネット ``` のような表記になります。

      最近VPNを導入し始めたので、もしかしたらVPN側でDNSのIPアドレスが何か差し替えられている可能性がります。
      というわけでいったんVPNを切って確認してみることに

## 原因の特定
とりあえず、KasperskyVPNを解除してみました
![KasperskyVPNScreen.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1298767/52e3f41a-399c-4da9-aba1-3afaa026b186.png)


再度VisualStudioでクリーン⇒リビルドを行うと・・・
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1298767/36630c05-6206-4503-9b7a-ec6ae2fb8d04.png)

正常にビルドが成功しました！


## まとめ
最近変わったことといえば、ルーターを変えた・VPNを導入しただったので、そのあたりが怪しいことが明確ではありましたが
まさか、VPN側で悪さしているとは思いませんでした・・・

そもそも、どうしてIPアドレスが書き換わっているのかも謎のままですが…
いったんビルドする際にはVPNを切ってビルドする必要がありそうです。

