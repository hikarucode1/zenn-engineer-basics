---
title: "NAT"
---

## この章のゴール

第2章で、`192.168.x.x` のような**プライベートアドレス**は「家や会社の中だけで自由に使ってよい」範囲だと説明しました。世界中の家で同じ `192.168.0.1` が使われているので、インターネット側は、どの家の `192.168.0.1` に返事を届ければいいのか分かりません。

それなのに、家の PC からインターネット上のサーバに普通にアクセスできています。間に入っている**家庭用ルータ**が、**NAT**（Network Address Translation、アドレス変換）という仕事をしているからです。

この章では、家庭用ルータを netns で作り、次のことを確かめます。

- プライベートアドレスのままでは、**インターネット上のサーバから返事が返ってこない**
- NAT は、出ていくパケットの送信元を**ルータのアドレスに書き換え**、返事を元に戻して中の PC に届ける
- 複数の PC が同時に外へ出られるのは、ルータが**ポート番号も書き換えて**区別しているから
- 外から中の PC へは、そのままでは届かない。届けるには**ポートフォワード**を設定する

## 9-1. 作るもの

```text
           家の中（LAN）                         インターネット（WAN）
        192.168.0.0/24                          203.0.113.0/24

 pc1  192.168.0.1 ---+
                     |    r1（家庭用ルータ）
                     +--- br0               r1-wan ============ veth-srv
                     |    192.168.0.254     203.0.113.1         203.0.113.100
 pc2  192.168.0.2 ---+                                           srv（サーバ）
```

- **`r1`**：家庭用ルータ。家の中側は第4章のスイッチ（bridge `br0`）になっていて、`pc1` と `pc2` がつながります。外側の口 `r1-wan` には、インターネット側の住所 `203.0.113.1` を付けます
- **`srv`**：インターネット上のサーバのつもりの機器です

`203.0.113.0/24` は、**説明や練習のために使ってよい**と決められている住所の範囲です（本物のインターネットでは使われていません）。ここではこれを、プロバイダから割り当てられた**グローバルアドレス**の代わりに使います。

## 9-2. 組み立てる

4台の箱を作ります。

```bash
sudo ip netns add pc1
sudo ip netns add pc2
sudo ip netns add r1
sudo ip netns add srv
```

`r1` の中に家の中側のスイッチ `br0` を作り、ケーブルを3本つなぎます。

```bash
sudo ip netns exec r1 ip link add br0 type bridge
sudo ip link add veth-pc1 type veth peer name r1-p1
sudo ip link add veth-pc2 type veth peer name r1-p2
sudo ip link add r1-wan type veth peer name veth-srv
sudo ip link set veth-pc1 netns pc1
sudo ip link set veth-pc2 netns pc2
sudo ip link set veth-srv netns srv
sudo ip link set r1-p1 netns r1
sudo ip link set r1-p2 netns r1
sudo ip link set r1-wan netns r1
sudo ip netns exec r1 ip link set r1-p1 master br0
sudo ip netns exec r1 ip link set r1-p2 master br0
```

住所を付けます。ルータの家の中側の住所は、ポートではなく **`br0` 自体**に付けます。こうすると、`br0` が「スイッチ」と「ルータの家の中側の口」を兼ねます。家庭用ルータの背面に LAN ポートが何個も並んでいるのと同じ形です。

```bash
sudo ip netns exec r1 ip addr add 192.168.0.254/24 dev br0
sudo ip netns exec r1 ip addr add 203.0.113.1/24 dev r1-wan
sudo ip netns exec pc1 ip addr add 192.168.0.1/24 dev veth-pc1
sudo ip netns exec pc2 ip addr add 192.168.0.2/24 dev veth-pc2
sudo ip netns exec srv ip addr add 203.0.113.100/24 dev veth-srv
```

口を全部 `up` にし、`r1` をルータにして、PC にデフォルトゲートウェイを教えます。

```bash
sudo ip netns exec r1 ip link set br0 up
sudo ip netns exec r1 ip link set r1-p1 up
sudo ip netns exec r1 ip link set r1-p2 up
sudo ip netns exec r1 ip link set r1-wan up
sudo ip netns exec pc1 ip link set veth-pc1 up
sudo ip netns exec pc2 ip link set veth-pc2 up
sudo ip netns exec srv ip link set veth-srv up
sudo ip netns exec r1 sysctl -w net.ipv4.ip_forward=1
sudo ip netns exec pc1 ip route add default via 192.168.0.254
sudo ip netns exec pc2 ip route add default via 192.168.0.254
```

`srv` には、`192.168.0.0/24` への道を**わざと教えません**。インターネット上のサーバは、どこかの家の中にあるプライベートアドレスへの道を知らないからです。

```bash
sudo ip netns exec srv ip route
```

```text
203.0.113.0/24 dev veth-srv proto kernel scope link src 203.0.113.100
```

## 9-3. 壊れている —— プライベートアドレスのまま外へ出ると

`pc1` から、家の中の `pc2` と、ルータの外側の住所 `203.0.113.1` には届きます。

```bash
sudo ip netns exec pc1 ping -c 1 192.168.0.2
sudo ip netns exec pc1 ping -c 1 203.0.113.1
```

```text
PING 192.168.0.2 (192.168.0.2) 56(84) bytes of data.
64 bytes from 192.168.0.2: icmp_seq=1 ttl=64 time=0.032 ms

--- 192.168.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.032/0.032/0.032/0.000 ms
PING 203.0.113.1 (203.0.113.1) 56(84) bytes of data.
64 bytes from 203.0.113.1: icmp_seq=1 ttl=64 time=0.021 ms

--- 203.0.113.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.021/0.021/0.021/0.000 ms
```

（`203.0.113.1` は `r1` 自身の住所なので、第5章の注意書きのとおり、転送しなくても返事が来ます。）

では、インターネット上の `srv` はどうでしょうか。

```bash
sudo ip netns exec pc1 ping -c 2 -W 1 203.0.113.100
```

```text
PING 203.0.113.100 (203.0.113.100) 56(84) bytes of data.

--- 203.0.113.100 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1031ms

```

届きません。`srv` で待ち受けてみます。

**ターミナル2**：

```bash
sudo ip netns exec srv tcpdump -n -i veth-srv icmp
```

**ターミナル1**：

```bash
sudo ip netns exec pc1 ping -c 1 -W 1 203.0.113.100
```

ターミナル2（時刻は省いています）：

```text
IP 192.168.0.1 > 203.0.113.100: ICMP echo request, id 1912, seq 1, length 64
```

`ping` は `srv` まで**届いています**。でも、送信元が **`192.168.0.1`** のままです。`srv` は返事を `192.168.0.1` に送ろうとしますが、

```bash
sudo ip netns exec srv ip route get 192.168.0.1
```

```text
RTNETLINK answers: Network is unreachable
```

その道を知りません。第5・6章で見た「帰り道がない」と同じ症状ですが、今回は**道を書き足して直すことができません**。本物のインターネットでは、世界中の家に `192.168.0.1` があるので、どの家への道を書けばいいのか決めようがないからです。

## 9-4. NAT を設定する

そこで、ルータ `r1` に「**外へ出ていくパケットの送信元を、自分の外側の住所 `203.0.113.1` に書き換えろ**」と設定します。返事は `203.0.113.1` 宛てに返ってくるので、`srv` は道に困りません。

Linux でこの設定をするのが **nftables**（`nft` コマンド）です。nftables は、ルータやサーバを通るパケットを**ルールに従って書き換えたり、捨てたり**する仕組みで、ファイアウォールもこれで作ります。

```bash
sudo ip netns exec r1 nft add table ip nat
sudo ip netns exec r1 nft add chain ip nat postrouting '{ type nat hook postrouting priority srcnat ; }'
sudo ip netns exec r1 nft add rule ip nat postrouting oifname r1-wan masquerade
```

3行の意味は次のとおりです。

| 行 | 意味 |
|---|---|
| 1 | `nat` という名前の**表**（table）を作る。ルールをまとめる入れ物 |
| 2 | その中に `postrouting` という**ルールの列**（chain）を作る。`hook postrouting` は「行き先の口が決まって、**出ていく直前**」のパケットにルールを当てる、という指定 |
| 3 | 「出ていく口（`oifname`）が `r1-wan` なら、送信元をその口の住所に書き換える（**`masquerade`**）」というルールを足す |

設定を確認します。

```bash
sudo ip netns exec r1 nft list ruleset
```

```text
table ip nat {
	chain postrouting {
		type nat hook postrouting priority srcnat; policy accept;
		oifname "r1-wan" masquerade
	}
}
```

## 9-5. 観察する —— 送信元が書き換わる

ルータの両側で `tcpdump` を取ります。

**ターミナル2**（インターネット側、`srv`）：

```bash
sudo ip netns exec srv tcpdump -n -i veth-srv icmp
```

**ターミナル3**（家の中側、`pc1`）：

```bash
sudo ip netns exec pc1 tcpdump -n -i veth-pc1 icmp
```

**ターミナル1**：

```bash
sudo ip netns exec pc1 ping -c 1 203.0.113.100
```

```text
PING 203.0.113.100 (203.0.113.100) 56(84) bytes of data.
64 bytes from 203.0.113.100: icmp_seq=1 ttl=63 time=0.046 ms

--- 203.0.113.100 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.046/0.046/0.046/0.000 ms
```

**届きました**。ルータの両側の様子を比べます（時刻は省いています）。

ターミナル3（家の中側）：

```text
IP 192.168.0.1 > 203.0.113.100: ICMP echo request, id 1938, seq 1, length 64
IP 203.0.113.100 > 192.168.0.1: ICMP echo reply, id 1938, seq 1, length 64
```

ターミナル2（インターネット側）：

```text
IP 203.0.113.1 > 203.0.113.100: ICMP echo request, id 1938, seq 1, length 64
IP 203.0.113.100 > 203.0.113.1: ICMP echo reply, id 1938, seq 1, length 64
```

| 場所 | 行き（echo request）の送信元 | 帰り（echo reply）の宛先 |
|---|---|---|
| 家の中側 | `192.168.0.1`（`pc1`） | `192.168.0.1`（`pc1`） |
| インターネット側 | **`203.0.113.1`**（`r1`） | **`203.0.113.1`**（`r1`） |

- 行き：`r1` が、送信元を `192.168.0.1` から **`203.0.113.1` に書き換えた**
- 帰り：`srv` は `203.0.113.1` に返事をした。`r1` は、宛先を **`192.168.0.1` に書き戻して** `pc1` に届けた

`srv` から見ると、話している相手は `r1`（`203.0.113.1`）だけです。**家の中に `pc1` がいることも、そのプライベートアドレスも、`srv` には見えていません**。

第5章の「ルータを通っても IP アドレスは変わらない」の例外が、この NAT です。

### ルータは書き換えを覚えている

`r1` は、帰りの返事を誰に戻せばいいかを、どうやって知ったのでしょうか。書き換えるときに、**「誰の通信を、どう書き換えたか」を記録している**のです。この記録（**conntrack**、コネクション追跡）を見るツールを入れます。

```bash
sudo apt install -y conntrack
sudo ip netns exec r1 conntrack -L
```

```text
icmp     1 25 src=192.168.0.1 dst=203.0.113.100 type=8 code=0 id=1938 src=203.0.113.100 dst=203.0.113.1 type=0 code=0 id=1938 mark=0 use=1
conntrack v1.4.8 (conntrack-tools): 1 flow entries have been shown.
```

1行に `src=... dst=...` が2組あります。

- 前半 `src=192.168.0.1 dst=203.0.113.100`：**元の通信**（`pc1` から `srv` へ）
- 後半 `src=203.0.113.100 dst=203.0.113.1`：**返事として期待している通信**（`srv` から `r1` へ）

`r1` は、後半に一致するパケットが来たら「これは前半の通信への返事だ」と判断して、宛先を `192.168.0.1` に書き戻します。`25` は、この記録が消えるまでの残り秒数です。前の `ping` から時間がたっていると、記録が消えて何も表示されないことがあります。そのときは `ping` をもう一度打ってから見てください。

## 9-6. 2台が同時に外へ出る —— ポート番号も書き換える

家の中の `pc1` と `pc2` が、**同時に同じサーバ**へ TCP で接続したらどうなるでしょうか。どちらの送信元も `203.0.113.1` に書き換えられたら、返事をどちらに戻せばいいのか区別がつかなくなりそうです。

わざと区別しにくい状況を作ります。`pc1` と `pc2` の両方に、**同じ送信元ポート 40000 番**を使わせます。

**ターミナル2**（`srv` の 80 番で待ち受ける。`-k` は、1つ目の接続が終わっても待ち受けを続ける指定）：

```bash
sudo ip netns exec srv nc -lk 80
```

**ターミナル1**：

```bash
sleep 60 | sudo ip netns exec pc1 nc -p 40000 203.0.113.100 80 &
sleep 60 | sudo ip netns exec pc2 nc -p 40000 203.0.113.100 80 &
```

`-p 40000` は「送信元ポートを 40000 番にする」、行末の `&` は「裏で動かして、すぐ次のコマンドを打てるようにする」という指定です。`sleep 60 |` を付けているので、どちらも60秒間つながったままになります。

それぞれの側から接続を見てみます。

```bash
sudo ip netns exec pc1 ss -tn
sudo ip netns exec pc2 ss -tn
sudo ip netns exec srv ss -tn
```

```text
State Recv-Q Send-Q Local Address:Port   Peer Address:PortProcess
ESTAB 0      0        192.168.0.1:40000 203.0.113.100:80
State Recv-Q Send-Q Local Address:Port   Peer Address:PortProcess
ESTAB 0      0        192.168.0.2:40000 203.0.113.100:80
State Recv-Q Send-Q Local Address:Port Peer Address:Port Process
ESTAB 0      0      203.0.113.100:80    203.0.113.1:54591
ESTAB 0      0      203.0.113.100:80    203.0.113.1:40000
```

`pc1` も `pc2` も、自分では「40000 番から接続している」と思っています。ところが `srv` から見ると、**`203.0.113.1` の 40000 番**と、**`203.0.113.1` の 54591 番**という、2つの別の相手に見えています。

`r1` の記録を見ると、何が起きたかが分かります。

```bash
sudo ip netns exec r1 conntrack -L -p tcp
```

```text
tcp      6 431998 ESTABLISHED src=192.168.0.1 dst=203.0.113.100 sport=40000 dport=80 src=203.0.113.100 dst=203.0.113.1 sport=80 dport=40000 [ASSURED] mark=0 use=1
tcp      6 431998 ESTABLISHED src=192.168.0.2 dst=203.0.113.100 sport=40000 dport=80 src=203.0.113.100 dst=203.0.113.1 sport=80 dport=54591 [ASSURED] mark=0 use=1
conntrack v1.4.8 (conntrack-tools): 2 flow entries have been shown.
```

| 家の中の通信 | 外から見た通信 |
|---|---|
| `192.168.0.1` の **40000** 番 → `srv` の 80 番 | `203.0.113.1` の **40000** 番 → `srv` の 80 番 |
| `192.168.0.2` の **40000** 番 → `srv` の 80 番 | `203.0.113.1` の **54591** 番 → `srv` の 80 番 |

先に接続した `pc1` はポート番号そのまま、後から来た `pc2` は、40000 番がもう使われていたので **54591 番に書き換えられました**（どちらが書き換えられるかは、接続した順番で変わります）。`srv` からの返事は宛先ポートで区別できるので、`r1` はそれぞれを正しい PC に戻せます。

このように、**IP アドレスだけでなくポート番号も書き換えて**、1つのグローバルアドレスを何台もの機器で共有する仕組みを **NAPT**（または IP マスカレード）と呼びます。家庭用ルータがやっている NAT は、ほとんどがこれです。

確認したら、ターミナル2の `nc` を `Ctrl+C` で止めておきます（ターミナル1の2つは、60秒たてば自然に終わります）。

## 9-7. 壊れている —— 外から中へは届かない

今度は逆向きに、**インターネット側の `srv` から、家の中の `pc1` へ**接続してみます。`pc1` で待ち受けます。

**ターミナル2**：

```bash
sudo ip netns exec pc1 nc -l 9999
```

**ターミナル1**：`srv` から `pc1` のプライベートアドレスへ接続します。

```bash
echo hi | sudo ip netns exec srv nc -v -N -w 2 192.168.0.1 9999
```

```text
nc: connect to 192.168.0.1 port 9999 (tcp) failed: Network is unreachable
```

`srv` は `192.168.0.1` への道を知らないので、送り出すこともできません。では、ルータの外側の住所 `203.0.113.1` に接続したらどうでしょうか。

```bash
echo hi | sudo ip netns exec srv nc -v -N -w 2 203.0.113.1 9999
```

```text
nc: connect to 203.0.113.1 port 9999 (tcp) failed: Connection refused
```

**`Connection refused`** です。第8章で見たとおり、`203.0.113.1`（`r1`）の 9999 番で待ち受けているプログラムはいません。`r1` は、外から来た接続を**家の中のどの PC に渡せばいいのか知らない**ので、自分宛てとして受け取り、断ったのです。

9-5 では、中から外へ出ていくときに記録（conntrack）を作り、その返事だけを中に戻していました。**外から突然来た接続には記録がない**ので、中には入れません。これは、外から家の中の機器に勝手に接続されないという意味で、**防御としても働いています**。

## 9-8. ポートフォワード —— 外から中へ道を開ける

家のサーバをインターネットに公開したいときなど、外から中の特定の PC に届けたいこともあります。そのためには、ルータに「**外側の 8080 番に来た接続は、`pc1` の 9999 番に回せ**」と設定します。これを**ポートフォワード**と呼びます。

```bash
sudo ip netns exec r1 nft add chain ip nat prerouting '{ type nat hook prerouting priority dstnat ; }'
sudo ip netns exec r1 nft add rule ip nat prerouting iifname r1-wan tcp dport 8080 dnat to 192.168.0.1:9999
```

| 行 | 意味 |
|---|---|
| 1 | `prerouting` という chain を作る。`hook prerouting` は「パケットが**入ってきた直後**、行き先を決める前」にルールを当てる、という指定 |
| 2 | 「`r1-wan` から入ってきた（`iifname`）、宛先ポートが 8080 番の TCP なら、宛先を `192.168.0.1` の 9999 番に書き換える（**`dnat`**）」 |

9-4 の `masquerade` は**送信元**（source）を書き換えるので SNAT、こちらは**宛先**（destination）を書き換えるので **DNAT** と呼ばれます。

**ターミナル2**（`pc1` で待ち受け直す）：

```bash
sudo ip netns exec pc1 nc -l 9999
```

**ターミナル1**（`srv` から `r1` の 8080 番へ）：

```bash
echo hello-from-internet | sudo ip netns exec srv nc -v -N -w 2 203.0.113.1 8080
```

```text
Connection to 203.0.113.1 8080 port [tcp/http-alt] succeeded!
```

ターミナル2（`pc1`）：

```text
hello-from-internet
```

**インターネット側から、家の中の `pc1` に届きました**。送る前に別のターミナルで `sudo ip netns exec pc1 tcpdump -n -i veth-pc1 tcp` を動かしておくと、宛先が `192.168.0.1.9999` に書き換えられて届いていることが分かります（時刻と `options` 以降は省いています）。

```text
IP 203.0.113.100.37546 > 192.168.0.1.9999: Flags [S], seq 3928294814, win 64240, ...
```

:::message
ポートフォワードは、家の中の機器を**インターネット全体に向けて開ける**設定です。公開した先のプログラムに弱点があれば、世界中から攻撃されます。使い終わったら必ず消しましょう。家庭用ルータの「ポート開放」や、ゲーム機・監視カメラの UPnP による自動設定も、この仕組みです。
:::

最後に、ここまでの設定を全部見ておきます。

```bash
sudo ip netns exec r1 nft list ruleset
```

```text
table ip nat {
	chain postrouting {
		type nat hook postrouting priority srcnat; policy accept;
		oifname "r1-wan" masquerade
	}

	chain prerouting {
		type nat hook prerouting priority dstnat; policy accept;
		iifname "r1-wan" tcp dport 8080 dnat to 192.168.0.1:9999
	}
}
```

## 9-9. 片付け

```bash
sudo ip -all netns delete
```

netns を消すと、その中の nftables の設定や conntrack の記録も一緒に消えます。

## まとめ

- プライベートアドレスのままインターネットへ出ると、**相手から返事が返ってこない**。インターネット側は、どの家の `192.168.0.1` かを区別できない
- **NAT**（`masquerade` / SNAT）は、出ていくパケットの送信元を**ルータのグローバルアドレスに書き換え**、返事を元の PC に書き戻す。外からは家の中の PC が見えない
- ルータは書き換えた通信を **conntrack** に記録していて、返事をどの PC に戻すかをそれで判断する
- 複数の PC が同時に外へ出るときは、**ポート番号も書き換えて**区別する（**NAPT**）
- 外から中への接続は、記録がないので届かない。届けるには **ポートフォワード**（DNAT）を設定する。開けすぎは危険
- Linux では、これらを **nftables**（`nft`）で設定する

ここまで、相手はずっと `203.0.113.100` のような**数字の住所**で指定してきました。でも普段、ブラウザに打ち込むのは `example.com` のような**名前**です。次の章では、名前から IP アドレスを調べる仕組み、**DNS** を作ります。
