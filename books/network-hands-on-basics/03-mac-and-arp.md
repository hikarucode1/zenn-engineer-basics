---
title: "MAC アドレスと ARP"
---

## この章のゴール

第2章の 2-5 では、`ping` のデータより先に、次の問い合わせが流れていました。

```text
ARP, Request who-has 192.168.1.2 tell 192.168.0.1, length 28
```

`pc1` は相手の IP アドレス（`192.168.1.2`）を知っているのに、なぜわざわざ「`192.168.1.2` を持っている人は？」と聞いているのでしょうか。

この章では、ケーブルの上を流れるデータの「宛名」を `tcpdump` で直接見て、次の2つを確かめます。

- 同じネットワークの中で、データは **MAC アドレス** を宛先にして届けられる
- 送る前に、IP アドレスから MAC アドレスを調べる問い合わせが **ARP**

## 3-0. 準備

第2章と同じ2台を作ります。

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

この章では、まだ `ping` を打たないでください。最初の1発で起きることを観察したいからです。

## 3-1. MAC アドレス —— 口ごとの番号

第1章で、`ip link` の出力に `link/ether` という行があったのを覚えているでしょうか。

```bash
sudo ip netns exec pc1 ip link show veth-pc1
sudo ip netns exec pc2 ip link show veth-pc2
```

```text
4: veth-pc1@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 7a:87:09:62:f1:9c brd ff:ff:ff:ff:ff:ff link-netns pc2
3: veth-pc2@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether c6:f4:41:ba:5d:7b brd ff:ff:ff:ff:ff:ff link-netns pc1
```

`link/ether` の後ろの `7a:87:09:62:f1:9c` が、その口の **MAC アドレス**です。0 と 1 が 48 個並んだ番号を、16進数2桁ずつ `:` で区切って書いています。

| | IP アドレス | MAC アドレス |
|---|---|---|
| 誰が決めるか | 人が設定する（第1章で `ip addr add` したもの） | 口ごとに最初から付いている |
| 長さ | 32 個の 0/1 | 48 個の 0/1 |
| 例 | `192.168.0.1` | `7a:87:09:62:f1:9c` |
| 役割 | 最終的な宛先を表す | **同じケーブルの上で**、次に受け取る相手を表す |

本物の PC の LAN の口や Wi-Fi の口にも、工場出荷時に MAC アドレスが付いています。veth のような仮想の口では、作ったときに Linux が自動で値を決めます。本書と同じ値になるとは限りません。

`brd ff:ff:ff:ff:ff:ff` は、全部が 1 の特別な MAC アドレスで、**ブロードキャスト**（同じネットワークの全員宛て）を意味します。第2章の「ホスト部が全部 1 なら全員宛て」と同じ考え方です。

## 3-2. ケーブルの上を流れるのは「封筒」

`ping` のデータ（IP パケット）は、そのままケーブルに流れるわけではありません。**MAC アドレスを宛名に書いた封筒（イーサネットフレーム）** に入れて流されます。

| 宛先 MAC | 送信元 MAC | 中身の種類 | 中身（IP パケット） |
|---|---|---|---|
| `c6:f4:41:ba:5d:7b`（`pc2`） | `7a:87:09:62:f1:9c`（`pc1`） | IPv4 | `192.168.0.1` → `192.168.0.2` の `ping` のデータ |

ケーブルの先にいる機器は、まず封筒の**宛先 MAC** を見て、自分宛てかどうかを判断します。自分宛てでなければ、中身の IP パケットは開けずに捨てます。

つまり `pc1` が `pc2` に `ping` を送るには、`pc2` の IP アドレスだけでなく、**`pc2` の MAC アドレスも知っている必要があります**。でも `pc1` が知っているのは IP アドレスだけです。そこで、送る前に「この IP アドレスの MAC アドレスは何？」と問い合わせます。これが **ARP**（Address Resolution Protocol）です。

## 3-3. ARP 表 —— 調べた結果のメモ

ARP で調べた結果は、毎回問い合わせなくて済むように **ARP 表**（ARP キャッシュ）にメモされます。`ip neigh` で見られます（neigh は neighbor = 隣人の略）。

```bash
sudo ip netns exec pc1 ip neigh
```

何も表示されません。`pc1` はまだ誰とも通信していないので、隣人のメモが空っぽです。

## 3-4. 観察する —— 最初の1発で何が起きるか

第2章と同じく、ターミナルを2つ使います。

**ターミナル2**（`pc2` で待ち受ける）：

```bash
sudo ip netns exec pc2 tcpdump -n -e -i veth-pc2 arp or icmp
```

第2章との違いは **`-e`** です。これを付けると、封筒の宛名（MAC アドレス）も表示されます。

**ターミナル1**（`pc1` から `ping` を1発だけ打つ）：

```bash
sudo ip netns exec pc1 ping -c 1 192.168.0.2
```

ターミナル2には、次の4行が出ます（見やすいように、この章では行頭の時刻を省き、途中で改行を入れて載せます）。

```text
7a:87:09:62:f1:9c > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42:
    Request who-has 192.168.0.2 tell 192.168.0.1, length 28
c6:f4:41:ba:5d:7b > 7a:87:09:62:f1:9c, ethertype ARP (0x0806), length 42:
    Reply 192.168.0.2 is-at c6:f4:41:ba:5d:7b, length 28
7a:87:09:62:f1:9c > c6:f4:41:ba:5d:7b, ethertype IPv4 (0x0800), length 98:
    192.168.0.1 > 192.168.0.2: ICMP echo request, id 1636, seq 1, length 64
c6:f4:41:ba:5d:7b > 7a:87:09:62:f1:9c, ethertype IPv4 (0x0800), length 98:
    192.168.0.2 > 192.168.0.1: ICMP echo reply, id 1636, seq 1, length 64
```

各行の先頭の `A > B` は「送信元 MAC `A` から、宛先 MAC `B` へ」という意味です。1行ずつ読んでいきます。

| # | 封筒の宛名 | 中身 | 意味 |
|---|---|---|---|
| 1 | `pc1` → **`ff:ff:ff:ff:ff:ff`**（全員） | ARP Request | 「`192.168.0.2` を持っている人は、`192.168.0.1` に教えて」。**相手の MAC が分からないので全員宛て**に送る |
| 2 | `pc2` → `pc1` | ARP Reply | 「`192.168.0.2` は `c6:f4:41:ba:5d:7b` です」。問い合わせに自分の MAC が入っているので、`pc1` だけに返す |
| 3 | `pc1` → `pc2` | ICMP echo request | MAC が分かったので、やっと `ping` のデータを `pc2` 宛ての封筒で送る |
| 4 | `pc2` → `pc1` | ICMP echo reply | `ping` の返事 |

`ping` は1回しか打っていないのに、ケーブルの上では**4つのフレーム**が流れていました。最初の2つが ARP です。

確認したら、ターミナル2は `Ctrl+C` で止めておきます。

## 3-5. ARP 表に記録される

もう一度 `pc1` の ARP 表を見ます。

```bash
sudo ip netns exec pc1 ip neigh
```

```text
192.168.0.2 dev veth-pc1 lladdr c6:f4:41:ba:5d:7b REACHABLE
```

「`192.168.0.2` は `veth-pc1` の先にいて、MAC アドレス（`lladdr`）は `c6:f4:41:ba:5d:7b`」とメモされました。最後の `REACHABLE` は「ついさっき通信できたことを確認済み」という状態です。

`pc2` の ARP 表も見てみましょう。

```bash
sudo ip netns exec pc2 ip neigh
```

```text
192.168.0.1 dev veth-pc2 lladdr 7a:87:09:62:f1:9c DELAY
```

`pc2` は自分からは何も問い合わせていないのに、`pc1` の MAC アドレスを知っています。ARP の問い合わせには `pc1` 自身の IP アドレスと MAC アドレスが書かれているので、**問い合わせを受け取った側もついでにメモする**のです。末尾の状態（`DELAY` など）はタイミングによって変わります。次の 3-6 で説明します。

## 3-6. 2回目は ARP を省略する

ターミナル2で `tcpdump` をもう一度動かし、ターミナル1で `ping` をもう1発打ちます。

```bash
sudo ip netns exec pc2 tcpdump -n -e -i veth-pc2 arp or icmp
```

```bash
sudo ip netns exec pc1 ping -c 1 192.168.0.2
```

```text
7a:87:09:62:f1:9c > c6:f4:41:ba:5d:7b, ethertype IPv4 (0x0800), length 98:
    192.168.0.1 > 192.168.0.2: ICMP echo request, id 1647, seq 1, length 64
c6:f4:41:ba:5d:7b > 7a:87:09:62:f1:9c, ethertype IPv4 (0x0800), length 98:
    192.168.0.2 > 192.168.0.1: ICMP echo reply, id 1647, seq 1, length 64
```

今度はいきなり ICMP から始まっています。`pc1` は ARP 表のメモを使って、問い合わせを省略しました。

:::message
少し後に、次のような ARP が流れることもあります。

```text
c6:f4:41:ba:5d:7b > 7a:87:09:62:f1:9c, ethertype ARP (0x0806), length 42:
    Request who-has 192.168.0.1 tell 192.168.0.2, length 28
```

宛先が `ff:ff:ff:ff:ff:ff` ではなく `pc1` の MAC アドレスになっていることに注目してください。これは `pc2` が 3-5 でついでにメモした内容について、「このメモ、まだ合ってる？」と**本人に直接確かめている**ものです。3-5 で見た `DELAY` は、この確認を少し待っている状態です。
:::

### メモには有効期限がある

ARP 表のメモは、ずっと信じ続けるわけではありません。相手の PC が交換されて MAC アドレスが変わることもあるからです。1分ほど待ってから、もう一度見てみます。

```bash
sudo ip netns exec pc1 ip neigh
```

```text
192.168.0.2 dev veth-pc1 lladdr c6:f4:41:ba:5d:7b STALE
```

`REACHABLE` が **`STALE`**（古くなった）に変わりました。メモは残っていますが、「次に使うときは、念のため確かめ直そう」という状態です。主な状態をまとめておきます。

| 状態 | 意味 |
|---|---|
| `REACHABLE` | 最近、通信できることを確認した。そのまま使ってよい |
| `STALE` | しばらく確認していない。次に使うときに確かめ直す |
| `DELAY` / `PROBE` | 確かめ直している途中 |
| `FAILED` | 問い合わせたが、返事がなかった |
| `PERMANENT` | 人が手で書いたメモ。期限切れにならない |

## 3-7. 壊してみる —— 間違った MAC アドレスをメモしたら

ARP 表に**間違ったメモ**を、手で書き込んでみます。`pc2`（`192.168.0.2`）の MAC アドレスを、存在しない `02:00:00:00:00:99` と書き換えます。

```bash
sudo ip netns exec pc1 ip neigh replace 192.168.0.2 lladdr 02:00:00:00:00:99 dev veth-pc1 nud permanent
sudo ip netns exec pc1 ip neigh
```

```text
192.168.0.2 dev veth-pc1 lladdr 02:00:00:00:00:99 PERMANENT
```

`nud permanent` は「期限切れにしない」という指定です。こうしないと、Linux がすぐに確かめ直して正しい値に戻してしまいます。

ターミナル2で `tcpdump` を動かし、ターミナル1で `ping` を打ちます。

```bash
sudo ip netns exec pc2 tcpdump -n -e -i veth-pc2 arp or icmp
```

```bash
sudo ip netns exec pc1 ping -c 2 -W 1 192.168.0.2
```

ターミナル1：

```text
PING 192.168.0.2 (192.168.0.2) 56(84) bytes of data.

--- 192.168.0.2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1028ms

```

ターミナル2：

```text
7a:87:09:62:f1:9c > 02:00:00:00:00:99, ethertype IPv4 (0x0800), length 98:
    192.168.0.1 > 192.168.0.2: ICMP echo request, id 1669, seq 1, length 64
7a:87:09:62:f1:9c > 02:00:00:00:00:99, ethertype IPv4 (0x0800), length 98:
    192.168.0.1 > 192.168.0.2: ICMP echo request, id 1669, seq 2, length 64
```

ここがこの章でいちばん大事なところです。

- `ping` のデータは、**`pc2` のケーブルまで届いている**
- 中身の IP アドレスも `192.168.0.1 > 192.168.0.2` で、**正しく `pc2` 宛て**
- それなのに `pc2` は返事（`echo reply`）をしていない

`pc2` は封筒の宛先 MAC（`02:00:00:00:00:99`）を見て「自分宛てではない」と判断し、**中身を開けずに捨てた**のです。IP アドレスが正しくても、封筒の宛名が違えば受け取ってもらえません。

:::message
現実には、次のような場面で同じことが起きます。

- 相手の PC や LAN の口を交換して MAC アドレスが変わったのに、古いメモが残っている
- 同じ IP アドレスを2台に付けてしまい（**IP アドレスの重複**）、ARP の返事が2つ返ってきて、メモがどちらかに定まらない
- 悪意のある第三者が偽の ARP の返事を送り、メモを自分の MAC アドレスに書き換える（**ARP スプーフィング**）。通信を盗み見る攻撃の入口になる
:::

## 3-8. 直す —— メモを消す

間違ったメモを消します。

```bash
sudo ip netns exec pc1 ip neigh del 192.168.0.2 dev veth-pc1
sudo ip netns exec pc1 ping -c 2 192.168.0.2
```

```text
PING 192.168.0.2 (192.168.0.2) 56(84) bytes of data.
64 bytes from 192.168.0.2: icmp_seq=1 ttl=64 time=0.021 ms
64 bytes from 192.168.0.2: icmp_seq=2 ttl=64 time=0.028 ms

--- 192.168.0.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1040ms
rtt min/avg/max/mdev = 0.021/0.024/0.028/0.003 ms
```

```bash
sudo ip netns exec pc1 ip neigh
```

```text
192.168.0.2 dev veth-pc1 lladdr c6:f4:41:ba:5d:7b REACHABLE
```

メモが消えたので、`pc1` はもう一度 ARP で問い合わせて、正しい MAC アドレスをメモし直しました。

## 3-9. 第2章の謎を解く

最後に、第2章の 2-5 を ARP 表の側から見直します。`pc1` を `/16`、`pc2` を `192.168.1.2/24` にした、「片方だけ町が広い」状態を作ります。

```bash
sudo ip netns exec pc1 ip addr del 192.168.0.1/24 dev veth-pc1
sudo ip netns exec pc1 ip addr add 192.168.0.1/16 dev veth-pc1
sudo ip netns exec pc2 ip addr del 192.168.0.2/24 dev veth-pc2
sudo ip netns exec pc2 ip addr add 192.168.1.2/24 dev veth-pc2
sudo ip netns exec pc1 ping -c 2 -W 1 192.168.1.2
sudo ip netns exec pc1 ip neigh
```

```text
192.168.1.2 dev veth-pc1 FAILED
```

`FAILED` ——問い合わせたが返事がなかった、という記録です。第2章の症状を、この章の言葉で言い直すとこうなります。

1. `pc1` は `192.168.1.2` を同じ町だと思っているので、直接送ろうとする
2. 封筒の宛名にする MAC アドレスを知らないので、ARP で全員に問い合わせる
3. `pc2` は問い合わせを受け取るが、`192.168.0.1` への帰り道がないので返事をしない
4. `pc1` は MAC アドレスが分からないまま、`ping` のデータを**封筒に入れることすらできず**に終わる

`100% packet loss` という同じ症状でも、3-7 は「データは届いたが捨てられた」、3-9 は「データを送り出すことすらできなかった」と、中で起きていることがまったく違います。**`tcpdump` と `ip neigh` を見れば、この2つを区別できます**。

## 3-10. 片付け

```bash
sudo ip -all netns delete
```

## まとめ

- **MAC アドレス**は口ごとの番号。同じケーブルの上では、データは MAC アドレスを宛名にした**封筒（イーサネットフレーム）** に入れて届けられる
- 受け取った側は封筒の**宛先 MAC** を見て、自分宛てでなければ中身を開けずに捨てる
- 送る前に「この IP アドレスの MAC アドレスは？」を**全員宛て（`ff:ff:ff:ff:ff:ff`）** に問い合わせるのが **ARP**
- 調べた結果は **ARP 表**（`ip neigh`）にメモされ、2回目からは問い合わせを省略する。メモには有効期限がある
- `tcpdump -e` で封筒の宛名が見える。`100% packet loss` のときは、**データが届いて捨てられたのか、送り出せなかったのか**を `tcpdump` と `ip neigh` で切り分ける

ここまでは、2台をケーブル1本で直接つないでいました。でも、ARP の問い合わせは「全員宛て」でした。3台、4台とつなぐには、どうすればいいのでしょうか。次の章では**スイッチ**を作り、全員宛ての問い合わせが実際に全員に届く様子と、スイッチが MAC アドレスを覚えていく様子を見ていきます。
