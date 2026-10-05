---
title: "スイッチで3台以上つなぐ"
---

## この章のゴール

ここまでは、2台を veth 1本で直接つないでいました。でも veth は**両端が1つずつしかないケーブル**なので、3台目をつなぐ場所がありません。

現実のネットワークでは、PC を**スイッチ**（スイッチングハブ）につなぎます。この章では、Linux の **bridge** という機能でスイッチを作り、3台の PC をつなぎます。そのうえで、次の2つを確かめます。

- 第3章で見た**全員宛て（`ff:ff:ff:ff:ff:ff`）の ARP は、本当に全員に届く**
- 一方で、**個別宛てのデータは、宛先の PC にしか届かない**。スイッチは、どの口の先にどの MAC アドレスがいるかを覚えている

## 4-1. 作るもの

```text
                +------------- sw1 -------------+
                |              br0              |
                |   sw1-p1    sw1-p2    sw1-p3  |
                +-----+---------+---------+-----+
                      |         |         |
               veth-pc1  veth-pc2  veth-pc3
                  pc1       pc2       pc3
          192.168.0.1  192.168.0.2  192.168.0.3
```

- **`sw1`**：スイッチ用の netns。PC と同じく netns で「箱」を作り、その中にスイッチの部品を置きます
- **`br0`**：スイッチ本体（bridge）
- **`sw1-p1`〜`sw1-p3`**：スイッチの差し込み口（**ポート**）。それぞれ veth で PC とつながります

## 4-2. スイッチを作る

スイッチ用の箱 `sw1` を作り、その中に bridge `br0` を作ります。

```bash
sudo ip netns add sw1
sudo ip netns exec sw1 ip link add br0 type bridge
```

`type bridge` は「複数の口を束ねて、口同士でフレームを中継する装置」を作る指定です。第1章で `type veth` でケーブルを作ったのと同じ書き方です。

## 4-3. PC を3台つなぐ

`pc1` を作ってスイッチにつなぎます。第1章とほとんど同じですが、veth の反対側を**別の PC ではなくスイッチの箱に挿す**ところが違います。

```bash
sudo ip netns add pc1
sudo ip link add veth-pc1 type veth peer name sw1-p1
sudo ip link set veth-pc1 netns pc1
sudo ip link set sw1-p1 netns sw1
sudo ip netns exec sw1 ip link set sw1-p1 master br0
sudo ip netns exec sw1 ip link set sw1-p1 up
sudo ip netns exec pc1 ip addr add 192.168.0.1/24 dev veth-pc1
sudo ip netns exec pc1 ip link set veth-pc1 up
```

新しいのは5行目です。`master br0` は「`sw1-p1` を `br0` の口の1つにする」という意味で、**スイッチのポートに LAN ケーブルを挿す**操作にあたります。

`pc2` と `pc3` も同じようにつなぎます。

```bash
sudo ip netns add pc2
sudo ip link add veth-pc2 type veth peer name sw1-p2
sudo ip link set veth-pc2 netns pc2
sudo ip link set sw1-p2 netns sw1
sudo ip netns exec sw1 ip link set sw1-p2 master br0
sudo ip netns exec sw1 ip link set sw1-p2 up
sudo ip netns exec pc2 ip addr add 192.168.0.2/24 dev veth-pc2
sudo ip netns exec pc2 ip link set veth-pc2 up
```

```bash
sudo ip netns add pc3
sudo ip link add veth-pc3 type veth peer name sw1-p3
sudo ip link set veth-pc3 netns pc3
sudo ip link set sw1-p3 netns sw1
sudo ip netns exec sw1 ip link set sw1-p3 master br0
sudo ip netns exec sw1 ip link set sw1-p3 up
sudo ip netns exec pc3 ip addr add 192.168.0.3/24 dev veth-pc3
sudo ip netns exec pc3 ip link set veth-pc3 up
```

`br0` に3つのポートがつながったことを確認します。

```bash
sudo ip netns exec sw1 ip link show master br0
```

```text
3: sw1-p1@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UP mode DEFAULT group default qlen 1000
    link/ether 1e:73:af:e5:ee:7f brd ff:ff:ff:ff:ff:ff link-netns pc1
5: sw1-p2@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UP mode DEFAULT group default qlen 1000
    link/ether 16:40:e0:9c:6d:0a brd ff:ff:ff:ff:ff:ff link-netns pc2
7: sw1-p3@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UP mode DEFAULT group default qlen 1000
    link/ether ce:c5:61:58:84:01 brd ff:ff:ff:ff:ff:ff link-netns pc3
```

3つとも `master br0` で、`UP,LOWER_UP`（自分も相手も生きている）です。ケーブルは全部つながりました。

## 4-4. まず失敗させてみる

`pc1` から `pc2` に `ping` を打ちます。

```bash
sudo ip netns exec pc1 ping -c 2 -W 1 192.168.0.2
```

```text
PING 192.168.0.2 (192.168.0.2) 56(84) bytes of data.

--- 192.168.0.2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1016ms

```

通りません。第3章で学んだとおり、まず ARP 表を見てみます。

```bash
sudo ip netns exec pc1 ip neigh
```

```text
192.168.0.2 dev veth-pc1 FAILED
```

第3章で見た **`FAILED`**（問い合わせたが、返事がなかった）です。`ping` の直後に見ると、**`INCOMPLETE`**（問い合わせ中で、まだ返事がない）と表示されることもあります。どちらにしても、`pc1` の問い合わせが `pc2` まで届いていないか、`pc2` の返事が戻ってきていないことになります。

ポートは全部 `UP,LOWER_UP` でした。では、スイッチ本体はどうでしょうか。

```bash
sudo ip netns exec sw1 ip link show br0
```

```text
2: br0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 16:40:e0:9c:6d:0a brd ff:ff:ff:ff:ff:ff
```

**`state DOWN`** ——スイッチ本体の電源が入っていませんでした。ポートの状態は `bridge link` で見られます。

```bash
sudo ip netns exec sw1 bridge link
```

```text
3: sw1-p1@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master br0 state disabled priority 32 cost 2
5: sw1-p2@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master br0 state disabled priority 32 cost 2
7: sw1-p3@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master br0 state disabled priority 32 cost 2
```

どのポートも **`state disabled`**（中継しない）です。ケーブルが全部つながっていて、ポートの1つ1つは生きていても、**本体が動いていなければフレームは中継されません**。

## 4-5. 直す —— スイッチの電源を入れる

```bash
sudo ip netns exec sw1 ip link set br0 up
sudo ip netns exec sw1 bridge link
```

```text
3: sw1-p1@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master br0 state forwarding priority 32 cost 2
5: sw1-p2@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master br0 state forwarding priority 32 cost 2
7: sw1-p3@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master br0 state forwarding priority 32 cost 2
```

ポートが **`state forwarding`**（中継する）に変わりました。`ping` を打ちます。

```bash
sudo ip netns exec pc1 ping -c 2 192.168.0.2
sudo ip netns exec pc1 ping -c 2 192.168.0.3
```

```text
PING 192.168.0.2 (192.168.0.2) 56(84) bytes of data.
64 bytes from 192.168.0.2: icmp_seq=1 ttl=64 time=0.017 ms
64 bytes from 192.168.0.2: icmp_seq=2 ttl=64 time=0.037 ms

--- 192.168.0.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1046ms
rtt min/avg/max/mdev = 0.017/0.027/0.037/0.010 ms
PING 192.168.0.3 (192.168.0.3) 56(84) bytes of data.
64 bytes from 192.168.0.3: icmp_seq=1 ttl=64 time=0.031 ms
64 bytes from 192.168.0.3: icmp_seq=2 ttl=64 time=0.039 ms

--- 192.168.0.3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1017ms
rtt min/avg/max/mdev = 0.031/0.035/0.039/0.004 ms
```

`pc1` から `pc2` にも `pc3` にも届きました。3台のネットワークの完成です。

:::message
`ttl=64` に注目してください。第1章で2台を直接つないだときと同じ値です。スイッチを通っても TTL は減りません。スイッチは封筒（フレーム）を中継するだけで、中身の IP パケットには手を付けないからです。TTL が減る様子は、ルータを作る第5章以降で見ます。
:::

## 4-6. スイッチの記憶 —— MAC アドレス表

スイッチは、**どのポートの先に、どの MAC アドレスの機器がいるか**を覚えています。これを **MAC アドレス表**（bridge では **FDB**、Forwarding DataBase）と呼びます。

```bash
sudo ip netns exec sw1 bridge fdb show br br0 dynamic
```

```text
7a:87:09:62:f1:9c dev sw1-p1 master br0
c6:f4:41:ba:5d:7b dev sw1-p2 master br0
82:3f:02:6b:fc:6c dev sw1-p3 master br0
```

`dynamic` は「スイッチが自分で覚えたものだけを表示する」という指定です。「`7a:87:...` は `sw1-p1` の先にいる」と読みます。各 PC の MAC アドレスと見比べてみましょう。

```bash
sudo ip netns exec pc1 ip link show veth-pc1
```

```text
4: veth-pc1@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 7a:87:09:62:f1:9c brd ff:ff:ff:ff:ff:ff link-netns sw1
```

`pc1` の MAC アドレス `7a:87:09:62:f1:9c` が、`sw1-p1` の先にいると記録されています。

スイッチはこの表を、**届いたフレームの送信元 MAC アドレス**から作っています。「`sw1-p1` から、送信元 `7a:87:...` のフレームが入ってきた。なら `7a:87:...` は `sw1-p1` の先にいる」と覚えるのです。誰かが設定したわけではなく、流れてきたフレームを見て自分で学習しています。

## 4-7. 観察する —— 誰に何が届いているか

スイッチが MAC アドレス表をどう使っているかを、**関係ない第三者の `pc3`** の側から観察します。

まず、3台すべての ARP 表を空にしておきます。こうすると、次の `ping` で ARP からやり直しになります。

```bash
sudo ip netns exec pc1 ip neigh flush all
sudo ip netns exec pc2 ip neigh flush all
sudo ip netns exec pc3 ip neigh flush all
```

`pc3` も空にするのは、観察の邪魔になる行を減らすためです。`pc3` の ARP 表にメモが残っていると、第3章の 3-6 で見た「このメモ、まだ合ってる？」という確認の ARP を `pc3` 自身が送り、それも画面に出てしまいます。

**ターミナル2**（`pc3` で待ち受ける）：

```bash
sudo ip netns exec pc3 tcpdump -n -e -i veth-pc3 arp or icmp
```

**ターミナル1**（`pc1` から `pc2` へ `ping`）：

```bash
sudo ip netns exec pc1 ping -c 2 192.168.0.2
```

ターミナル2（`pc3`）には、次の1行だけが出ます（時刻は省いています）。

```text
7a:87:09:62:f1:9c > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42:
    Request who-has 192.168.0.2 tell 192.168.0.1, length 28
```

`pc1` と `pc2` の間では、第3章で見たとおり ARP の問い合わせ・返事と、`ping` の行き帰り2往復が流れたはずです。でも `pc3` に届いたのは、**全員宛て（`ff:ff:ff:ff:ff:ff`）の ARP の問い合わせだけ**でした。

| フレーム | 宛先 MAC | スイッチの動き | `pc3` に届く？ |
|---|---|---|---|
| ARP Request | `ff:ff:ff:ff:ff:ff`（全員） | 入ってきたポート以外の**全ポート**に送る | 届く |
| ARP Reply | `pc1` | MAC アドレス表を見て、`pc1` のポートにだけ送る | 届かない |
| ICMP echo request | `pc2` | `pc2` のポートにだけ送る | 届かない |
| ICMP echo reply | `pc1` | `pc1` のポートにだけ送る | 届かない |

スイッチは、**全員宛てなら全員に、個別宛てなら MAC アドレス表を見てその人にだけ**届けています。だから `pc3` には、自分に関係のない `pc1` と `pc2` の会話が聞こえません。

## 4-8. 壊してみる —— スイッチの記憶を消したら

スイッチが MAC アドレス表を持てなくなると、どうなるでしょうか。

bridge には、覚えた内容を何秒で忘れるかという設定（`ageing_time`）があります。これを **0**（すぐ忘れる）にして、スイッチが何も覚えられない状態にします。

```bash
sudo ip netns exec sw1 ip link set br0 type bridge ageing_time 0
sudo ip netns exec sw1 bridge fdb show br br0 dynamic
```

何も表示されません。MAC アドレス表が空になりました。

ターミナル2で `pc3` の `tcpdump` をもう一度動かし、ターミナル1で `pc1` から `pc2` に `ping` を打ちます。

```bash
sudo ip netns exec pc3 tcpdump -n -e -i veth-pc3 arp or icmp
```

```bash
sudo ip netns exec pc1 ping -c 2 192.168.0.2
```

ターミナル1の `ping` は、今までどおり成功します。ところがターミナル2（`pc3`）には、次のような行が出ます。

```text
7a:87:09:62:f1:9c > c6:f4:41:ba:5d:7b, ethertype IPv4 (0x0800), length 98:
    192.168.0.1 > 192.168.0.2: ICMP echo request, id 1711, seq 1, length 64
c6:f4:41:ba:5d:7b > 7a:87:09:62:f1:9c, ethertype IPv4 (0x0800), length 98:
    192.168.0.2 > 192.168.0.1: ICMP echo reply, id 1711, seq 1, length 64
7a:87:09:62:f1:9c > c6:f4:41:ba:5d:7b, ethertype IPv4 (0x0800), length 98:
    192.168.0.1 > 192.168.0.2: ICMP echo request, id 1711, seq 2, length 64
c6:f4:41:ba:5d:7b > 7a:87:09:62:f1:9c, ethertype IPv4 (0x0800), length 98:
    192.168.0.2 > 192.168.0.1: ICMP echo reply, id 1711, seq 2, length 64
```

（第3章の 3-6 で見た、確認のための ARP が混じることもあります。）

**`pc1` と `pc2` の個別の会話が、`pc3` に丸見えになりました**。宛先 MAC は `pc2` や `pc1` なのに、`pc3` のケーブルにまで流れてきています。

スイッチは、宛先 MAC アドレスが表に無いとき、**とりあえず全ポートに送る**という動きをします（**フラッディング**）。表が空なら、毎回すべてのフレームが全員に配られます。これはスイッチが登場する前の **ハブ**（リピータハブ）と同じ動きです。

この状態の困ったところは2つあります。

- **盗み見られる**：関係ない PC にも会話が流れる。ここでは `ping` ですが、暗号化されていない通信なら中身まで読めてしまいます
- **混む**：台数が増えると、全員のケーブルに全員の通信が流れて、ネットワーク全体が遅くなります

:::message
攻撃者が偽の送信元 MAC アドレスのフレームを大量に送り、スイッチの MAC アドレス表をあふれさせて、この「全員に配る」状態に追い込む攻撃があります（**MAC フラッディング**）。業務用のスイッチには、ポートごとに覚えられる MAC アドレスの数を制限する機能が付いています。
:::

## 4-9. 直す —— 記憶を戻す

`ageing_time` を元の値に戻します。単位は 1/100 秒で、`30000` は 300 秒（5分）です。

```bash
sudo ip netns exec sw1 ip link set br0 type bridge ageing_time 30000
sudo ip netns exec pc1 ping -c 1 192.168.0.2
sudo ip netns exec sw1 bridge fdb show br br0 dynamic
```

```text
PING 192.168.0.2 (192.168.0.2) 56(84) bytes of data.
64 bytes from 192.168.0.2: icmp_seq=1 ttl=64 time=0.016 ms

--- 192.168.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.016/0.016/0.016/0.000 ms
7a:87:09:62:f1:9c dev sw1-p1 master br0
c6:f4:41:ba:5d:7b dev sw1-p2 master br0
```

`ping` 1回分のフレームを見て、スイッチは `pc1` と `pc2` の居場所を覚え直しました。`pc3` はまだ何も送っていないので、表に載っていません。

もう一度 `pc3` で `tcpdump` を動かし、`pc1` から `pc2` に `ping` を打ちます。

```bash
sudo ip netns exec pc3 tcpdump -n -e -i veth-pc3 arp or icmp
```

```bash
sudo ip netns exec pc1 ping -c 2 192.168.0.2
```

今度は、`pc3` には何も表示されません。スイッチが元どおり、宛先のポートにだけ届けるようになりました。

## 4-10. 練習問題

**問1.** 4台目の `pc4` を `sw1` につなぎ、IP アドレスを `192.168.1.4/24` にしました。`pc1`（`192.168.0.1/24`）から `pc4` に `ping` は通りますか？

:::details 答え
**通りません**。`pc1` は `Network is unreachable` になります。

スイッチは MAC アドレスでフレームを中継するだけで、IP アドレスは見ていません。`pc4` は `pc1` と同じスイッチにつながっていますが、`pc1` から見ると `192.168.1.4` は**よその町**です（第2章の 2-3）。よその町の相手には直接送らないので、ARP の問い合わせすら出しません。

「同じスイッチにつながっている」と「同じネットワーク（町）にいる」は別の話です。実際に `pc4` をつないで確かめてみてください。
:::

**問2.** 4-7 で、`pc3` に ARP の問い合わせが届いたのに、`pc3` は返事をしませんでした。なぜですか？

:::details 答え
問い合わせの中身が「`192.168.0.2` を持っている人は？」だったからです。`pc3` は `192.168.0.3` なので、自分のことではないと判断して無視しました。全員宛ての問い合わせは全員に届きますが、**返事をするのは該当する1台だけ**です。
:::

## 4-11. 片付け

```bash
sudo ip -all netns delete
```

## まとめ

- **スイッチ**は複数のポートを持ち、ポート同士でフレームを中継する。Linux では **bridge** で作れる
- ポートが全部つながっていても、**スイッチ本体（`br0`）が `DOWN` だと何も中継されない**。ポートの状態は `bridge link` で `forwarding` か確認する
- スイッチは、届いたフレームの**送信元 MAC アドレス**から「どのポートの先に誰がいるか」を学習する（**MAC アドレス表 / FDB**）
- **全員宛ては全ポートへ、個別宛ては表を見てそのポートだけへ**。だから関係ない PC には他人の会話が流れない
- 表に無い宛先は全ポートに配られる（**フラッディング**）。表が使えないスイッチはハブと同じで、盗み見と混雑の原因になる
- スイッチは IP アドレスを見ない。同じスイッチにつながっていても、**町（ネットワーク）が違えば直接は通信できない**

問1で見たとおり、町が違う相手とは、同じスイッチにつながっていても話せません。では、`192.168.0.x` の町と `192.168.1.x` の町をつなぐにはどうすればいいのでしょうか。次の章では、町と町をつなぐ**ルータ**を作ります。
