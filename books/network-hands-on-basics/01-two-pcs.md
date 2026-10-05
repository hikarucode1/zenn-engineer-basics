---
title: "PC の中に PC を2台作る"
---

## この章のゴール

この章では、**PC を2台作り、LAN ケーブル1本でつないで、`ping` を通します**。

```text
+-------------------+                      +-------------------+
|        pc1        |                      |        pc2        |
|                   |       (veth)         |                   |
|   veth-pc1 [o]===========================[o] veth-pc2        |
|  192.168.0.1/24   |                      |  192.168.0.2/24   |
+-------------------+                      +-------------------+
```

ネットワークの最小単位は「2台が1本のケーブルでつながっている状態」です。ここで「PC を作る」「ケーブルを挿す」「住所（IP アドレス）を決める」「電源を入れる」という手順を一つずつ踏むと、後の章で何十本コマンドを打っても、やっていることはこの組み合わせだと分かるようになります。

## 1-1. network namespace —— 1台の中の「別の PC」

**network namespace**（以下 netns）は、Linux の中に**ネットワークだけが独立した小部屋**を作る機能です。

ふつうの PC は、次のようなネットワークの持ち物を1セットずつ持っています。

- ネットワークインターフェース（LAN ケーブルの差し込み口。Wi-Fi なら無線の口）
- IP アドレス
- ルーティングテーブル（「この宛先にはこの口から出す」という道案内の表）

netns を1つ作ると、この**持ち物一式がもう1セット**できます。小部屋の中のプログラムからは、自分の部屋の持ち物しか見えません。つまり外から見ると、**ネットワーク的には別の PC が1台増えたのと同じ**です。

:::message
Docker のコンテナが、ホストとは別の IP アドレスを持てるのも netns のおかげです。この本でやることは、Docker が裏で自動でやっていることを手作業でやり直すことでもあります。
:::

## 1-2. PC を2台作る

`pc1` と `pc2` という名前で netns を2つ作ります。

```bash
sudo ip netns add pc1
sudo ip netns add pc2
```

作った netns の一覧を確認します。

```bash
ip netns list
```

```text
pc2
pc1
```

2台できました。では `pc1` の中に入って、どんなネットワークの口を持っているか見てみましょう。

`ip netns exec <netns名> <コマンド>` と書くと、**その netns の中でコマンドを実行**できます。本書ではこの書き方を何百回も使います。「`pc1` の中で `ip link` を打つ」と読んでください。

```bash
sudo ip netns exec pc1 ip link
```

```text
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

`ip link` はネットワークの口（インターフェース）の一覧を出すコマンドです。`pc1` には `lo` が1つあるだけです。

`lo`（ループバック）は「自分自身」につながる特別な口で、外の PC とはつながっていません。つまり生まれたての `pc1` は、**LAN ケーブルの差し込み口が1つもない PC** です。しかも `state DOWN` ——電源すら入っていません。

## 1-3. LAN ケーブルを作って挿す

### veth —— 両端が付いたケーブル

PC 同士をつなぐケーブルとして、**veth**（virtual ethernet）を使います。veth は**必ず2つの口がペアで作られ、片方に入れたデータがもう片方から出てくる**仮想のケーブルです。両端にコネクタが付いた LAN ケーブルを1本作る、とイメージしてください。

```bash
sudo ip link add veth-pc1 type veth peer name veth-pc2
```

「`veth-pc1` という口を veth として作る。反対側（peer）の口の名前は `veth-pc2`」という意味です。作ったばかりのケーブルは、まだどの PC にも挿さっておらず、コマンドを打った**元の Linux（以下ホスト）** に置かれています。

```bash
ip link show type veth
```

```text
3: veth-pc2@veth-pc1: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether c6:f4:41:ba:5d:7b brd ff:ff:ff:ff:ff:ff
4: veth-pc1@veth-pc2: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 7a:87:09:62:f1:9c brd ff:ff:ff:ff:ff:ff
```

`veth-pc1@veth-pc2` は「`veth-pc1` の相方は `veth-pc2`」という意味です。2つの口が1本のケーブルの両端であることが分かります。

### ケーブルを PC に挿す

ケーブルの両端を、それぞれの PC に挿します。

```bash
sudo ip link set veth-pc1 netns pc1
sudo ip link set veth-pc2 netns pc2
```

ホストからもう一度見てみます。

```bash
ip link show type veth
```

何も表示されません。口が `pc1` と `pc2` に移動したので、**ホストからは見えなくなった**のです。netns が本当に「別の部屋」になっていることが分かります。

`pc1` の中から見ると、口が増えています。

```bash
sudo ip netns exec pc1 ip link
```

```text
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
4: veth-pc1@if3: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 7a:87:09:62:f1:9c brd ff:ff:ff:ff:ff:ff link-netns pc2
```

相方の表示が `@veth-pc2` から `@if3` に変わりました。相方が別の部屋に行ったので名前では呼べず、「3番の口」と番号で呼んでいます。`link-netns pc2` は「ケーブルの向こう側は `pc2` にある」という意味です。

`link/ether 7a:87:...` は、この口の **MAC アドレス**（口ごとに振られた識別番号）です。第3章で主役になるので、今は「口ごとに番号がある」とだけ覚えておいてください。

## 1-4. 住所（IP アドレス）を決める

ケーブルはつながりましたが、まだお互いの住所がありません。それぞれの口に IP アドレスを付けます。

```bash
sudo ip netns exec pc1 ip addr add 192.168.0.1/24 dev veth-pc1
sudo ip netns exec pc2 ip addr add 192.168.0.2/24 dev veth-pc2
```

「`pc1` の中で、`veth-pc1` という口（dev = device）に `192.168.0.1/24` を付ける」という意味です。末尾の `/24` は「`192.168.0.` までが同じなら同じネットワーク」という範囲の指定です。詳しくは第2章でやります。

## 1-5. まず失敗させてみる

これで通信できそうな気がします。`pc1` から `pc2` に `ping` を打ってみましょう。`ping` は「相手に『届いたら返事して』というデータを送り、返事が来るかを確かめる」コマンドです。`-c 2` は「2回だけ送る」という意味です。

```bash
sudo ip netns exec pc1 ping -c 2 192.168.0.2
```

```text
ping: connect: Network is unreachable
```

失敗しました。**`Network is unreachable`** は「その宛先に出ていく道がない」という意味です。データを送ろうとすらしていません。

原因は、口の電源が入っていない（`state DOWN`）ことです。道案内の表（ルーティングテーブル）を見ると、それがはっきり分かります。

```bash
sudo ip netns exec pc1 ip route
```

何も表示されません。`pc1` は「どこへ出るにも使える道が1本もない」状態です。口が `DOWN` の間は、その口を使う道が表に載らないのです。

## 1-6. 電源を入れて、ping を通す

口を `up` にします。ケーブルの両端なので、両方の PC で行います。

```bash
sudo ip netns exec pc1 ip link set veth-pc1 up
sudo ip netns exec pc2 ip link set veth-pc2 up
```

もう一度、道案内の表を見ます。

```bash
sudo ip netns exec pc1 ip route
```

```text
192.168.0.0/24 dev veth-pc1 proto kernel scope link src 192.168.0.1
```

「`192.168.0.0/24`（`192.168.0.` で始まる宛先）へは `veth-pc1` から出す」という道が、自動で1本追加されました。IP アドレスを付けた口が `up` になると、Linux はその口の先にあるネットワークへの道を自分で書き足します。

では、もう一度 `ping` です。

```bash
sudo ip netns exec pc1 ping -c 3 192.168.0.2
```

```text
PING 192.168.0.2 (192.168.0.2) 56(84) bytes of data.
64 bytes from 192.168.0.2: icmp_seq=1 ttl=64 time=0.017 ms
64 bytes from 192.168.0.2: icmp_seq=2 ttl=64 time=0.026 ms
64 bytes from 192.168.0.2: icmp_seq=3 ttl=64 time=0.029 ms

--- 192.168.0.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2023ms
rtt min/avg/max/mdev = 0.017/0.024/0.029/0.005 ms
```

**通りました。** これが本書で最初に作ったネットワークです。

出力の読み方は次のとおりです。

| 部分 | 意味 |
|---|---|
| `64 bytes from 192.168.0.2` | `192.168.0.2` から 64 バイトの返事が来た |
| `icmp_seq=1` | 何回目に送ったものへの返事か。抜けていたら、その回は返事が来なかった |
| `ttl=64` | 寿命の残り。第6章で、ルータを通るたびに減っていく様子を見ます |
| `time=0.017 ms` | 送ってから返事が来るまでの時間。同じ PC の中なので非常に速い |
| `3 packets transmitted, 3 received, 0% packet loss` | 3回送って3回返ってきた。失敗はゼロ |

口の状態も確認しておきましょう。

```bash
sudo ip netns exec pc1 ip addr show veth-pc1
```

```text
4: veth-pc1@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 7a:87:09:62:f1:9c brd ff:ff:ff:ff:ff:ff link-netns pc2
    inet 192.168.0.1/24 scope global veth-pc1
       valid_lft forever preferred_lft forever
    inet6 fe80::7887:9ff:fe62:f19c/64 scope link
       valid_lft forever preferred_lft forever
```

見るべきところは3か所です。

- **`UP`**：この口の電源が入っている（自分側で `up` にした）
- **`LOWER_UP`**：ケーブルの向こう側も生きている（ランプが点いている状態）
- **`inet 192.168.0.1/24`**：付けた IP アドレス

`inet6 fe80::...` は IPv6 のアドレスで、口を `up` にすると自動で付きます。本書では IPv6 は扱わないので、読み飛ばしてかまいません。

## 1-7. 壊してみる —— ケーブルの向こうが落ちたら

通信できたので、わざと壊します。**`pc2` 側の口だけを `down`** にします。現実でいえば、相手の PC のケーブルが抜けた状態です。

```bash
sudo ip netns exec pc2 ip link set veth-pc2 down
```

`pc1` から `ping` を打ちます。`-W 1` は「返事を1秒だけ待つ」という意味です。

```bash
sudo ip netns exec pc1 ping -c 2 -W 1 192.168.0.2
```

```text
PING 192.168.0.2 (192.168.0.2) 56(84) bytes of data.

--- 192.168.0.2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1000ms

```

今度のエラーは **`100% packet loss`** です。1-5 の `Network is unreachable` とは症状が違うことに注目してください。

| 症状 | 意味 | 調べる場所 |
|---|---|---|
| `Network is unreachable` | 自分の側に、その宛先へ出ていく道がない。**送ってすらいない** | 自分の PC（口の状態・IP アドレス・ルーティングテーブル） |
| `100% packet loss` | 送ったが、返事が1つも返ってこない | 途中の経路、または相手側 |

この違いを知っているだけで、調べる範囲が半分になります。

`pc1` 側の口は、相手が落ちたことに気づいています。

```bash
sudo ip netns exec pc1 ip link show veth-pc1
```

```text
4: veth-pc1@if3: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default qlen 1000
    link/ether 7a:87:09:62:f1:9c brd ff:ff:ff:ff:ff:ff link-netns pc2
```

自分は `UP` のままですが、`LOWER_UP` が消えて **`NO-CARRIER`**（相手からの信号がない）になりました。本物の PC で LAN ケーブルを抜いたときも、同じ表示になります。

### 直す

`pc2` の口を `up` に戻します。

```bash
sudo ip netns exec pc2 ip link set veth-pc2 up
sudo ip netns exec pc1 ping -c 3 192.168.0.2
```

```text
PING 192.168.0.2 (192.168.0.2) 56(84) bytes of data.
64 bytes from 192.168.0.2: icmp_seq=1 ttl=64 time=0.026 ms
64 bytes from 192.168.0.2: icmp_seq=2 ttl=64 time=0.027 ms
64 bytes from 192.168.0.2: icmp_seq=3 ttl=64 time=0.029 ms

--- 192.168.0.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2084ms
rtt min/avg/max/mdev = 0.026/0.027/0.029/0.001 ms
```

:::message
`up` にした直後は、口が使える状態になるまで一瞬かかります。直後の1発目だけ返事が来ないことがあるので、`-c 3` のように複数回送って様子を見てください。
:::

## 1-8. ホストからは届かない

ここで、`pc1` でも `pc2` でもない**ホスト**から `pc2` に `ping` を打ってみます（`ip netns exec` を付けません）。

```bash
ping -c 1 -W 1 192.168.0.2
```

```text
PING 192.168.0.2 (192.168.0.2) 56(84) bytes of data.

--- 192.168.0.2 ping statistics ---
1 packets transmitted, 0 received, 100% packet loss, time 0ms

```

届きません。ホストは `pc1`・`pc2` とケーブルでつながっていないからです。同じ Linux の上で動いていても、**ネットワーク的には完全に別の PC** として扱われていることが分かります。

## 1-9. 自分自身にも ping してみる

最後に、`pc1` から自分自身（`127.0.0.1`）に `ping` を打ちます。

```bash
sudo ip netns exec pc1 ping -c 1 -W 1 127.0.0.1
```

```text
ping: connect: Network is unreachable
```

自分自身にすら届きません。1-2 で見たとおり、`pc1` の `lo` はまだ `DOWN` だからです。ふつうの PC では起動時に `lo` が自動で `up` になりますが、netns では何もかも自分でやる必要があります。

```bash
sudo ip netns exec pc1 ip link set lo up
sudo ip netns exec pc1 ping -c 1 127.0.0.1
```

```text
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.013 ms

--- 127.0.0.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.013/0.013/0.013/0.000 ms
```

## 1-10. 片付け

章の終わりには、作ったものを消しておきます。

```bash
sudo ip netns delete pc1
sudo ip netns delete pc2
```

```bash
ip netns list
ip link show type veth
```

どちらも何も表示されなければ片付け完了です。PC（netns）を消すと、挿さっていたケーブル（veth）も一緒に消えます。

## まとめ

- **netns** は「ネットワークだけが独立した別の PC」。口・IP アドレス・ルーティングテーブルを自分だけで持つ
- **veth** は両端が付いた LAN ケーブル。片方を `pc1`、もう片方を `pc2` に挿す
- 通信できるまでには **PC を作る → ケーブルを挿す → IP アドレスを付ける → 口を `up` にする** の4手順が要る
- 口が `up` になると、Linux はその先のネットワークへの道をルーティングテーブルに自動で書き足す
- **`Network is unreachable`** は「自分側に道がない」、**`100% packet loss`** は「送ったが返事がない」。症状で調べる場所が変わる
- `ip link` の **`NO-CARRIER`** は「ケーブルの向こうから信号が来ていない」

この章では `/24` を「おまじない」のまま使いました。次の章では、この `/24` を変えると何が起きるのかを確かめながら、**「同じネットワーク」とは何か**を見ていきます。
