---
title: "TCP"
---

## この章のゴール

第7章では、UDP で送ったデータが黙って消えても、送った側は気づきませんでした。Web ページやファイルの一部が黙って消えたら困ります。そこで使われるのが **TCP** です。

この章では、TCP の通信を `tcpdump` で1つずつ見て、さらに**わざとパケットを落として**、次のことを確かめます。

- TCP は、データを送る前に**相手と接続を確立する**（3ウェイハンドシェイク）
- 相手がいなければ、送った側は**すぐに気づく**
- 途中でパケットが消えても、**届くまで送り直す**

## 8-0. 準備

第7章と同じ2台を作ります。

```bash
sudo ip netns add pc1
sudo ip netns add pc2
sudo ip link add veth-pc1 type veth peer name veth-pc2
sudo ip link set veth-pc1 netns pc1
sudo ip link set veth-pc2 netns pc2
sudo ip netns exec pc1 ip addr add 192.168.0.1/24 dev veth-pc1
sudo ip netns exec pc2 ip addr add 192.168.0.2/24 dev veth-pc2
sudo ip netns exec pc1 ip link set veth-pc1 up
sudo ip netns exec pc2 ip link set veth-pc2 up
```

## 8-1. TCP で待ち受ける

`nc` は、`-u` を付けなければ TCP を使います。

**ターミナル2**：

```bash
sudo ip netns exec pc2 nc -l 9999
```

別のターミナルで、待ち受けの状態を見ます。TCP なので、`ss` には `-u` ではなく **`-t`** を付けます。

```bash
sudo ip netns exec pc2 ss -tlnp
```

```text
State  Recv-Q Send-Q Local Address:Port Peer Address:PortProcess
LISTEN 0      1            0.0.0.0:9999      0.0.0.0:*    users:(("nc",pid=1725,fd=3))
```

第7章の UDP では `State` が `UNCONN` でしたが、TCP では **`LISTEN`**（接続を待っている）になります。TCP は「接続」という考え方を持っているので、状態の名前もそれに合わせたものになっています。

## 8-2. 観察する —— 3ウェイハンドシェイク

TCP で `hello` を1回送るときに、ケーブルの上で何が起きるかを見ます。

**ターミナル3**：

```bash
sudo ip netns exec pc2 tcpdump -n -i veth-pc2 tcp
```

**ターミナル1**（ターミナル2の `nc -l 9999` は動かしたまま）：

```bash
echo hello | sudo ip netns exec pc1 nc -N 192.168.0.2 9999
```

`-N` は「送り終わったら接続を閉じる」という指定です。ターミナル2に `hello` と表示され、ターミナル2の `nc` も終了します。

ターミナル3には、次の8行が出ます。長いので、時刻と、行末の `options [...]` 以降を省いて載せます。

```text
IP 192.168.0.1.52812 > 192.168.0.2.9999: Flags [S], seq 1613710660, win 64240, ...
IP 192.168.0.2.9999 > 192.168.0.1.52812: Flags [S.], seq 2814943072, ack 1613710661, win 65160, ...
IP 192.168.0.1.52812 > 192.168.0.2.9999: Flags [.], ack 1, win 502, ...
IP 192.168.0.1.52812 > 192.168.0.2.9999: Flags [P.], seq 1:7, ack 1, win 502, ..., length 6
IP 192.168.0.2.9999 > 192.168.0.1.52812: Flags [.], ack 7, win 510, ...
IP 192.168.0.1.52812 > 192.168.0.2.9999: Flags [F.], seq 7, ack 1, win 502, ...
IP 192.168.0.2.9999 > 192.168.0.1.52812: Flags [F.], seq 1, ack 8, win 510, ...
IP 192.168.0.1.52812 > 192.168.0.2.9999: Flags [.], ack 2, win 502, ...
```

UDP では `hello` を運ぶパケットが1つ流れるだけでした。TCP では、**同じ `hello` を送るのに8つのパケット**が流れています。

`Flags [...]` の中の記号が、そのパケットの役割を表しています。

| 記号 | 名前 | 意味 |
|---|---|---|
| `S` | SYN | 接続したい |
| `.` | ACK | 受け取った（確認応答） |
| `P` | PSH | データを運んでいる |
| `F` | FIN | もう送るものはない（切断したい） |
| `R` | RST | 接続を拒否する、強制的に切る |

`S.` は「SYN と ACK の両方」という意味です。これを使って8行を読むと、次のようになります。

| # | 向き | Flags | 意味 |
|---|---|---|---|
| 1 | `pc1` → `pc2` | `S` | 接続していいですか？ |
| 2 | `pc2` → `pc1` | `S.` | いいですよ。こちらからも接続していいですか？ |
| 3 | `pc1` → `pc2` | `.` | どうぞ。（ここで接続が確立） |
| 4 | `pc1` → `pc2` | `P.` | `hello`（6バイト）を送ります |
| 5 | `pc2` → `pc1` | `.` | 6バイト受け取りました |
| 6 | `pc1` → `pc2` | `F.` | こちらはもう送るものはありません |
| 7 | `pc2` → `pc1` | `F.` | 了解。こちらも送るものはありません |
| 8 | `pc1` → `pc2` | `.` | 了解。（ここで切断） |

最初の3つ（`S` → `S.` → `.`）が、TCP が通信を始めるときに必ず行うあいさつ、**3ウェイハンドシェイク**です。データを送る前に、相手が本当にいて、受け取る準備ができていることを確かめているのです。

### seq と ack —— 何バイト目まで届いたか

4行目の `seq 1:7` は「**1バイト目から6バイト目まで**（7の手前まで）を送る」、5行目の `ack 7` は「**6バイト目まで受け取ったので、次は7バイト目をください**」という意味です。

TCP は、送ったデータに通し番号（**シーケンス番号**、`seq`）を付けて、相手から「何番まで受け取ったか」（`ack`）を返してもらいます。だから送った側は、**どこまで届いたかを正確に知ることができます**。届いていない部分があれば、そこから送り直せばいいわけです。

:::message
1行目の `seq 1613710660` のような大きな数は、接続ごとにランダムに選ばれる最初の番号です。`tcpdump` は2行目以降、読みやすいように「最初の番号からの差」に直して表示しています。4行目の `seq 1:7` の `1` は、その差の値です。
:::

## 8-3. 接続を見る

接続が続いている間の状態を見てみます。

**ターミナル2**：

```bash
sudo ip netns exec pc2 nc -l 9999
```

**ターミナル1**（何も打たずにそのままにしておく）：

```bash
sudo ip netns exec pc1 nc 192.168.0.2 9999
```

**ターミナル3**：

```bash
sudo ip netns exec pc1 ss -tn
sudo ip netns exec pc2 ss -tn
```

```text
State Recv-Q Send-Q Local Address:Port  Peer Address:PortProcess
ESTAB 0      0        192.168.0.1:52828  192.168.0.2:9999
State Recv-Q Send-Q Local Address:Port Peer Address:Port Process
ESTAB 0      0        192.168.0.2:9999  192.168.0.1:52828
```

両側とも **`ESTAB`**（ESTABLISHED、接続が確立している）です。`pc1` から見ると「自分の 52828 番と、相手の 9999 番がつながっている」、`pc2` から見ると「自分の 9999 番と、相手の 52828 番がつながっている」。1本の接続を、両端から見ているわけです。

TCP の接続は、**送信元 IP・送信元ポート・宛先 IP・宛先ポートの4つの組**で区別されます。だから Web サーバは、同じ 443 番で何千人もの接続を同時に受けられます。相手の IP やポートが違えば、別の接続だからです。

確認したら、ターミナル1と2を `Ctrl+C` で止めておきます。

## 8-4. 壊してみる① —— 誰もいないポートに接続したら

第7章の 7-7 と同じく、誰も待ち受けていない 7777 番に送ってみます。

**ターミナル2**：

```bash
sudo ip netns exec pc1 tcpdump -n -i veth-pc1 tcp
```

**ターミナル1**（`-v` は、何が起きたかを表示する指定）：

```bash
echo hello | sudo ip netns exec pc1 nc -v -N 192.168.0.2 7777; echo exit=$?
```

```text
nc: connect to 192.168.0.2 port 7777 (tcp) failed: Connection refused
exit=1
```

ターミナル2（時刻と `options` 以降は省いています）：

```text
IP 192.168.0.1.54206 > 192.168.0.2.7777: Flags [S], seq 2136379072, win 64240, ...
IP 192.168.0.2.7777 > 192.168.0.1.54206: Flags [R.], seq 0, ack 2136379073, win 0, length 0
```

`pc1` の「接続していいですか？」（`S`）に、`pc2` が **`R.`（RST、拒否）** を返しました。`nc` は **`Connection refused`**（接続を拒否された）と表示し、`exit=1`（失敗）で終わっています。

| | UDP（第7章 7-7） | TCP |
|---|---|---|
| 相手の返事 | ICMP `port unreachable` | TCP の `RST` |
| `nc` の終わり方 | `exit=0`（成功） | `exit=1`（失敗）、`Connection refused` |
| データ（`hello`）は流れたか | 流れた（そして捨てられた） | **流れていない**。接続の段階で断られた |

TCP は、3ウェイハンドシェイクで相手がいることを確かめてからデータを送ります。だから、**相手がいなければデータを送る前に分かり**、送った側のプログラムもそれに気づけます。

:::message
`Connection refused` は、現場でとてもよく見るエラーです。意味は「相手の機器までは届いたが、そのポートで待ち受けているプログラムがいない」。ネットワークではなく、**相手のプログラムが起動しているか、ポート番号が合っているか**を疑います。`ss -tlnp` で確かめましょう。
:::

## 8-5. 壊してみる② —— 途中でケーブルが切れたら

いよいよ、TCP の一番大事な性質を確かめます。接続した後で**通信路が一時的に切れたら**どうなるでしょうか。

パケットを落とすには、Linux の **`tc`**（traffic control）コマンドと、その中の **netem**（network emulator）という機能を使います。netem は、パケットをわざと落としたり遅らせたりして、調子の悪いネットワークを再現できます。

まず接続します。

**ターミナル2**：

```bash
sudo ip netns exec pc2 nc -l 9999
```

**ターミナル1**：

```bash
sudo ip netns exec pc1 nc 192.168.0.2 9999
```

**ターミナル3**で、`pc1` から出ていくパケットを**100%落とす**設定を入れます。ケーブルが切れたのと同じ状態です。

```bash
sudo ip netns exec pc1 tc qdisc add dev veth-pc1 root netem loss 100%
sudo ip netns exec pc1 tc qdisc show dev veth-pc1
```

```text
qdisc netem 8001: root refcnt 3 limit 1000 loss 100%
```

この状態で、**ターミナル1に `hello` と打って Enter** を押します。ターミナル2には何も表示されません。

数秒待ってから、ターミナル3で `pc1` の接続の詳しい状態を見ます。`-i` は詳しい情報を表示する指定です。

```bash
sudo ip netns exec pc1 ss -tni
```

```text
State Recv-Q Send-Q Local Address:Port  Peer Address:PortProcess
ESTAB 0      6        192.168.0.1:58556  192.168.0.2:9999
	 cubic wscale:7,7 rto:3200 backoff:4 rtt:0.022/0.011 mss:1448 ... bytes_retrans:30 ... unacked:1 retrans:1/5 lost:1 ...
```

（2行目はとても長いので、一部を省いています。）

見るべきところは4つです。

| 項目 | 値 | 意味 |
|---|---|---|
| `Send-Q` | `6` | 送ったが、まだ「受け取った」と言われていないデータが 6 バイト（`hello` と改行） |
| `retrans:1/5` | 5 | これまでに **5回送り直した** |
| `rto:3200` | 3200 | 次に送り直すまでの待ち時間が **3200 ミリ秒** |
| `backoff:4` | 4 | 待ち時間を **4回倍にした** |

TCP は、送ったデータに `ack` が返ってこないと、しばらく待ってから**自動で送り直します**（**再送**）。この待ち時間（**RTO**、再送タイムアウト）は、送り直すたびに倍に延びていきます。最初は 200 ミリ秒ほどですが、`backoff:4` で 2⁴ = 16 倍の 3200 ミリ秒になりました。ネットワークが混んでいるときに、再送でさらに混ませないための工夫です。

しばらく時間をおいて `ss -tni` をもう一度打つと、`retrans` と `rto` がさらに増えているはずです。

では、**ケーブルをつなぎ直します**。netem の設定を消します。

```bash
sudo ip netns exec pc1 tc qdisc del dev veth-pc1 root
```

しばらくすると（次の再送のタイミングで。待ち時間が延びていると十数秒かかります）、ターミナル2に表示されます。

```text
hello
```

**消えたはずの `hello` が、遅れて届きました**。もう一度 `ss -tni` を見ると、`Send-Q` が `0`（未確認のデータは無い）に戻っています。

```bash
sudo ip netns exec pc1 ss -tni
```

```text
State Recv-Q Send-Q Local Address:Port  Peer Address:PortProcess
ESTAB 0      0        192.168.0.1:58556  192.168.0.2:9999
	 cubic wscale:7,7 rto:201 rtt:0.144/0.252 mss:1448 ... bytes_retrans:42 ... retrans:0/7 ...
```

`rto` も 201 ミリ秒に戻りました。確認したら、ターミナル1と2を `Ctrl+C` で止めます。

### UDP で同じことをすると

比べるために、UDP で同じことをします。

**ターミナル2**：

```bash
sudo ip netns exec pc2 nc -u -l 9999
```

**ターミナル3**（ケーブルを切る → 送る → つなぎ直す）：

```bash
sudo ip netns exec pc1 tc qdisc add dev veth-pc1 root netem loss 100%
echo hello-udp | sudo ip netns exec pc1 nc -u -w 1 192.168.0.2 9999
sudo ip netns exec pc1 tc qdisc del dev veth-pc1 root
```

ターミナル2には、いくら待っても何も表示されません。UDP は送り直さないので、**切れている間に送ったデータは消えたまま**です。確認したら、ターミナル2を `Ctrl+C` で止めます。

## 8-6. 調子の悪いネットワークで大きなデータを送る

最後に、**30% のパケットが落ちる**ひどいネットワークで、10万行のデータを送ってみます。

**ターミナル2**（受け取ったデータをファイルに保存する）：

```bash
sudo ip netns exec pc2 nc -l 9999 > /tmp/recv.txt
```

**ターミナル1**：

```bash
sudo ip netns exec pc1 tc qdisc add dev veth-pc1 root netem loss 30%
seq 1 100000 > /tmp/send.txt
md5sum /tmp/send.txt
sudo ip netns exec pc1 nc -N 192.168.0.2 9999 < /tmp/send.txt
```

`seq 1 100000` は 1 から 100000 までの数を1行ずつ書き出すコマンドで、約 59 万バイトのファイルができます。`md5sum` は、ファイルの中身から**指紋**（ハッシュ値）を計算するコマンドで、中身が1文字でも違えば別の値になります。

```text
dea9193b768319cbb4ff1a137ac03113  /tmp/send.txt
```

ターミナル2の `nc` が終了したら、全部受け取り終わった合図です。パケットが落ちるぶん、ふだんより時間がかかります。どれくらいかかるかは、落ちるパケットの運次第で、数秒のこともあれば1分以上のこともあります。

:::message
ターミナル1の `nc` は、ターミナル2より先に終わることがあります。送る側の `nc` は、データを Linux に渡し終えた時点で終了し、相手に届き終わるのを待たないからです。その後の送り直しは、プログラムが終わった後も Linux の TCP が続けてくれます。
:::

受け取ったファイルの指紋を比べます。

```bash
md5sum /tmp/recv.txt
```

```text
dea9193b768319cbb4ff1a137ac03113  /tmp/recv.txt
```

**送ったファイルとまったく同じ**です。3割のパケットが落ちるネットワークでも、TCP は1バイトも欠けずに、順番どおりに届けました。どれだけ送り直したかは、`nstat` で見られます。

```bash
sudo ip netns exec pc1 nstat -az TcpRetransSegs
```

```text
#kernel
TcpRetransSegs                  169                0.0
```

`pc1` の TCP は、この netns ができてから **169 回**送り直していました（8-5 の分も含みます。回数は毎回変わります）。プログラム（`nc`）は何もしていません。**送り直しも順番の並べ直しも、全部 TCP がやってくれた**のです。そのかわり、時間はかかりました。

netem の設定を消しておきます。

```bash
sudo ip netns exec pc1 tc qdisc del dev veth-pc1 root
```

## 8-7. TCP と UDP の使い分け

| | TCP | UDP |
|---|---|---|
| 始める前 | 3ウェイハンドシェイクで接続 | いきなり送る |
| 相手がいないとき | `Connection refused` ですぐ分かる | 送った側は気づかないことが多い |
| 届いたかの確認 | `ack` で確認する | しない |
| 消えたとき | 届くまで送り直す | 消えたまま |
| 順番 | 並べ直して届ける | 届いた順のまま |
| 速さ・軽さ | 確認や再送のぶん遅く、重い | 速く、軽い |
| 主な用途 | Web（HTTP）、SSH、メール、ファイル転送 | DNS、動画・音声通話、時刻合わせ、ゲーム |

**欠けたら困るデータは TCP、遅れるくらいなら捨てたいデータは UDP**、と覚えておきましょう。通話で1秒前の音声が遅れて届いても、もう使い道がありません。

## 8-8. 片付け

```bash
sudo ip -all netns delete
rm -f /tmp/send.txt /tmp/recv.txt
```

## まとめ

- **TCP** は、データを送る前に **3ウェイハンドシェイク**（`S` → `S.` → `.`）で接続を確立する。待ち受けは `LISTEN`、接続中は `ESTAB`（`ss -tn`）
- 接続は **送信元 IP・送信元ポート・宛先 IP・宛先ポート**の4つの組で区別される
- 相手のポートで誰も待ち受けていなければ **`RST`** が返り、**`Connection refused`** になる。ネットワークではなく相手のプログラムとポート番号を疑う
- TCP は送ったデータに通し番号（`seq`）を付け、相手は「何番まで受け取ったか」（`ack`）を返す。`ack` が来なければ**自動で再送**し、待ち時間（RTO）は再送のたびに倍に延びる
- `tc qdisc ... netem loss` で、パケットが落ちるネットワークを再現できる
- パケットが3割落ちても、TCP は欠けずに順番どおり届ける。そのかわり遅くなる。UDP は速いが、消えたら消えたまま

ここまでの章では、`192.168.x.x` のような**プライベートアドレス**だけを使ってきました。でも、家の PC も `192.168.x.x` なのに、インターネット上のサーバと通信できています。次の章では、プライベートアドレスのままインターネットに出ていくための仕組み、**NAT** を作ります。
