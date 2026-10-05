---
title: "ルーティングテーブル"
---

## この章のゴール

第5章のルータは、2つの町の両方に直接つながっていたので、何も教えなくても両方の町への道を知っていました。

この章ではルータを2台にして、**ルータが直接つながっていない町**を作ります。そこへの道を、ルーティングテーブルに自分で書き込みます（**静的ルート**）。そのうえで次のことを確かめます。

- 道を知らないルータは、**「その町は知らない」と送り主に知らせてくる**
- 行きの道と帰りの道は、**通り道のルータ全部**にそれぞれ必要
- **TTL** を使うと、途中のルータを1台ずつ確かめられる。これが `traceroute` の仕組み
- 道案内を間違えると、パケットがルータの間を**ぐるぐる回る**（ルーティングループ）。TTL はそれを止めるためにある

## 6-1. 作るもの

```text
  192.168.0.0/24 の町          10.0.0.0/30 の町          192.168.2.0/24 の町

 pc1                  r1                       r2                  pc2
 veth-pc1 ===== r1-eth0  r1-eth1 ======= r2-eth0  r2-eth1 ===== veth-pc2
 .0.1           .0.254   10.0.0.1        10.0.0.2  .2.254          .2.2
```

町が3つになりました。真ん中の `10.0.0.0/30` は、**ルータ同士をつなぐためだけの町**です。第2章で見たとおり、`/30` は機器をちょうど2台だけ置ける小さな町で、こういう使い方によく出てきます。

ポイントは、**`r1` は `192.168.2.0/24` に、`r2` は `192.168.0.0/24` に直接つながっていない**ことです。

## 6-2. 組み立てる

4台の箱を作ります。

```bash
sudo ip netns add pc1
sudo ip netns add pc2
sudo ip netns add r1
sudo ip netns add r2
```

ケーブルを3本作って挿します。

```bash
sudo ip link add veth-pc1 type veth peer name r1-eth0
sudo ip link add r1-eth1 type veth peer name r2-eth0
sudo ip link add r2-eth1 type veth peer name veth-pc2
sudo ip link set veth-pc1 netns pc1
sudo ip link set r1-eth0 netns r1
sudo ip link set r1-eth1 netns r1
sudo ip link set r2-eth0 netns r2
sudo ip link set r2-eth1 netns r2
sudo ip link set veth-pc2 netns pc2
```

住所を付けて `up` にします。

```bash
sudo ip netns exec pc1 ip addr add 192.168.0.1/24 dev veth-pc1
sudo ip netns exec r1 ip addr add 192.168.0.254/24 dev r1-eth0
sudo ip netns exec r1 ip addr add 10.0.0.1/30 dev r1-eth1
sudo ip netns exec r2 ip addr add 10.0.0.2/30 dev r2-eth0
sudo ip netns exec r2 ip addr add 192.168.2.254/24 dev r2-eth1
sudo ip netns exec pc2 ip addr add 192.168.2.2/24 dev veth-pc2
sudo ip netns exec pc1 ip link set veth-pc1 up
sudo ip netns exec r1 ip link set r1-eth0 up
sudo ip netns exec r1 ip link set r1-eth1 up
sudo ip netns exec r2 ip link set r2-eth0 up
sudo ip netns exec r2 ip link set r2-eth1 up
sudo ip netns exec pc2 ip link set veth-pc2 up
```

第5章で学んだ「ルータにする設定」と「デフォルトゲートウェイ」も、最初から入れておきます。

```bash
sudo ip netns exec r1 sysctl -w net.ipv4.ip_forward=1
sudo ip netns exec r2 sysctl -w net.ipv4.ip_forward=1
sudo ip netns exec pc1 ip route add default via 192.168.0.254
sudo ip netns exec pc2 ip route add default via 192.168.2.254
```

## 6-3. 2台のルータの道案内を比べる

```bash
sudo ip netns exec r1 ip route
```

```text
10.0.0.0/30 dev r1-eth1 proto kernel scope link src 10.0.0.1
192.168.0.0/24 dev r1-eth0 proto kernel scope link src 192.168.0.254
```

```bash
sudo ip netns exec r2 ip route
```

```text
10.0.0.0/30 dev r2-eth0 proto kernel scope link src 10.0.0.2
192.168.2.0/24 dev r2-eth1 proto kernel scope link src 192.168.2.254
```

それぞれのルータは、**自分が直接つながっている2つの町**への道しか知りません。`r1` の表に `192.168.2.0/24` は無く、`r2` の表に `192.168.0.0/24` は無い。人間から見れば「`r1` の隣の `r2` の先に `192.168.2.0/24` がある」のは明らかですが、`r1` はそれを知る方法を持っていません。

## 6-4. 壊れている① —— ルータが「知らない」と言う

`pc1` から `pc2` に `ping` を打ちます。

```bash
sudo ip netns exec pc1 ping -c 2 -W 1 192.168.2.2
```

```text
PING 192.168.2.2 (192.168.2.2) 56(84) bytes of data.
From 192.168.0.254 icmp_seq=1 Destination Net Unreachable
From 192.168.0.254 icmp_seq=2 Destination Net Unreachable

--- 192.168.2.2 ping statistics ---
2 packets transmitted, 0 received, +2 errors, 100% packet loss, time 1007ms

```

新しい症状です。**`From 192.168.0.254 ... Destination Net Unreachable`** は、「`192.168.0.254`（`r1`）から、『宛先の町に届けられない』という知らせが来た」という意味です。

| 症状 | 誰が言っているか | 意味 |
|---|---|---|
| `ping: connect: Network is unreachable` | 自分（`pc1`） | 自分の表に道がない（第1・2章） |
| `From <ルータ> ... Destination Net Unreachable` | 途中のルータ | **そのルータ**の表に道がない |
| `100% packet loss`（それ以外は何も出ない） | 誰も言っていない | どこかで黙って捨てられている |

ルータは、道が分からないパケットを捨てるとき、送り主に **ICMP** というメッセージで理由を知らせます。`ping` 自体も ICMP の一種（echo request / echo reply）ですが、ICMP にはこうした「届けられなかった理由」を伝える種類もあるのです。

知らせてきたルータの表を確認します。

```bash
sudo ip netns exec r1 ip route get 192.168.2.2
```

```text
RTNETLINK answers: Network is unreachable
```

予想どおり、`r1` は `192.168.2.2` への道を知りません。

## 6-5. 静的ルート —— 道を書き込む

`r1` に「`192.168.2.0/24` へは、`10.0.0.2`（`r2`）に渡せ」と教えます。

```bash
sudo ip netns exec r1 ip route add 192.168.2.0/24 via 10.0.0.2
sudo ip netns exec r1 ip route
```

```text
10.0.0.0/30 dev r1-eth1 proto kernel scope link src 10.0.0.1
192.168.0.0/24 dev r1-eth0 proto kernel scope link src 192.168.0.254
192.168.2.0/24 via 10.0.0.2 dev r1-eth1
```

3行目が増えました。第5章のデフォルトゲートウェイ（`default via ...`）は「どの行にも当てはまらない宛先」の道でしたが、こちらは **`192.168.2.0/24` という特定の町への道**です。人が手で書いた道なので、**静的ルート**と呼びます。

`via` に書けるのは、デフォルトゲートウェイと同じく**自分が直接つながっている町の住所**だけです。`r1` は `10.0.0.0/30` の町にいるので、同じ町の `10.0.0.2` を指定できます。

## 6-6. 壊れている② —— 帰り道は、通り道のルータ全部に要る

```bash
sudo ip netns exec pc1 ping -c 2 -W 1 192.168.2.2
```

```text
PING 192.168.2.2 (192.168.2.2) 56(84) bytes of data.

--- 192.168.2.2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1001ms

```

`Destination Net Unreachable` は消えましたが、今度は**何の知らせもなく**届きません。黙って捨てられているので、`tcpdump` で「どこまで届いているか」を順に調べます。

**ターミナル2**（`r2` の `r1` 側の口）：

```bash
sudo ip netns exec r2 tcpdump -n -i r2-eth0 icmp
```

**ターミナル1**：

```bash
sudo ip netns exec pc1 ping -c 1 -W 1 192.168.2.2
```

```text
IP 192.168.0.1 > 192.168.2.2: ICMP echo request, id 2072, seq 1, length 64
```

`r2` までは届いています。次に、ターミナル2を止めて、`pc2` で待ち受けます。

```bash
sudo ip netns exec pc2 tcpdump -n -i veth-pc2 icmp
```

```bash
sudo ip netns exec pc1 ping -c 1 -W 1 192.168.2.2
```

今度は何も出ません。**`r2` で止まっています**。`r2` に、送り主 `192.168.0.1` への道を聞いてみます。

```bash
sudo ip netns exec r2 ip route get 192.168.0.1
```

```text
RTNETLINK answers: Network is unreachable
```

`r2` は `192.168.0.0/24` への道を知りません。つまり、仮に `pc2` が返事をしても、`r2` はそれを `pc1` に届けられません。

第5章では、帰り道を `pc2` のデフォルトゲートウェイに教えれば済みました。ルータが2台になると、**`pc2` → `r2` → `r1` → `pc1` の帰り道を、通り道のルータ全部が知っている必要があります**。`r1` に行きの道を書いただけでは足りないのです。

:::message
`pc2` まで届かずに `r2` で捨てられているのは、Ubuntu の初期設定にある**逆経路フィルタ**（`rp_filter`）のためです。ルータは「送り主への帰り道が無いパケット」を、送り主の住所が偽物かもしれない怪しいパケットとみなして捨てます。帰り道が無ければどうせ返事は届かないので、結果は同じです。
:::

`r2` に帰り道を書き込みます。

```bash
sudo ip netns exec r2 ip route add 192.168.0.0/24 via 10.0.0.1
```

## 6-7. 通った

```bash
sudo ip netns exec pc1 ping -c 3 192.168.2.2
```

```text
PING 192.168.2.2 (192.168.2.2) 56(84) bytes of data.
64 bytes from 192.168.2.2: icmp_seq=1 ttl=62 time=0.033 ms
64 bytes from 192.168.2.2: icmp_seq=2 ttl=62 time=0.037 ms
64 bytes from 192.168.2.2: icmp_seq=3 ttl=62 time=0.042 ms

--- 192.168.2.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2040ms
rtt min/avg/max/mdev = 0.033/0.037/0.042/0.004 ms
```

`ttl=62` です。`pc2` が 64 で送り出した返事が、ルータを2台通って 62 になりました。

ここまでに書いた道を表にしておきます。

| 機器 | 行き（→ `192.168.2.0/24`） | 帰り（→ `192.168.0.0/24`） |
|---|---|---|
| `pc1` | デフォルトゲートウェイ `192.168.0.254`（`r1`） | 直接つながっている |
| `r1` | **静的ルート** `via 10.0.0.2`（`r2`） | 直接つながっている |
| `r2` | 直接つながっている | **静的ルート** `via 10.0.0.1`（`r1`） |
| `pc2` | 直接つながっている | デフォルトゲートウェイ `192.168.2.254`（`r2`） |

**どの行も「次に誰に渡すか」だけ**を書いていることに注目してください。`pc1` は `r2` の存在を知りませんし、`r1` は `pc2` の MAC アドレスを知りません。各機器が「次の1台」だけを知っていて、それをリレーすることで遠くまで届くのが、IP の基本的な仕組みです。

## 6-8. TTL で途中のルータを1台ずつ確かめる

`ping` の `-t` で、送り出す IP パケットの TTL を指定できます。TTL を 1 にして送ってみます。

```bash
sudo ip netns exec pc1 ping -c 1 -t 1 192.168.2.2
```

```text
PING 192.168.2.2 (192.168.2.2) 56(84) bytes of data.
From 192.168.0.254 icmp_seq=1 Time to live exceeded

--- 192.168.2.2 ping statistics ---
1 packets transmitted, 0 received, +1 errors, 100% packet loss, time 0ms

```

`From 192.168.0.254 ... Time to live exceeded`（寿命切れ）です。ルータは転送するときに TTL を1減らし、**0 になったらそのパケットを捨てて、送り主に「寿命が切れた」と知らせます**。TTL 1 のパケットは、最初のルータ `r1` で 0 になりました。

TTL を 2、3 と増やしていきます。

```bash
sudo ip netns exec pc1 ping -c 1 -t 2 192.168.2.2
```

```text
PING 192.168.2.2 (192.168.2.2) 56(84) bytes of data.
From 10.0.0.2 icmp_seq=1 Time to live exceeded

--- 192.168.2.2 ping statistics ---
1 packets transmitted, 0 received, +1 errors, 100% packet loss, time 0ms

```

```bash
sudo ip netns exec pc1 ping -c 1 -t 3 192.168.2.2
```

```text
PING 192.168.2.2 (192.168.2.2) 56(84) bytes of data.
64 bytes from 192.168.2.2: icmp_seq=1 ttl=62 time=0.012 ms

--- 192.168.2.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.012/0.012/0.012/0.000 ms
```

| 送った TTL | 寿命が切れた場所 | 分かること |
|---|---|---|
| 1 | `192.168.0.254`（`r1`） | 1台目のルータは `r1` |
| 2 | `10.0.0.2`（`r2`） | 2台目のルータは `r2` |
| 3 | 切れずに `pc2` に届いた | `pc2` はルータ2台の先にいる |

TTL を1ずつ増やすだけで、**道の途中にいるルータを順番に割り出せました**。

## 6-9. traceroute —— 6-8 を自動でやるコマンド

6-8 の手順を自動でやってくれるのが `traceroute` です。入れておきます。

```bash
sudo apt install -y traceroute
```

```bash
sudo ip netns exec pc1 traceroute -n 192.168.2.2
```

```text
traceroute to 192.168.2.2 (192.168.2.2), 30 hops max, 60 byte packets
 1  192.168.0.254  0.008 ms  0.004 ms  0.006 ms
 2  10.0.0.2  0.006 ms  0.005 ms  0.005 ms
 3  192.168.2.2  0.008 ms  0.006 ms  0.006 ms
```

左の数字が TTL、その右が「寿命切れを知らせてきた機器」です。各行に時間が3つあるのは、同じ TTL で3回ずつ試しているからです。`-n` は住所を名前に変換しない指定です。

裏で何が起きているかを `pc1` で見てみます。`-q 1` は「各 TTL で1回だけ試す」という指定です。

**ターミナル2**：

```bash
sudo ip netns exec pc1 tcpdump -n -i veth-pc1 icmp
```

**ターミナル1**：

```bash
sudo ip netns exec pc1 traceroute -n -q 1 192.168.2.2
```

ターミナル2（時刻は省いています）：

```text
IP 192.168.0.254 > 192.168.0.1: ICMP time exceeded in-transit, length 68
IP 10.0.0.2 > 192.168.0.1: ICMP time exceeded in-transit, length 68
IP 192.168.2.2 > 192.168.0.1: ICMP 192.168.2.2 udp port 33436 unreachable, length 68
IP 192.168.2.2 > 192.168.0.1: ICMP 192.168.2.2 udp port 33437 unreachable, length 68
IP 192.168.2.2 > 192.168.0.1: ICMP 192.168.2.2 udp port 33438 unreachable, length 68
IP 192.168.2.2 > 192.168.0.1: ICMP 192.168.2.2 udp port 33439 unreachable, length 68
IP 192.168.2.2 > 192.168.0.1: ICMP 192.168.2.2 udp port 33440 unreachable, length 68
```

1・2行目が、6-8 で見た「寿命切れ」の知らせ（`time exceeded`）です。3行目からは種類が違い、`udp port ... unreachable` になっています。`traceroute` は `ping` ではなく **UDP** という種類の通信で試していて、最後に `pc2` 本人から「その**ポート**は使われていない」と返ってきたところで「到着した」と判断しています。UDP とポートは、次の第7章で扱います。

`udp port ... unreachable` が何行も出ているのは、`traceroute` が時間を節約するために、TTL 1・2・3・4……のパケットを**まとめて送り出している**からです。TTL 3 以上のパケットはどれも `pc2` まで届くので、それぞれに返事が来ます。行数は環境によって変わります。

:::message
インターネット上の `traceroute` で `* * *` と表示される行は、そのルータが「寿命切れ」の知らせを返さない設定になっていることを表します。そのルータで止まっているとは限りません。
:::

## 6-10. 壊してみる —— ルーティングループ

最後に、道案内の間違いでパケットがぐるぐる回る様子を見ます。

今、`pc1` から、どの町にも属さない `8.8.8.8` に `ping` を打つとこうなります。

```bash
sudo ip netns exec pc1 ping -c 1 -W 1 8.8.8.8
```

```text
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
From 192.168.0.254 icmp_seq=1 Destination Net Unreachable

--- 8.8.8.8 ping statistics ---
1 packets transmitted, 0 received, +1 errors, 100% packet loss, time 0ms

```

`r1` が「知らない」と正しく返しています。ここで、**`r1` と `r2` の両方に、相手をデフォルトゲートウェイとして設定**してしまいます。「知らない宛先は `r2` に任せよう」「知らない宛先は `r1` に任せよう」と、お互いに相手が知っていると思い込んでいる状態です。

```bash
sudo ip netns exec r1 ip route add default via 10.0.0.2
sudo ip netns exec r2 ip route add default via 10.0.0.1
sudo ip netns exec pc1 ping -c 2 -W 1 8.8.8.8
```

```text
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
From 10.0.0.2 icmp_seq=1 Time to live exceeded
From 10.0.0.2 icmp_seq=2 Time to live exceeded

--- 8.8.8.8 ping statistics ---
2 packets transmitted, 0 received, +2 errors, 100% packet loss, time 1043ms

```

宛先は `8.8.8.8` なのに、`Time to live exceeded`（寿命切れ）が返ってきました。2台のルータの間で何が起きているかを、`-v`（詳しく表示）を付けた `tcpdump` で見てみます。

**ターミナル2**：

```bash
sudo ip netns exec r1 tcpdump -n -v -i r1-eth1 icmp
```

**ターミナル1**：

```bash
sudo ip netns exec pc1 ping -c 1 -W 1 8.8.8.8
```

ターミナル2（時刻は省き、途中を省略しています）：

```text
IP (tos 0x0, ttl 63, id 16934, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.0.1 > 8.8.8.8: ICMP echo request, id 1857, seq 1, length 64
IP (tos 0x0, ttl 62, id 16934, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.0.1 > 8.8.8.8: ICMP echo request, id 1857, seq 1, length 64
IP (tos 0x0, ttl 61, id 16934, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.0.1 > 8.8.8.8: ICMP echo request, id 1857, seq 1, length 64

（中略：ttl が 1 ずつ減りながら、同じパケットが何度も通る）

IP (tos 0x0, ttl 2, id 16934, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.0.1 > 8.8.8.8: ICMP echo request, id 1857, seq 1, length 64
IP (tos 0x0, ttl 1, id 16934, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.0.1 > 8.8.8.8: ICMP echo request, id 1857, seq 1, length 64
IP (tos 0xc0, ttl 64, id 38132, offset 0, flags [none], proto ICMP (1), length 112)
    10.0.0.2 > 192.168.0.1: ICMP time exceeded in-transit, length 92
```

`ping` は1回しか打っていないのに、`r1` と `r2` をつなぐケーブルを、**同じパケット**（`id 16934`）が `ttl 63` から `ttl 1` まで何十回も行き来しています。`r1` は `r2` に渡し、`r2` は `r1` に返し、また `r1` が `r2` に渡す……。最後に `r2` で TTL が 0 になり、ようやく捨てられて `pc1` に寿命切れが知らされました。

`traceroute` で見ると、ループがはっきり分かります。`-m 8` は「TTL 8 まで試す」という指定です。

```bash
sudo ip netns exec pc1 traceroute -n -m 8 8.8.8.8
```

```text
traceroute to 8.8.8.8 (8.8.8.8), 8 hops max, 60 byte packets
 1  192.168.0.254  0.016 ms  0.005 ms  0.004 ms
 2  10.0.0.2  0.009 ms  0.006 ms  0.006 ms
 3  192.168.0.254  0.005 ms  0.006 ms  0.005 ms
 4  10.0.0.2  0.008 ms  0.007 ms  0.007 ms
 5  * * *
 6  * * *
 7  * * *
 8  * * *
```

`192.168.0.254`（`r1`）と `10.0.0.2`（`r2`）が交互に出てきます。**`traceroute` で同じ住所が繰り返し出てきたら、ルーティングループ**です。5行目以降の `* * *` は、Linux が「寿命切れ」の知らせを短時間に送りすぎないよう、数を制限しているためです。どの行から `*` になるかは、実行するたびに変わります。

もし TTL が無かったら、このパケットは永遠に回り続け、ループするパケットが増えるたびにケーブルが埋まっていきます。**TTL は、こういう事故が起きてもネットワーク全体が止まらないようにするための安全装置**です。

### 直す

ループの原因は、`r2` の「知らない宛先は `r1` に任せる」です。`r1` も知らないので、どちらかが「知らない」と言わなければいけません。`r2` のデフォルトゲートウェイを消します。

```bash
sudo ip netns exec r2 ip route del default
sudo ip netns exec pc1 ping -c 1 -W 1 8.8.8.8
```

```text
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
From 10.0.0.2 icmp_seq=1 Destination Net Unreachable

--- 8.8.8.8 ping statistics ---
1 packets transmitted, 0 received, +1 errors, 100% packet loss, time 0ms

```

今度は `r2` が「知らない」（`Destination Net Unreachable`）と正しく返すようになりました。

## 6-11. 練習問題

**問1.** `r2` に3つ目の口 `r2-eth2` を付け、`192.168.3.0/24` の町（`r2` の住所は `192.168.3.254`）と、そこにいる `pc3`（`192.168.3.3`）をつなぎました。`pc1` と `pc3` が `ping` し合えるようにするには、6-7 までの状態に加えて、どの機器にどんな道を書けばいいですか？

:::details 答え
- **`r1`**：`192.168.3.0/24` への静的ルート（`sudo ip netns exec r1 ip route add 192.168.3.0/24 via 10.0.0.2`）
- **`pc3`**：デフォルトゲートウェイ（`sudo ip netns exec pc3 ip route add default via 192.168.3.254`）

`r2` は `192.168.3.0/24` に直接つながり、`192.168.0.0/24` への道（6-6 で書いたもの）も知っているので、追加は要りません。「通り道のルータ全部が、行きと帰りの両方を知っているか」を1台ずつ確かめるのがコツです。
:::

**問2.** `ping` を打ったら `From 10.0.0.2 icmp_seq=1 Destination Net Unreachable` と出ました。最初にどの機器の何を見ますか？

:::details 答え
**`10.0.0.2` の機器（`r2`）のルーティングテーブル**です。`From` の後ろの住所が「届けられない」と言ってきたルータなので、そのルータに宛先への道があるかを `ip route` や `ip route get <宛先>` で確かめます。
:::

## 6-12. 片付け

```bash
sudo ip -all netns delete
```

## まとめ

- ルータは、**直接つながっている町への道しか自動では知らない**。それ以外の町への道は**静的ルート**（`ip route add <町> via <次のルータ>`）で教える
- ルーティングテーブルの各行は「**次に誰に渡すか**」だけを書く。各機器が次の1台を知っていれば、リレーで遠くまで届く
- 行きの道と帰りの道は、**通り道のルータ全部**にそれぞれ必要
- 道を知らないルータは **`Destination Net Unreachable`** を返してくる。`From` の後ろが、道を知らないルータ
- ルータは TTL を1減らし、0 になったら **`Time to live exceeded`** を返す。TTL を1ずつ増やして途中のルータを割り出すのが **`traceroute`**
- 道案内が互いを指し合うと**ルーティングループ**になる。`traceroute` で同じ住所が繰り返し出たら疑う。TTL はループでネットワークが止まらないための安全装置

6-9 で、`traceroute` の最後に `udp port 33436 unreachable` という知らせが返ってきました。ここまでは「どの機器に届けるか」を見てきましたが、1台の機器の中では、たくさんのプログラムが通信しています。届いたデータを**どのプログラムに渡すか**を決めるのが**ポート番号**です。次の章では、UDP とポートを見ていきます。
