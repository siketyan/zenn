---
title: "それ，本当に SSL ですか？意外と知られていない昔話"
emoji: "🔐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ssl", "tls", "network"]
published: true
publication_name: "chot"
---

:::message
この記事は以前ちょっと株式会社 社員ブログで公開していたものです
:::

こんにちは，外部技術顧問の しけちあ です．今回は， SSL (Secure Socket Layer) をご紹介します．

## SSL とは

SSL は 1995 年ごろに NetScape Communications において開発された[^1] セキュアな E2E 通信を提供するためのプロトコルです． SSL 2.0 ，3.0 とバージョンアップを重ねていきましたが， 2011 年には SSL 3.0 の仕様が IETF RFC 6101 に Historical Document としてアーカイブされ[^2]，併せて IETF RFC 6176 では多数の脆弱性を持っていた[^3] SSL 2.0 が Deprecated (非推奨) と定義されました．さらに，2014 年に POODLE という脆弱性が見つかったため[^4] 2015 年には SSL 3.0 も RFC 7568 で Deprecated と定義されました[^5]．これ以降 SSL の新しいバージョンは発表されませんでした．

[^1]: https://web.archive.org/web/19970614020952/http://home.netscape.com/newsref/std/SSL.html
[^2]: https://datatracker.ietf.org/doc/html/rfc6101
[^3]: この詳細は RFC 6176 のセクション 2 に詳しく書かれています．
[^4]: https://cve.org/CVERecord?id=CVE-2014-3566
[^5]: https://datatracker.ietf.org/doc/html/rfc7568

大事なことなのでもう一度述べます． SSL 3.0 は非推奨です．そして SSL 3 より先のバージョンはありません．言い換えればすでに SSL はすべて脆弱性が見つかったために非推奨と定義されています．そのため SSL 3.0 は現在ほとんどの Web サイトで使用されていません[^6]．では一体私たちは何を SSL と呼んでいるのでしょうか．

[^6]: https://www.f5.com/labs/articles/threat-intelligence/the-2021-tls-telemetry-report

## SSL と TLS

1999 年に IETF RFC 2246 で TLS 1.0 という規格が定義されました[^7]．これは SSL 3.0 の代替として開発されたもので，その違いについては以下のように述べられています:

[^7]: https://datatracker.ietf.org/doc/html/rfc2246


> The differences between this protocol and SSL 3.0 are not dramatic, but they are significant enough that TLS 1.0 and SSL 3.0 do not interoperate (although TLS 1.0 does incorporate a mechanism by which a TLS implementation can back down to SSL 3.0).


TLS 1.0 は革新的な規格でしたが， 2006 年に IETF RFC 4346 で TLS 1.1 が定義された[^8] のち， 2008 年には TLS 1.2[^9] ， 2018 年には TLS 1.3[^10] が定義され，機能追加や改善が盛り込まれました．ここでは TLS の各バージョンの説明はしませんが，新しいバージョンの普及とともに古いバージョンは徐々に使われなくなってきました．

[^8]: https://datatracker.ietf.org/doc/html/rfc4346
[^9]: https://datatracker.ietf.org/doc/html/rfc5246
[^10]: https://datatracker.ietf.org/doc/html/rfc8446

## 実際の普及率

現在，どのバージョンがどのくらい使われているのかは，パケットを実際にキャプチャみると明確に分かります．以下は Wireshark を利用して SSL および TLS のハンドシェイクパケットを抽出した例です[^11]:

[^11]: 説明の都合上，QUIC による TLS 1.3 ハンドシェイクは除外しています．

![Wireshark によるキャプチャ例](https://storage.googleapis.com/zenn-user-upload/8099bf837b9c-20221021.png)

## おわりに

パケットキャプチャの結果からもわかる通り，現在私たちが使っているのでは SSL ではなく TLS であり，また，ほとんどが TLS 1.2 または TLS 1.3 です． TLS 1.3 はまだ新しい規格であるため，これから QUIC[^12] とともに普及していくことが期待できます．人前で TLS のことを SSL と言ってしまうと， TLS のことを知らないと思われてしまう可能性がありますので，この機会にぜひ覚えてみてください．

[^12]: https://datatracker.ietf.org/doc/html/rfc9000

余談ですが，ここまで SSL という名前は染み付いているのには， SSL の Deprecation が 2011 年と比較的最近であることに加えて OpenSSL というライブラリの存在があると感じます．いつの日か OpenTLS という名前に変更されることを祈願して，終わりのあいさつとします．

---

Cover photo by Towfiqu barbhuiya via Unsplash
