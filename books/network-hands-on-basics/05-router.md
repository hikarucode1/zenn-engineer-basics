---
title: "ルータを作る"
---

## この章のゴール

第4章の練習問題で、**同じスイッチにつながっていても、町（ネットワーク）が違えば直接は通信できない**ことを確かめました。

この章では、`192.168.0.x` の町と `192.168.1.x` の町をつなぐ**ルータ**を作ります。そして次の3つを確かめます。

- 町の外へ出るときは、**デフォルトゲートウェイ**（町の出口になるルータ）に頼む
- ルータは**転送する設定**をしないと、ただの「口が2つある PC」でしかない
- ルータを通ると、**IP アドレスはそのまま、MAC アドレスは付け替えられる**

## 5-1. 作るもの

```text
     192.168.0.0/24 の町                     192.168.1.0/24 の町

  pc1                          r1                          pc2
  veth-pc1 ============ r1-eth0    r1-eth1 ============ veth-pc2
  192.168.0.1     192.168.0.254    192.168.1.254     192.168.1.2
```

ルータ `r1` は、**口を2つ持ち、それぞれ別の町に住所を持つ**機器です。`r1-eth0` は `192.168.0.x` の町に、`r1-eth1` は `192.168.1.x` の町にいます。

スイッチは省略して、PC とルータを veth で直接つなぎます。第4章のスイッチを間に挟んでも、この章で起きることは変わりません。

:::message
ルータの住所に `.254` を使っているのは、町の番地のうち、端の番号をルータに使うことが多いからです。家庭用ルータの `192.168.0.1` のように、`.1` を使う流儀もあります。
:::

## 5-2. 組み立てる

3台の箱を作ります。ルータも、PC やスイッチと同じく netns で作ります。

```bash
sudo ip netns add pc1
sudo ip netns add pc2
sudo ip netns add r1
```

ケーブルを2本作り、それぞれの端を挿します。

```bash
sudo ip link add veth-pc1 type veth peer name r1-eth0
sudo ip link add veth-pc2 type veth peer name r1-eth1
sudo ip link set veth-pc1 netns pc1
sudo ip link set r1-eth0 netns r1
sudo ip link set veth-pc2 netns pc2
sudo ip link set r1-eth1 netns r1
```

住所を付けて、口を `up` にします。ルータには口が2つあるので、住所も2つ付けます。

```bash
sudo ip netns exec pc1 ip addr add 192.168.0.1/24 dev veth-pc1
sudo ip netns exec pc2 ip addr add 192.168.1.2/24 dev veth-pc2
sudo ip netns exec r1 ip addr add 192.168.0.254/24 dev r1-eth0
sudo ip netns exec r1 ip addr add 192.168.1.254/24 dev r1-eth1
sudo ip netns exec pc1 ip link set veth-pc1 up
sudo ip netns exec pc2 ip link set veth-pc2 up
sudo ip netns exec r1 ip link set r1-eth0 up
sudo ip netns exec r1 ip link set r1-eth1 up
```

ルータのルーティングテーブルを見てみます。

```bash
sudo ip netns exec r1 ip route
```

```text
192.168.0.0/24 dev r1-eth0 proto kernel scope link src 192.168.0.254
192.168.1.0/24 dev r1-eth1 proto kernel scope link src 192.168.1.254
```

第1章で見たとおり、住所を付けた口が `up` になると、その先の町への道が自動で書き足されます。`r1` は2つの町に住所があるので、**両方の町への道を知っています**。

## 5-3. まず、そのまま送ってみる

`pc1` から、同じ町にいるルータ（`192.168.0.254`）には届きます。

```bash
sudo ip netns exec pc1 ping -c 1 192.168.0.254
```

```text
PING 192.168.0.254 (192.168.0.254) 56(84) bytes of data.
64 bytes from 192.168.0.254: icmp_seq=1 ttl=64 time=0.026 ms

--- 192.168.0.254 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.026/0.026/0.026/0.000 ms
```

では、ルータの向こう側にいる `pc2`（`192.168.1.2`）はどうでしょうか。

```bash
sudo ip netns exec pc1 ping -c 1 -W 1 192.168.1.2
```

```text
ping: connect: Network is unreachable
```

第2章で見た症状です。`pc1` の道案内の表には `192.168.0.0/24` への道しかなく、よその町への道がありません。ケーブルの先にルータがいても、**`pc1` はそれがルータだと知らない**のです。

## 5-4. デフォルトゲートウェイ —— 「町の外はこの人に頼む」

`pc1` に「自分の町以外の宛先は、全部 `192.168.0.254` に頼む」と教えます。この「頼む先」を**デフォルトゲートウェイ**と呼びます。

```bash
sudo ip netns exec pc1 ip route add default via 192.168.0.254
sudo ip netns exec pc1 ip route
```

```text
default via 192.168.0.254 dev veth-pc1
192.168.0.0/24 dev veth-pc1 proto kernel scope link src 192.168.0.1
```

1行増えました。`default` は「ほかのどの行にも当てはまらない宛先すべて」、`via 192.168.0.254` は「`192.168.0.254` に渡す」という意味です。

`ip route get` で `pc1` の判断を聞いてみます。

```bash
sudo ip netns exec pc1 ip route get 192.168.1.2
```

```text
192.168.1.2 via 192.168.0.254 dev veth-pc1 src 192.168.0.1 uid 0
    cache
```

第2章の `ip route get 192.168.0.200` と比べると、**`via 192.168.0.254`** が増えています。「`192.168.1.2` は町の外なので、`192.168.0.254` 経由で送る」という判断です。

:::message
デフォルトゲートウェイには、**自分と同じ町の住所**しか指定できません。たとえば `pc1` に `ip route add default via 192.168.1.254` と打つと、`Error: Nexthop has invalid gateway.` と断られます。`pc1` は町の外の相手に直接は渡せないからです。
:::

## 5-5. 壊れている —— ルータが転送しない

これで届くはずです。

```bash
sudo ip netns exec pc1 ping -c 2 -W 1 192.168.1.2
```

```text
PING 192.168.1.2 (192.168.1.2) 56(84) bytes of data.

--- 192.168.1.2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1024ms

```

届きません。症状は `Network is unreachable` から `100% packet loss` に変わったので、`pc1` は送り出せています。どこで止まっているのかを調べます。

ルータの `pc2` 側の口（`r1-eth1`）で待ち受けます。

**ターミナル2**：

```bash
sudo ip netns exec r1 tcpdump -n -i r1-eth1 arp or icmp
```

**ターミナル1**：

```bash
sudo ip netns exec pc1 ping -c 2 -W 1 192.168.1.2
```

ターミナル2には何も出ません。`ping` のデータは、**ルータの `pc2` 側の口から出ていません**。ルータの中で止まっています。

原因は、Linux の初期設定です。

```bash
sudo ip netns exec r1 sysctl net.ipv4.ip_forward
```

```text
net.ipv4.ip_forward = 0
```

`ip_forward` は「**自分宛てではない IP パケットを、別の口から送り出す（転送する）か**」という設定で、初期値は `0`（転送しない）です。ふつうの PC が勝手にルータとして働くと困るので、最初はオフになっています。

`r1` は口を2つ持っていても、転送しない限り「口が2つある PC」にすぎません。届いたパケットの宛先が自分（`192.168.0.254` か `192.168.1.254`）でなければ、捨ててしまいます。

:::message
転送がオフでも、`pc1` から `r1` の反対側の住所（`192.168.1.254`）への `ping` は通ります。宛先が `r1` 自身なので、転送ではないからです。「ルータの向こう側の住所に `ping` が通るから、ルータは動いている」とは言えないので注意してください。
:::

`ip_forward` を `1` にします。`sysctl -w` は Linux の設定値を書き換えるコマンドです。

```bash
sudo ip netns exec r1 sysctl -w net.ipv4.ip_forward=1
```

```text
net.ipv4.ip_forward = 1
```

## 5-6. まだ壊れている —— 帰り道がない

もう一度 `ping` を打ちます。今度は `pc2` 側で待ち受けます。

**ターミナル2**：

```bash
sudo ip netns exec pc2 tcpdump -n -i veth-pc2 arp or icmp
```

**ターミナル1**：

```bash
sudo ip netns exec pc1 ping -c 2 -W 1 192.168.1.2
```

ターミナル1の結果は、まだ `100% packet loss` です。でもターミナル2には、次のように出ます（時刻は省いています）。

```text
ARP, Request who-has 192.168.1.2 tell 192.168.1.254, length 28
ARP, Reply 192.168.1.2 is-at c6:f4:41:ba:5d:7b, length 28
IP 192.168.0.1 > 192.168.1.2: ICMP echo request, id 1704, seq 1, length 64
IP 192.168.0.1 > 192.168.1.2: ICMP echo request, id 1704, seq 2, length 64
```

前進しました。ルータが `pc2` の MAC アドレスを ARP で調べ、`ping` のデータ（echo request）を `pc2` まで届けています。ところが、**`pc2` は返事（echo reply）を送っていません**。

`pc2` に、返事の宛先 `192.168.0.1` への道を聞いてみます。

```bash
sudo ip netns exec pc2 ip route get 192.168.0.1
```

```text
RTNETLINK answers: Network is unreachable
```

`pc2` にはデフォルトゲートウェイがないので、**町の外にいる `pc1` への帰り道がありません**。届いた `ping` に返事をしようとしても、送り出せないのです。第2章の 2-5 と同じく、**行きの道と帰りの道は、それぞれの機器が別々に持っている**ことを思い出してください。

`pc2` にも、デフォルトゲートウェイを教えます。`pc2` の町の出口は、ルータの `192.168.1.254` です。

```bash
sudo ip netns exec pc2 ip route add default via 192.168.1.254
```

## 5-7. 通った

```bash
sudo ip netns exec pc1 ping -c 3 192.168.1.2
```

```text
PING 192.168.1.2 (192.168.1.2) 56(84) bytes of data.
64 bytes from 192.168.1.2: icmp_seq=1 ttl=63 time=0.018 ms
64 bytes from 192.168.1.2: icmp_seq=2 ttl=63 time=0.032 ms
64 bytes from 192.168.1.2: icmp_seq=3 ttl=63 time=0.033 ms

--- 192.168.1.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2042ms
rtt min/avg/max/mdev = 0.018/0.028/0.033/0.007 ms
```

**町を越えて届きました**。ここまでに直したことを整理しておきます。

| 段階 | `pc1` から見た症状 | 止まっていた場所 | 直したこと |
|---|---|---|---|
| 1 | `Network is unreachable` | `pc1`：町の外への道がない | `pc1` にデフォルトゲートウェイ |
| 2 | `100% packet loss` | `r1`：転送しない | `r1` の `ip_forward=1` |
| 3 | `100% packet loss` | `pc2`：帰り道がない | `pc2` にデフォルトゲートウェイ |

2と3は、`pc1` から見るとまったく同じ症状です。**`tcpdump` で「どこまで届いているか」を順に見ていく**と、止まっている場所を特定できます。

### TTL が減った

`ttl=63` に注目してください。第4章までは、ずっと `ttl=64` でした。

TTL（Time To Live）は IP パケットの「寿命」で、**ルータを1台通るたびに1減ります**。`pc2` は TTL 64 で返事を送り出し、ルータ `r1` を1台通って 63 になって `pc1` に届きました。スイッチ（第4章）では減らなかったので、TTL の減り方を見れば**途中にルータが何台あるか**が分かります。

## 5-8. 観察する —— ルータは封筒を入れ替える

最後に、ルータの両側で同時に `tcpdump` を取り、封筒（イーサネットフレーム）の宛名を比べます。ターミナルを3つ使います。

**ターミナル2**（ルータの `pc1` 側）：

```bash
sudo ip netns exec r1 tcpdump -n -e -i r1-eth0 icmp
```

**ターミナル3**（ルータの `pc2` 側）：

```bash
sudo ip netns exec r1 tcpdump -n -e -i r1-eth1 icmp
```

**ターミナル1**：

```bash
sudo ip netns exec pc1 ping -c 1 192.168.1.2
```

ターミナル2（`pc1` 側の口、`r1-eth0`）：

```text
7a:87:09:62:f1:9c > 1a:7f:ad:06:7d:92, ethertype IPv4 (0x0800), length 98:
    192.168.0.1 > 192.168.1.2: ICMP echo request, id 1734, seq 1, length 64
1a:7f:ad:06:7d:92 > 7a:87:09:62:f1:9c, ethertype IPv4 (0x0800), length 98:
    192.168.1.2 > 192.168.0.1: ICMP echo reply, id 1734, seq 1, length 64
```

ターミナル3（`pc2` 側の口、`r1-eth1`）：

```text
d6:f3:5b:fb:83:b1 > c6:f4:41:ba:5d:7b, ethertype IPv4 (0x0800), length 98:
    192.168.0.1 > 192.168.1.2: ICMP echo request, id 1734, seq 1, length 64
c6:f4:41:ba:5d:7b > d6:f3:5b:fb:83:b1, ethertype IPv4 (0x0800), length 98:
    192.168.1.2 > 192.168.0.1: ICMP echo reply, id 1734, seq 1, length 64
```

どの MAC アドレスが誰のものかは、`ip link` で確かめられます（値は環境ごとに違います）。

| MAC アドレス | 持ち主 |
|---|---|
| `7a:87:09:62:f1:9c` | `pc1` の `veth-pc1` |
| `1a:7f:ad:06:7d:92` | `r1` の `r1-eth0`（`pc1` 側） |
| `d6:f3:5b:fb:83:b1` | `r1` の `r1-eth1`（`pc2` 側） |
| `c6:f4:41:ba:5d:7b` | `pc2` の `veth-pc2` |

行き（echo request）の1つのパケットを、ルータの両側で比べます。

| 場所 | 送信元 MAC → 宛先 MAC | 送信元 IP → 宛先 IP |
|---|---|---|
| `pc1` 側のケーブル | `pc1` → **`r1-eth0`** | `192.168.0.1` → `192.168.1.2` |
| `pc2` 側のケーブル | **`r1-eth1`** → `pc2` | `192.168.0.1` → `192.168.1.2` |

ここがこの章でいちばん大事なところです。

- **IP アドレスは、最初から最後まで同じ**（`pc1` → `pc2`）
- **MAC アドレスは、ケーブルごとに違う**。`pc1` 側のケーブルでは「`pc1` からルータへ」、`pc2` 側のケーブルでは「ルータから `pc2` へ」

第3章で「IP アドレスは最終的な宛先、MAC アドレスは同じケーブルの上で次に受け取る相手」と書きました。その意味がここで分かります。`pc1` は `pc2` 宛ての IP パケットを、**ルータ宛ての封筒**に入れて送ります。ルータは封筒を開けて IP パケットを取り出し、宛先の IP アドレスを見て、**`pc2` 宛ての新しい封筒**に入れ直して送り出します。

`pc1` の ARP 表を見ると、これがよく分かります。

```bash
sudo ip netns exec pc1 ip neigh
```

```text
192.168.0.254 dev veth-pc1 lladdr 1a:7f:ad:06:7d:92 REACHABLE
```

`pc1` は `pc2` と通信しているのに、ARP 表には **`pc2` が載っていません**。`pc1` が MAC アドレスを調べたのはルータ（`192.168.0.254`）だけです。**町の外の相手の MAC アドレスは、知る必要がない**のです。

## 5-9. 壊してみる —— デフォルトゲートウェイを間違えたら

最後に、よくある設定ミスを再現します。`pc1` のデフォルトゲートウェイを、存在しない `192.168.0.253` に書き換えます。`ip route replace` は、既存の行を置き換えるコマンドです。

```bash
sudo ip netns exec pc1 ip route replace default via 192.168.0.253
sudo ip netns exec pc1 ping -c 2 -W 1 192.168.1.2
```

```text
PING 192.168.1.2 (192.168.1.2) 56(84) bytes of data.

--- 192.168.1.2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1014ms

```

```bash
sudo ip netns exec pc1 ip neigh
```

```text
192.168.0.253 dev veth-pc1 INCOMPLETE
192.168.0.254 dev veth-pc1 lladdr 1a:7f:ad:06:7d:92 STALE
```

`192.168.0.253` が `INCOMPLETE`（返事を待っている）か `FAILED` になっています。`pc1` は「`192.168.0.253` に渡そう」と ARP で問い合わせましたが、そんな機器はいないので返事がありません。**宛先は `pc2` なのに、ARP 表に出てくるのはゲートウェイの住所**です。

「町の外にだけ届かない」「ARP 表でゲートウェイの住所が `INCOMPLETE` / `FAILED`」——この2つがそろっていたら、デフォルトゲートウェイの設定を疑ってください。

直します。

```bash
sudo ip netns exec pc1 ip route replace default via 192.168.0.254
sudo ip netns exec pc1 ping -c 1 192.168.1.2
```

```text
PING 192.168.1.2 (192.168.1.2) 56(84) bytes of data.
64 bytes from 192.168.1.2: icmp_seq=1 ttl=63 time=0.017 ms

--- 192.168.1.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.017/0.017/0.017/0.000 ms
```

## 5-10. 片付け

```bash
sudo ip -all netns delete
```

## まとめ

- **ルータ**は、複数の町に住所を持ち、町と町の間で IP パケットを**転送**する機器
- 町の外への宛先は、**デフォルトゲートウェイ**（`ip route add default via ...`）に渡す。ゲートウェイには自分と同じ町の住所しか指定できない
- Linux は初期状態では転送しない（`net.ipv4.ip_forward = 0`）。**`ip_forward=1` にして初めてルータになる**
- 行きの道だけでは通信できない。**相手にも帰り道（デフォルトゲートウェイ）が要る**
- ルータを1台通るたびに **TTL が1減る**。スイッチでは減らない
- ルータを通っても **IP アドレスは変わらず、MAC アドレスはケーブルごとに付け替えられる**。町の外の相手の MAC アドレスは知る必要がない

この章のルータは、2つの町の両方に直接つながっていたので、どちらの町への道も最初から知っていました。では、ルータが2台になり、**直接つながっていない町**への道が必要になったらどうなるでしょうか。次の章では、ルーティングテーブルを自分で書き、`traceroute` で道筋をたどります。
