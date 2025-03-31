---
title: '#02 C# WinFormsによるメール受信検索アプリ'
tags:
  - C#
  - WinForms
  - mailkit
private: false
updated_at: '2025-02-01T14:01:43+09:00'
id: 1f0af7ddb20eba1d62f4
organization_url_name: null
slide: false
ignorePublish: false
---
# 受信したメールの一部が文字化けして検索がうまくいかない

前回記事
[#01 C# WinFormsによるメール受信検索アプリ](https://qiita.com/rin-bamboo/items/f068fedc1f763b1d37c8 "#01 C# WinFormsによるメール受信検索アプリ")

:::note warn
警告
MailKit自体慣れてないため、間違えている可能性があります。
確認次第都度修正していきます。
参考にはできますが、このままの実装はできませんのでご了承ください
:::


## 文字化けの解消

### 解決手段を教えてもらい…
前回から時間が経ってしまいましたが、無事文字化けすることなくメールを受信することができました。
前回記事に[解決策のコメント](https://qiita.com/rin-bamboo/items/f068fedc1f763b1d37c8#comment-968b3db0e362edc081e0)をいただき確認したところ、非同期(Async)のメソッドだとなぜかうまくいきそう

もともとの取得処理（抜粋して記載）
* GetCharsetAndEncodingのメソッドで受信メールの文字コードを取得
* AllConvertToUtf8のメソッドで取得した文字コードをUTF-8に変換
の流れでしたが、この段階で文字化けしたりして困っていました。

``` C#
//非同期ではない方法で取得
    //メールサーバー接続
    this._client.Connect(this.mailhost, this.portNo, this.ssl);
    this._client.Authenticate(this.mail_addr, this.passwd);

    var folder = this._client.GetFolder(this._client.PersonalNamespaces[0]);
    
    //IMAPフォルダを開く
    folder.Open(FolderAccess.ReadOnly);

    // 指定した日付範囲内のメールを検索
    var query = SearchQuery.DeliveredAfter(FromDate).And(SearchQuery.DeliveredBefore(ToDate));
    var uids = folder.Search(query);

    //メールの文字コードを取得
    (string? charset, bool isBase64) = GetCharsetAndEncoding(m);
    if (charset == null || charset.Length == 0) { continue; }

    string EncordingSubject = m.Subject;
    string EncordingBody;
    if (m.HtmlBody is not null) 
    {
        EncordingBody = m.HtmlBody;
    }
    else
    {
        EncordingBody = m.TextBody;
    }

    //文字コード変換
    EncordingSubject = utf8Converter.AllConvertToUtf8(EncordingSubject, charset, isBase64); //件名
    EncordingBody = utf8Converter.AllConvertToUtf8(EncordingBody, charset, isBase64);       //本文
    _from = ((MailboxAddress)m.From[0]).Address;
    _fromName = utf8Converter.AddressEncUtf8((MailboxAddress)m.From[0]);

    var _toList = new List<string>();
    foreach (MailboxAddress addr in m.To.Cast<MailboxAddress>())
    {
        _toList.Add(addr.Address);
    }
    _to = string.Join(";", _toList);

    var _ccList = new List<string>();
    foreach (MailboxAddress c in m.Cc.Cast<MailboxAddress>())
    {
        _ccList.Add(c.Address);
    }
    _cc = string.Join(";", _ccList);
    
```

教えていただいたものを改良（抜粋して記載）
* 特別な変換などは行わずに取得できています
``` C#
//非同期(Async)のメソッドで取得

//メールサーバー接続
await imap4.ConnectAsync(this.mailhost, this.portNo, this.ssl); 

await imap4.AuthenticateAsync(this.mail_addr, this.passwd);
await imap4.Inbox.OpenAsync(FolderAccess.ReadOnly);

var folders = await imap4.GetFoldersAsync(imap4.PersonalNamespaces[0]);

var message = await folder.GetMessageAsync(uid);
if (string.IsNullOrEmpty(message.Subject))
{
    message.Subject = "(件名なし)";
}

string messageText = message.TextBody ?? message.HtmlBody ?? string.Empty;
string attachment = string.Join(";", message.Attachments.Select(a => a.ContentType.Name));

_from = ((MailboxAddress)message.From[0]).Address;
_fromName = message.From[0].Name;

foreach (MailboxAddress addr in message.To.Cast<MailboxAddress>())
{
    _toList.Add(addr.Address);
}
_to = string.Join(";", _toList);

foreach (MailboxAddress c in message.Cc.Cast<MailboxAddress>())
{
    _ccList.Add(c.Address);
}
_cc = string.Join(";", _ccList);

```

もともとの処理をデバッグしてみると、やはり取得時に文字化けしている状態で取得していました。
初期のころは、この時メールに設定されている文字コードを取得して変換するように頑張りました。

もともと、MailKitはUTF-8で受信する前提？になっているようなので、SJISやJISその他もろもろで受信している場合、UTF-8の変換時に文字化けしているのかと思っていました。

調べてみると、MailKitと一緒に使うライブラリ「MimeKit」は自動で文字コードを判断して適切な変換をしてくれるとかなんとか・・・
とはいえ、MimeKitも使って文字化けしているのでライブラリ自体のバグか仕様か・・・

ドキュメントが明らかに少ないので、謎は深まるばかり・・・

教えていただいた、非同期(Async)処理のメソッドで取得するとデバッグ段階ではUTF-8のバイナリデータで取得している状態になりました。もともとの処理はデバッグの段階で通常の文字で取得していたためここで違いが出ている状態です。

とはいえ、メソッド自体が非同期処理なので待つことなく処理が進みます。
このあたりは処理の流れの考慮が必要そうです。
（そもそも、非同期処理に慣れていないので、呼び出しメソッドをすべて非同期にしてそこで処理を行うことにしました。）

文字化けすることなく無事取得することができました。

## メール重複取得
ここでも少し問題が発生。
同じメールが重複して取得されてしまいました。

同じメールを処理しないように修正しました。
``` C#
    var uids = await folder.SearchAsync(query); //フォルダ内を検索条件で検索

    foreach (var uid in uids)
    {
        if (uidHistory.Contains(uid.ToString()))
        {
            // すでに処理済みのUIDの場合はスキップ
            continue;
        }
        uidHistory.Add(uid.ToString()); 

        //取得処理省略
    }
```
uids繰り返しの中でuidを保管しすでに持っている場合はcontinueさせます。


## メール検索範囲指定の問題
他にも発生したのは、日付範囲で検索する際に未来日付を指定している場合うまく検索がかからないということ。

クエリ生成時に、未来の日付がToに入力されている場合はFrom以降すべて取得するようにしました。

``` C#

    //検索条件　Toが未来日付の場合は、Fromのみ指定
    SearchQuery query = new SearchQuery();
    if (ToDate > DateTime.Now)
    {
        query = SearchQuery.DeliveredAfter(FromDate);
    }
    else
    {
        query = SearchQuery.DeliveredAfter(FromDate).And(SearchQuery.DeliveredBefore(ToDate));
    }
    
```

これで大まかにバグなくメールを取得することができます。

非同期にしても処理の流れはほぼ変わりません。
メールサーバー接続　⇒　メールフォルダ取得　⇒　メールフォルダ内メール取得（範囲指定）　⇒　メール情報取得

希望する方がいましたら、より詳しく実装の内容を記載していきます。
（その場合会社で使っているツールの為、別で構築します。）

## 無事文字化けせずに取得
前回から期間が開いてしまいましたが、無事に文字化けせず取得することができました。
ただし、既読フラグなどは一切考慮していないため今後必要になった場合は調べて記事の続き書きます


