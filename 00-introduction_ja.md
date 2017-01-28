# BOLT #0: イントロダクションとインデックス

ようこそ！このLightning Networkの基本技術（BOLT）のドキュメントは、オフチェーンを利用したBitcoinの送付と
オンチェーン取引のためのレイヤー２プロトコルについて記述しています。

ここに記載している内容のバックグラウンドについて強調して書いているつもりですが、不足している部分もあると思うので
よく分からない部分や間違った記載を見つけた場合は、私達に連絡して改善の手助けをしてください。

これはバージョン0です。

1. [BOLT #1](01-messaging.md): ベースプロトコル
2. [BOLT #2](02-peer-protocol.md): チャネル管理用のピアプロトコル
3. [BOLT #3](03-transactions.md): Bitcoinのトランザクションとスクリプトのフォーマット
4. [BOLT #4](04-onion-routing.md): オニオンルーティングプロトコル
5. [BOLT #5](05-onchain.md): オンチェーントランザクションをハンドリングするための推奨事項
6. [BOLT #6](06-irc-announcements.md): 暫定ノードとチャネルの検出
7. [BOLT #7](07-routing-gossip.md): P2Pノードとチャネルの検出
8. [BOLT #8](08-transport.md): 暗号化及び認証されたトランスポート

## 用語集

* *ファンディングトランザクション*:
   * チャネル上の両ピアに支払いをする不可逆のオンチェーントランザクション。このトランザクションを使用するには両者の合意が必要となります。


* *チャネル*:
   * ２つの*ピア*間で相互交換を行う高速なオフチェーンの方式。資金を移動する際は、更新した*コミットメントトランザクション*の署名を交換します。


* *コミットメントトランザクション*:
   * ファンディングトランザクションを使用するトランザクションで、各ピアは他のピアから受け取った
        コミットメントトランザクションを使用するのに必要な署名を保持しているため、
        いつでもコミットメントトランザクションを使用できます。
        新しいコミットメントトランザクションが交換されると、古いコミットメントトランザクションは*取り消されます*。

* *HTLC*: Hashed Time Locked Contract.
   * ２つのピア間の条件付き決済：受取人は署名及び*プリイメージ*を提示することでコインを入手することができ、
         提示されない場合は支払人が所定の時間後に契約を取り消すことができます。
         これらは*コミットメントトランザクション*の出力として実装されます。

* *支払いハッシュとプリイメージ*:
   * HTLCには、プリイメージのハッシュが含まれています。最終的な受取人のみがプリイメージを知っているため、
         資金を入手するためにプリイメージを明らかにしたということは、支払いを受けた証拠とみなされます。


* *コミットメント取消キー*:
   * 全ての*コミットメントトランザクション*には一意の*コミットメント取消キー*があり、これを使うと他のピアが
         全ての出力をすぐに使用できるようになります。この鍵を明らかにすることは、古いコミットメントトランザクションを
         取り消すことを意味します。この取消機能のため、各出力はコミットメント取消公開鍵を参照します。

* *コミットメント毎のシークレット*:
   * 全てのコミットメントは*コミットメント毎のシークレット*から鍵を導出します。
     これは今までの一連のコミットメントのシークレットから生成されます。

* *協調閉鎖*:
   * *ファンディングトランザクション*を最終的な各ピアの残高で各ピア宛てに送る出力を持つトランザクションを
         ブロードキャストすることで、両ピアが連携してチャネルを閉じます。

* *一方的な閉鎖*:
   * *コミットメントトランザクション*をブロードキャストする非協力的なチャネルの閉鎖。
        このトランザクションは協調閉鎖のトランザクションよりサイズが大きく効率悪く、
        *コミットメントトランザクション*をブロードキャストしたピアは、予め*コミットメントトランザクション*に設定されている
        所定の期間、自身のコインにアクセスすることができません。


* *取消トランザクションによる閉鎖*:
   * 取り消された古い*コミットメントトランザクション*をブロードキャストすることによる無効なチャネルの閉鎖。
         古い*コミットメントトランザクション*がブロードキャストされたことを知った相手のピアは、
         *コミットメント取消シークレットキー*を知っているため、*ペナルティトランザクション*を作成することができます。


* *ペナルティトランザクション*:
   * *コミットメント取消シークレットキー*を使って取消トランザクションの全ての出力を使用することができるトランザクション。
         他のピアが取り消された*コミットメントトランザクション*をブロードキャストして不正行為をしようとする場合に、
         ピアはこれを使います。
         

* *コミットメント番号*:
   * *コミットメントトランザクション*毎の48ビットの増分カウンタ。チャネル内の各ピアから独立しており、0から始まります。

* *チャネルショートID*:
   * *ファンディングトランザクション*（＝ チャネル）のグローバルで一意な8バイトの識別子。

* *It's ok to be odd*:
   * 機能がオプションかサポートが強制されるものかを示す数値フィールドに適用されるルール。
     数値が偶数の場合は両方のエンドポイントが機能をサポートしなければならないことを示し、
     奇数の場合は機能が他のエンドポイントによって無視される可能性があることを示します。

## テーマソング


      Why this network could be democratic...
      Numismatic...
      Cryptographic!
      Why it could be released Lightning!
      (Release Lightning!)


      We'll have some timelocked contracts with hashed pubkeys, oh yeah.
      (Keep talking, whoa keep talkin')
      We'll segregate the witness for trustless starts, oh yeah.
      (I'll get the money, I've got to get the money)
      With dynamic onion routes, they'll be shakin' in their boots;
      You know that's just the truth, we'll be scaling through the roof.
      Release Lightning!
      (Go, go, go, go; go, go, go, go, go, go)


      [Chorus:]
      Oh released Lightning, it's better than a debit card..
      (Release Lightning, go release Lightning!)
      With released Lightning, micropayments just ain't hard...
      (Release Lightning, go release Lightning!)
      Then kaboom: we'll hit the moon -- release Lightning!
      (Go, go, go, go; go, go, go, go, go, go)


      We'll have QR codes, and smartphone apps, oh yeah.
      (Ooo ooo ooo ooo ooo ooo ooo)
      P2P messaging, and passive incomes, oh yeah.
      (Ooo ooo ooo ooo ooo ooo ooo)
      Outsourced closure watch, gives me feelings in my crotch.
      You'll know it's not a brag when the repo gets a tag:
      Released Lightning.


      [Chorus]
      [Instrumental, ~1m10s]
      [Chorus]
      (Lightning! Lightning! Lightning! Lightning!
       Lightning! Lightning! Lightning! Lightning!)


      C'mon guys, let's get to work!


   -- Anthony Towns <aj@erisian.com.au>


## 著者


[ FIXME: Insert Author List ]


![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
この仕様は [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/)のライセンス下にあります。
