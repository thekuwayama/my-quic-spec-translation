- [Abdtract](#abdtract)
- [1.  Introduction](#1--introduction)
- [2.  Conventions and Terminology](#2--conventions-and-terminology)
- [3.  Potential Usages of OWL in QUIC](#3--potential-usages-of-owl-in-quic)
  - [3.1.  Crucial Data Scheduling](#31--crucial-data-scheduling)
  - [3.2.  Congestion Control](#32--congestion-control)
  - [3.3.  Packet Retransmission](#33--packet-retransmission)
- [4.  OWL Measurement](#4--owl-measurement)

# Abdtract

この文書では、QUICのマルチパス通信をより良くするためにOne Way Latency（OWL）を使用する方法について記述します。 輻輳制御メカニズム、再送ポリシー、重要なデータのスケジューリングなど、OWLのいくつかの代表的な使用法を分析します。 2種類のOWL測定手法を提示し、比較します。 QUICのパフォーマンスを向上させるために、OWLに関連するより多くの調査が行われるでしょう。

# 1.  Introduction
ラウンドトリップタイム（RTT）は、データ通信において輻輳制御および損失回復メカニズムで一般的に使用されます。 しかし、データ通信の重要な問題は、単に帰りの経路を含まないパスのデータ通信の遅延だけである。 2つのピア間のアップリンクおよびダウンリンクのレイテンシは非常に異なる場合があります。RTTは、経路上のデータ通信の遅延を正確に反映することができません。その経路に沿った反対方向のレイテンシの影響を容易にうけます。 したがって、データが送信されてから受信されるまでの正確なレイテンシを評価するために、One Way Latency（OWL）[I-D.song-mptcp-owl]を使用することが提案されています。

One Way Latencyは、QUIC [I-D.ietf-quic-transport]のACKフレームのタイムスタンプ情報を使用して、絶対値または相対値で計算できます。 マルチパスはQUICによってサポートされるため、One Way Latencyに基づくパス選択は、輻輳制御、パケットの再送、重要なデータのスケジューリングなど、いくつかの状況でQUICのマルチパスのパフォーマンスを向上させることができます。

QUICでは、OWLの必要な考慮事項について議論することをお勧めします。    以下では、QUICにおけるOWLの可能な使用法を分析し、次に2種類のOWL測定値を列挙して比較します。


# 2.  Conventions and Terminology
(中略)

One Way Latency（OWL）：信号が送信されてから信号が受信されるまでの送信者と受信者の間の伝播遅延。

# 3.  Potential Usages of OWL in QUIC
OWLの用途は数多くあり、特にQUICのマルチパスの用途があります。 この文書には3つの重要な側面のみが示されていますが、より多くの研究がまだ必要です。

## 3.1.  Crucial Data Scheduling
送信処理中に、宛先にすぐに送信する必要のある重要なデータが存在することがよくあります。 そのような重要なデータは例えば、マルチメディアのキーフレーム、緊急通信の高優先度チャンクなどです。複数経路でRTTのみが使用される場合、複数経路上でデータ到着の順序を保証することはできません。

与えられた任意のリンクにおけるデータレートは、非対称のかもしれません。 さらに、ある方向の遅延は、パケット待ち行列の量に応じて変化する可能性があります。 したがって、パス内の順方向の遅延は、図1に例示したように逆方向の遅延と必ずしも同じではありません。

```
        --------   OWL(s-to-c,path1)=16ms   <--------
      /                                               \
     |     ----->  OWL(c-to-s,path1)= 5ms    -----     |
     |   /            RTT(path1)=21ms              \   |
     |  |                                           |  |
   +------+                                       +------+
   |      |----->  OWL(c-to-s,path2)= 8ms    -----|      |
   |Client|                                       |Server|
   |      |-----   OWL(s-to-c,path2)= 8ms   <-----|      |
   +------+           RTT(path2)=16ms             +------+
     |  |                                           |  |
     |   \                                         /   |
     |     ----->  OWL(c-to-s,path3)=10ms    -----     |
      \                                               /
        --------   OWL(s-to-c,path3)= 8ms   <--------
                      RTT(path3)=18ms
```

図1. に示すように、例えばクライアントとサーバーの間に3つのパスがあり、OWLが示されています。 RTT情報だけでは、サーバーへの最速のパスはパス2、パス3、パス1の順であることがクライアントに示されます。パス2は最も高速ですが、OWLはクライアントからサーバーへの最速パスはパス1、パス2、パス3の順番になります。

OWL測定の結果を使用すると、送信側は重要なデータ転送のために、順方向の待ち時間の観点から、より高速な経路を簡単に選択することができます。 さらに、これらの重要なデータのACK応答は、逆方向の最小レイテンシを持つパス上で送信することができます。 ピギーバックは、二重通信モードのときにも便利です。

# 3.2.  Congestion Control
ある方向の輻輳は必ずしも逆方向の輻輳を意味するものではない。
```
        --------   No congestion (path 1)   <--------
      /                                               \
     |     ----->  Congestion    (path 1)    -----     |
     |   /                                         \   |
     |  |                                           |  |
   +------+                                       +------+
   |Client|                                       |Server|
   +------+                                       +------+
     |  |                                           |  |
     |   \                                         /   |
     |     ----->  No congestion (path 2)    -----     |
      \                                               /
        --------   Congestion    (path 2)   <--------
```
図2に示すように、例えばクライアントとサーバーの間の2つのパスに輻輳状況があります。クライアントからサーバへの輻輳は経路1上で、またサーバからクライアントへは経路2上で輻輳しています。RTT情報だけでは両方の経路で輻輳が示されるが、OWL情報はクライアントからサーバへは経路2がより低負荷であることが分かります。

ある方向のネットワーク輻輳は、RTTを使用するのではなく、OWLを使用してよりよく評価できます。 特に、輻輳が単方向の状況になる可能性がある場合、クライアントからサーバーへのパスの輻輳は、サーバーからクライアントへのパスの輻輳とは異なります。 RTTは、経路上のデータ伝送の遅延を正確に反映することができません。  QUICのマルチパスの場合、クライアントはパケットを送るために負荷の低いパスを選択する必要があります[RFC6356]。 RTTを異なるパス間で比較することは賢明でなく、代わりにOWLをパス間で比較する必要があります。


# 3.3.  Packet Retransmission
Continuous Multipath Transmission（CMT）は、複数のパスを経由して送信元から送信先にホストに新しいデータを同時に転送することによってスループットを向上させます。 ただし、パケットが失われた場合、図3に示すようにReceive Buffer BlockingRBB）が発生します。

```
                 Stream 5, Offset    0, Length 500 (lost)
          -----> Stream 5, Offset 1000, Length 500 (rcvd) -----
        /        Stream 5, Offset 2000, Length 500 (rcvd)       \
       |                                                         |
   +------+                                                  +--------+
   |Sender|                                                  |Receiver|
   +------+                                                  +--------+
       |                                                         |
        \        Stream 5, Offset  500, Length 500 (rcvd)       /
          -----> Stream 5, Offset 1500, Length 500 (rcvd) -----
                 Stream 5, Offset 2500, Length 500 (rcvd)
```

図3に示すように、受信バッファブロッキングの例：ストリームID = 5のオクテット0〜499を含むパケットは失われました。 一方、ストリームID = 5のオクテット500-999,1000-1499,1500-1999,2000-2499を含むパケットはすべて受信されています。 オクテット500-2000はすべて受信機でバッファリングされましたが、損失したオクテット0-499によってブロックされます。したがって、送信側は、出来る限り早くを再送するのに適したパスを選択する必要があります。 OWL測定の結果を使用すると、送信者は最小の転送待ち時間で特定のパスを素早く判断できます。 受信機が再送信されたパケット内で最も必要とされるフレームを取得し、それらを上位層に提出するとすぐに、RBBを解放することができます。

# 4.  OWL Measurement
絶対値測定と相対値測定の2種類のOWL測定手法が利用できます。

OWLの絶対値を得るための、測定の主要条件はクロック同期である。 Network Time Protocol（NTP）[RFC5905]を使用すると、エンドホストはリモートNTPサーバでローカルクロックを較正できます。 付加的な情報またはオプション機能は、標準NTPヘッダ[RFC7822]の拡張フィールドを介して追加することもできます。 調整精度は、輻輳の少ない状況ではミリ秒レベルに達することがあります。 ここでの明らかな難しさは、エンドホストにNTPオプションを初期化させることです。

OWLの相対的値を得ることは、状況によっては、その上でアプリケーションを確立するのに十分なものです。 例えば、再送が必要な場合、送信者は、どの経路がその方向で最小の連天使を有するかだけを気にするだけになります。 帯域幅が推定されているとき、すべての利用可能な経路の間にその方向でのレイテンシ、すなわちレイテンシ時間の差が必要です。 通信相手エンドホストとパケットを送受信するローカルタイムスタンプと交換することにより、両者はOWLの相対値を得ることができます。

絶対値を得るための考慮事項は、追加のプロトコル要件と同期精度です。しかし、絶対値を使用することは、そのアプリケーションの役に立ちます。 反対に、相対的測定は、確認応答にタイムスタンプを送るだけでよく、クロック同期について心配する必要はありません。