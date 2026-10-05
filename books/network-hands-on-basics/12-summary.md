---
title: "総まとめ：URL を開くまでに起きること"
---

## この章のゴール

この本では、PC 1台の中に、PC・ケーブル・スイッチ・ルータ・NAT・DNS サーバ・Web サーバを1つずつ作ってきました。最後の章では、これらを**1つのネットワークに組み上げ**、`curl http://www.example.test/` を1回打つ間に何が起きているかを、**最初のパケットから最後のパケットまで**追いかけます。

そして最後に、そのネットワークをいろいろな場所で壊し、これまでの章で身につけた方法で原因を突き止める**総合問題**に挑戦します。

## 12-1. 作るもの

```text
         家の中                     インターネット                  サーバの町
     192.168.0.0/24                 203.0.113.0/24              198.51.100.0/24

                    r1（家庭用ルータ）               r2（インターネット側のルータ）
  pc1 ------------ br0         r1-wan ================ r2-wan        br0 ----+---- dns1
  192.168.0.1      192.168.0.254  203.0.113.1    203.0.113.254  198.51.100.254 |    198.51.100.53
                   （NAT）                                                     |
                                                                               +---- web1
                                                                                    198.51.100.80
```

| 機器 | 役割 | 関係する章 |
|---|---|---|
| `pc1` | あなたの PC | 第1・2章 |
| `r1` | 家庭用ルータ。家の中側はスイッチ、外へ出るときは NAT | 第4・5・9章 |
| `r2` | インターネット側のルータ。`r1` とサーバの町をつなぐ | 第5・6章 |
| `dns1` | DNS サーバ（`www.example.test` → `198.51.100.80`） | 第10章 |
| `web1` | Web サーバ | 第11章 |

`198.51.100.0/24` も、`203.0.113.0/24` と同じく説明用に決められている住所の範囲です。ここではサーバが置かれている、どこか遠くの町の代わりに使います。

## 12-2. 組み立てる

これまでの章では、コマンドを1行ずつ打ってきました。この章は機器が多いので、組み立てのコマンドを**1つのファイル（スクリプト）にまとめて**、一度に実行します。

まず、この章で使う道具がそろっているか確かめます。

```bash
sudo apt install -y dnsmasq-base conntrack traceroute
```

次のコマンドを**まるごとコピーして貼り付けて**ください。`cat > ファイル名 <<'EOF'` から `EOF` までの間の文字が、そのままファイルに書き込まれます。

```bash
cat > ~/netlab-build.sh <<'EOF'
#!/bin/bash
# 第12章：総まとめのネットワークを組み立てる
set -e

# --- 箱を作る（第1章） ---
for ns in pc1 r1 r2 dns1 web1; do
  ip netns add $ns
done

# --- 家の中：pc1 と家庭用ルータ r1（第4・5・9章） ---
ip netns exec r1 ip link add br0 type bridge
ip link add veth-pc1 type veth peer name r1-p1
ip link set veth-pc1 netns pc1
ip link set r1-p1 netns r1
ip netns exec r1 ip link set r1-p1 master br0
ip netns exec r1 ip addr add 192.168.0.254/24 dev br0
ip netns exec pc1 ip addr add 192.168.0.1/24 dev veth-pc1

# --- r1 と r2 をつなぐ：インターネット側（第6・9章） ---
ip link add r1-wan type veth peer name r2-wan
ip link set r1-wan netns r1
ip link set r2-wan netns r2
ip netns exec r1 ip addr add 203.0.113.1/24 dev r1-wan
ip netns exec r2 ip addr add 203.0.113.254/24 dev r2-wan

# --- サーバ側：r2 のスイッチに dns1 と web1（第4・10・11章） ---
ip netns exec r2 ip link add br0 type bridge
ip netns exec r2 ip addr add 198.51.100.254/24 dev br0
for host in dns1:53 web1:80; do
  name=${host%:*}; num=${host#*:}
  ip link add veth-$name type veth peer name r2-$name
  ip link set veth-$name netns $name
  ip link set r2-$name netns r2
  ip netns exec r2 ip link set r2-$name master br0
  ip netns exec $name ip addr add 198.51.100.$num/24 dev veth-$name
done

# --- 口を全部 up にする（第1章） ---
ip netns exec pc1 ip link set veth-pc1 up
ip netns exec r1 ip link set br0 up
ip netns exec r1 ip link set r1-p1 up
ip netns exec r1 ip link set r1-wan up
ip netns exec r2 ip link set r2-wan up
ip netns exec r2 ip link set br0 up
for name in dns1 web1; do
  ip netns exec r2 ip link set r2-$name up
  ip netns exec $name ip link set veth-$name up
done

# --- ルーティング（第5・6章） ---
ip netns exec r1 sysctl -qw net.ipv4.ip_forward=1
ip netns exec r2 sysctl -qw net.ipv4.ip_forward=1
ip netns exec pc1 ip route add default via 192.168.0.254
ip netns exec r1 ip route add default via 203.0.113.254
ip netns exec dns1 ip route add default via 198.51.100.254
ip netns exec web1 ip route add default via 198.51.100.254

# --- NAT（第9章） ---
ip netns exec r1 nft add table ip nat
ip netns exec r1 nft add chain ip nat postrouting '{ type nat hook postrouting priority srcnat ; }'
ip netns exec r1 nft add rule ip nat postrouting oifname r1-wan masquerade

# --- DNS（第10章） ---
ip netns exec dns1 dnsmasq --no-resolv --no-hosts --bind-interfaces \
  --listen-address=198.51.100.53 --local=/example.test/ --local-ttl=300 \
  --host-record=www.example.test,198.51.100.80
mkdir -p /etc/netns/pc1
echo 'nameserver 198.51.100.53' > /etc/netns/pc1/resolv.conf

# --- Web サーバ（第11章） ---
mkdir -p /tmp/www
echo '<h1>Hello from web1</h1>' > /tmp/www/index.html
ip netns exec web1 python3 -m http.server 80 --bind 198.51.100.80 \
  --directory /tmp/www > /tmp/web1.log 2>&1 &

# Web サーバが 80 番で待ち受けを始めるまで待つ
for i in $(seq 1 30); do
  ip netns exec web1 ss -tln | grep -q ':80 ' && break
  sleep 0.5
done

echo "できました"
EOF
```

片付け用のスクリプトも作っておきます。

```bash
cat > ~/netlab-clean.sh <<'EOF'
#!/bin/bash
# 第12章：総まとめのネットワークを片付ける
pkill -f "http.server 80 --bind 198.51.100.80"
pkill dnsmasq
ip -all netns delete
rm -rf /etc/netns/pc1 /tmp/www /tmp/web1.log
echo "片付けました"
EOF
```

スクリプトの中身を、一度ゆっくり読んでみてください。コメントに書いた章を思い出しながら、**1行ずつ「これは何をしているか」を言えるか**確かめてみましょう。言えない行があったら、その章に戻るのがおすすめです。

:::message
スクリプトの中では、`for` を使って同じ形のコマンドをまとめています。たとえば `for ns in pc1 r1 r2 dns1 web1; do ip netns add $ns; done` は、`ip netns add pc1` から `ip netns add web1` までの5行と同じです。また、スクリプトは `sudo` で丸ごと実行するので、中の各行には `sudo` を付けていません。
:::

組み立てます。

```bash
sudo bash ~/netlab-build.sh
```

```text
できました
```

## 12-3. まず開いてみる

```bash
sudo ip netns exec pc1 curl http://www.example.test/
```

```text
<h1>Hello from web1</h1>
```

家の中の `pc1` から、ルータ2台の向こうにある Web サーバのページが開けました。道筋を `traceroute` で確かめます。

```bash
sudo ip netns exec pc1 traceroute -n 198.51.100.80
```

```text
traceroute to 198.51.100.80 (198.51.100.80), 30 hops max, 60 byte packets
 1  192.168.0.254  0.022 ms  0.006 ms  0.005 ms
 2  203.0.113.254  0.043 ms  0.014 ms  0.010 ms
 3  198.51.100.80  0.016 ms  0.011 ms  0.011 ms
```

`r1`（`192.168.0.254`）→ `r2`（`203.0.113.254`）→ `web1` と、図のとおりの道を通っています。

ここで、`r2` の道案内の表を見てみましょう。

```bash
sudo ip netns exec r2 ip route
```

```text
198.51.100.0/24 dev br0 proto kernel scope link src 198.51.100.254
203.0.113.0/24 dev r2-wan proto kernel scope link src 203.0.113.254
```

`r2` は、家の中の `192.168.0.0/24` への道を**知りません**。それでも `pc1` と通信できているのは、`r1` の NAT のおかげです（第9章）。`r2` から見ると、話している相手は `203.0.113.1`（`r1`）だけなので、家の中への道は要らないのです。インターネット上のルータが、世界中の家の中への道を知らなくてよいのと同じです。

## 12-4. URL を開くまでを1本通す

いよいよ、`curl` を1回打つ間に起きることを全部記録します。

### 準備：覚えていることを全部忘れさせる

12-3 の通信で、各機器の ARP 表や `r1` の NAT の記録には、もう情報が残っています。最初から全部見たいので、消しておきます。

```bash
sudo ip netns exec pc1 ip neigh flush all
sudo ip netns exec r1 ip neigh flush all
sudo ip netns exec r2 ip neigh flush all
sudo ip netns exec r1 conntrack -F
```

### 3か所で同時に記録する

`pc1` のケーブル、`r1` と `r2` の間のケーブル、`web1` のケーブルの3か所で、`tcpdump` の表示を**ファイルに書き出しながら**裏で動かします。ターミナルを何個も開く代わりの方法です。

```bash
mkdir -p ~/cap
sudo ip netns exec pc1 tcpdump -n -e -l -i veth-pc1 arp or udp or tcp > ~/cap/pc1.txt 2>/dev/null &
sudo ip netns exec r1 tcpdump -n -e -l -i r1-wan arp or udp or tcp > ~/cap/wan.txt 2>/dev/null &
sudo ip netns exec web1 tcpdump -n -e -l -i veth-web1 arp or tcp > ~/cap/web1.txt 2>/dev/null &
```

`-l` は1行ずつすぐ書き出す指定、`> ~/cap/pc1.txt` は表示をファイルに保存する指定、`2>/dev/null` は開始時のメッセージを捨てる指定です。2秒ほど待ってから、`curl` を1回だけ打ちます。

```bash
sudo ip netns exec pc1 curl -s http://www.example.test/
```

```text
<h1>Hello from web1</h1>
```

記録を止めます。

```bash
sudo pkill tcpdump
```

### 記録を読む

3つのファイルを、起きた順に読んでいきます。長いので、時刻と、TCP の行の `options` 以降は省いて載せます。MAC アドレスの値は環境ごとに違います。

#### ① まず、ゲートウェイの MAC アドレスを調べる（第3・5章）

`~/cap/pc1.txt` の先頭です。

```text
3a:2d:76:8f:b6:cb > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.0.254 tell 192.168.0.1, length 28
d2:01:69:e9:f7:26 > 3a:2d:76:8f:b6:cb, ethertype ARP (0x0806), length 42: Reply 192.168.0.254 is-at d2:01:69:e9:f7:26, length 28
```

`pc1` が最初にしたのは、Web サーバでも DNS サーバでもなく、**デフォルトゲートウェイ `192.168.0.254` の MAC アドレス**を ARP で調べることでした。DNS サーバも Web サーバも町の外にいるので、どちらに送るにも、まず町の出口（`r1`）に渡す必要があるからです（第5章）。

#### ② 名前を調べる（第7・9・10章）

`~/cap/pc1.txt` の続きです。

```text
3a:2d:76:8f:b6:cb > d2:01:69:e9:f7:26, ethertype IPv4 (0x0800), length 76: 192.168.0.1.54452 > 198.51.100.53.53: 42154+ A? www.example.test. (34)
3a:2d:76:8f:b6:cb > d2:01:69:e9:f7:26, ethertype IPv4 (0x0800), length 76: 192.168.0.1.54452 > 198.51.100.53.53: 59305+ AAAA? www.example.test. (34)
d2:01:69:e9:f7:26 > 3a:2d:76:8f:b6:cb, ethertype IPv4 (0x0800), length 92: 198.51.100.53.53 > 192.168.0.1.54452: 42154* 1/0/0 A 198.51.100.80 (50)
d2:01:69:e9:f7:26 > 3a:2d:76:8f:b6:cb, ethertype IPv4 (0x0800), length 76: 198.51.100.53.53 > 192.168.0.1.54452: 59305 0/0/0 (34)
```

`pc1` が DNS サーバの **UDP 53 番**に「`www.example.test` の A レコード（IPv4）は？」「AAAA レコード（IPv6）は？」と聞き、`A 198.51.100.80` という答えを受け取りました（第7・10章）。AAAA には答えがありません（`0/0/0`）。

同じ問い合わせを、`r1` と `r2` の間のケーブル（`~/cap/wan.txt`）で見ると、こうなっています。

```text
d2:a8:ec:c4:0f:43 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 203.0.113.254 tell 203.0.113.1, length 28
52:33:4e:25:87:68 > d2:a8:ec:c4:0f:43, ethertype ARP (0x0806), length 42: Reply 203.0.113.254 is-at 52:33:4e:25:87:68, length 28
d2:a8:ec:c4:0f:43 > 52:33:4e:25:87:68, ethertype IPv4 (0x0800), length 76: 203.0.113.1.54452 > 198.51.100.53.53: 42154+ A? www.example.test. (34)
...
52:33:4e:25:87:68 > d2:a8:ec:c4:0f:43, ethertype IPv4 (0x0800), length 92: 198.51.100.53.53 > 203.0.113.1.54452: 42154* 1/0/0 A 198.51.100.80 (50)
```

- `r1` も、次に渡す相手 `r2`（`203.0.113.254`）の MAC アドレスを ARP で調べている（第5章）
- 送信元が `192.168.0.1` から **`203.0.113.1` に書き換わっている**。`r1` の NAT です（第9章）

#### ③ TCP で接続する（第8章）

名前が `198.51.100.80` だと分かったので、`pc1` は Web サーバの 80 番に TCP で接続します。`~/cap/pc1.txt` の続きです。

```text
3a:2d:76:8f:b6:cb > d2:01:69:e9:f7:26, ethertype IPv4 (0x0800), length 74: 192.168.0.1.59376 > 198.51.100.80.80: Flags [S], seq 1918577097, win 64240, ...
d2:01:69:e9:f7:26 > 3a:2d:76:8f:b6:cb, ethertype IPv4 (0x0800), length 74: 198.51.100.80.80 > 192.168.0.1.59376: Flags [S.], seq 3557679166, ack 1918577098, win 65160, ...
3a:2d:76:8f:b6:cb > d2:01:69:e9:f7:26, ethertype IPv4 (0x0800), length 66: 192.168.0.1.59376 > 198.51.100.80.80: Flags [.], ack 1, win 502, ...
```

`S` → `S.` → `.` の**3ウェイハンドシェイク**です。

同じ接続を、`web1` のケーブル（`~/cap/web1.txt`）で見ます。

```text
6a:92:7f:ac:61:fb > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 198.51.100.53 tell 198.51.100.254, length 28
6a:92:7f:ac:61:fb > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 198.51.100.80 tell 198.51.100.254, length 28
b2:9d:12:24:02:30 > 6a:92:7f:ac:61:fb, ethertype ARP (0x0806), length 42: Reply 198.51.100.80 is-at b2:9d:12:24:02:30, length 28
6a:92:7f:ac:61:fb > b2:9d:12:24:02:30, ethertype IPv4 (0x0800), length 74: 203.0.113.1.59376 > 198.51.100.80.80: Flags [S], seq 1918577097, win 64240, ...
```

- 1行目：`r2` が `dns1`（`198.51.100.53`）を探す ARP が、**`web1` にも届いている**。全員宛て（`ff:ff:ff:ff:ff:ff`）なので、`r2` のスイッチが全ポートに配ったからです（第4章）
- 2・3行目：`r2` が `web1` の MAC アドレスを調べている
- 4行目：`web1` に届いた接続要求の送信元は **`203.0.113.1`**。`web1` は、相手が家の中の `192.168.0.1` だとは知りません（第9章）

`web1` のアクセスログを見ても、記録されているのは `r1` の住所です。

```bash
cat /tmp/web1.log
```

```text
203.0.113.1 - - [05/Oct/2026 21:13:12] "GET / HTTP/1.1" 200 -
203.0.113.1 - - [05/Oct/2026 21:13:14] "GET / HTTP/1.1" 200 -
```

（1行目は 12-3 の `curl`、2行目が今回の `curl` です。）

#### ④ HTTP でページを受け取る（第11章）

`~/cap/pc1.txt` の続きです。

```text
3a:2d:76:8f:b6:cb > d2:01:69:e9:f7:26, ethertype IPv4 (0x0800), length 145: 192.168.0.1.59376 > 198.51.100.80.80: Flags [P.], seq 1:80, ack 1, win 502, ..., length 79: HTTP: GET / HTTP/1.1
d2:01:69:e9:f7:26 > 3a:2d:76:8f:b6:cb, ethertype IPv4 (0x0800), length 66: 198.51.100.80.80 > 192.168.0.1.59376: Flags [.], ack 80, win 509, ...
d2:01:69:e9:f7:26 > 3a:2d:76:8f:b6:cb, ethertype IPv4 (0x0800), length 251: 198.51.100.80.80 > 192.168.0.1.59376: Flags [P.], seq 1:186, ack 80, win 509, ..., length 185: HTTP: HTTP/1.0 200 OK
3a:2d:76:8f:b6:cb > d2:01:69:e9:f7:26, ethertype IPv4 (0x0800), length 66: 192.168.0.1.59376 > 198.51.100.80.80: Flags [.], ack 186, win 501, ...
d2:01:69:e9:f7:26 > 3a:2d:76:8f:b6:cb, ethertype IPv4 (0x0800), length 91: 198.51.100.80.80 > 192.168.0.1.59376: Flags [P.], seq 186:211, ack 80, win 509, ..., length 25: HTTP
```

`GET / HTTP/1.1` を送り、`HTTP/1.0 200 OK` のヘッダと、25 バイトの中身（`<h1>Hello from web1</h1>`）を受け取りました。それぞれに `ack` が返っています（第8・11章）。

#### ⑤ 切断する（第8章）

```text
3a:2d:76:8f:b6:cb > d2:01:69:e9:f7:26, ethertype IPv4 (0x0800), length 66: 192.168.0.1.59376 > 198.51.100.80.80: Flags [F.], seq 80, ack 211, win 501, ...
d2:01:69:e9:f7:26 > 3a:2d:76:8f:b6:cb, ethertype IPv4 (0x0800), length 66: 198.51.100.80.80 > 192.168.0.1.59376: Flags [F.], seq 211, ack 80, win 509, ...
3a:2d:76:8f:b6:cb > d2:01:69:e9:f7:26, ethertype IPv4 (0x0800), length 66: 192.168.0.1.59376 > 198.51.100.80.80: Flags [.], ack 212, win 501, ...
d2:01:69:e9:f7:26 > 3a:2d:76:8f:b6:cb, ethertype IPv4 (0x0800), length 66: 198.51.100.80.80 > 192.168.0.1.59376: Flags [.], ack 81, win 509, ...
```

お互いに `F.`（もう送るものはない）を送り合って、接続が終わりました。

### `r1` は何を覚えていたか

```bash
sudo ip netns exec r1 conntrack -L
```

```text
udp      17 27 src=192.168.0.1 dst=198.51.100.53 sport=54452 dport=53 src=198.51.100.53 dst=203.0.113.1 sport=53 dport=54452 mark=0 use=1
tcp      6 117 TIME_WAIT src=192.168.0.1 dst=198.51.100.80 sport=59376 dport=80 src=198.51.100.80 dst=203.0.113.1 sport=80 dport=59376 [ASSURED] mark=0 use=1
conntrack v1.4.8 (conntrack-tools): 2 flow entries have been shown.
```

DNS の問い合わせ（UDP）と、Web の接続（TCP）の2つを書き換えた記録が残っています。

### 封筒の宛名は、ケーブルごとに付け替えられた

③の接続要求（`Flags [S]`）1つを、3か所で比べます。

| 場所 | 送信元 MAC → 宛先 MAC | 送信元 IP → 宛先 IP |
|---|---|---|
| `pc1` のケーブル | `pc1` → `r1` の `br0` | `192.168.0.1` → `198.51.100.80` |
| `r1`–`r2` のケーブル | `r1-wan` → `r2-wan` | **`203.0.113.1`** → `198.51.100.80` |
| `web1` のケーブル | `r2` の `br0` → `web1` | `203.0.113.1` → `198.51.100.80` |

- **MAC アドレスは、ケーブルごとに付け替えられる**（第3・5章）
- **IP アドレスは、NAT で送信元が書き換えられた以外は変わらない**（第5・9章）

### 全体をまとめると

`curl http://www.example.test/` を1回打つ間に起きたことを、章と対応させて並べます。

| # | 起きたこと | 章 |
|---|---|---|
| 1 | `/etc/resolv.conf` を読み、DNS サーバは `198.51.100.53` だと知る | 第10章 |
| 2 | DNS サーバは町の外なので、デフォルトゲートウェイ `r1` に渡すことにする | 第2・5章 |
| 3 | `r1` の MAC アドレスを ARP で調べる | 第3章 |
| 4 | DNS の問い合わせを UDP の 53 番に送る | 第7・10章 |
| 5 | `r1` が送信元を `203.0.113.1` に書き換え、`r2` に転送する | 第5・9章 |
| 6 | `r2` が、`dns1` へ転送する。スイッチが全員宛ての ARP を全ポートに配る | 第4・6章 |
| 7 | `www.example.test` は `198.51.100.80` だという答えが、同じ道を逆にたどって返る | 第9・10章 |
| 8 | `198.51.100.80` の 80 番に、3ウェイハンドシェイクで TCP 接続する | 第8章 |
| 9 | HTTP で `GET /` を送り、`200 OK` とページの中身を受け取る | 第11章 |
| 10 | FIN で接続を閉じる | 第8章 |

ブラウザでリンクを1回クリックするたびに、これだけのことが、1秒もかからずに起きています。

## 12-5. 総合問題 —— 壊れたネットワークを直す

最後に、このネットワークを5か所で壊します。各問題では、まず**壊すコマンドを打ち**、`curl` でどんなエラーが出るかを見て、**どこが壊れているかを、これまでの章の道具で突き止めて**ください。

壊すコマンドを見れば原因は分かってしまいますが、大事なのは「**原因を知らない人が、症状からどうやってそこにたどり着くか**」を考えることです。答えには、調べる手順を書いています。

各問題を始める前に、ネットワークを作り直して、壊れていない状態に戻してください。

```bash
sudo bash ~/netlab-clean.sh
sudo bash ~/netlab-build.sh
```

### 問1

```bash
sudo ip netns exec r1 ip route del default
sudo ip netns exec pc1 curl http://www.example.test/
```

:::details 答え
`curl` は5秒ほど待ってから、次のエラーで終わります。

```text
curl: (6) Could not resolve host: www.example.test
```

**`Could not resolve host`** なので、まず DNS を疑います。DNS サーバ `198.51.100.53` に `ping` を打ってみます。

```bash
sudo ip netns exec pc1 ping -c 1 -W 1 198.51.100.53
```

```text
From 192.168.0.254 icmp_seq=1 Destination Net Unreachable
```

`r1`（`192.168.0.254`）が「その町には届けられない」と言っています（第6章）。`r1` のルーティングテーブルを見ると、デフォルトゲートウェイが消えています。

**原因**：`r1` に外への道が無いので、DNS サーバに問い合わせが届かなかった。**`Could not resolve host` でも、DNS サーバの設定ではなく、DNS サーバまでの道が壊れていることがある**、という例です。
:::

### 問2

```bash
sudo ip netns exec r2 sysctl -w net.ipv4.ip_forward=0
sudo ip netns exec pc1 curl http://www.example.test/
```

:::details 答え
`curl` は10秒ほど待ってから、`Could not resolve host` で終わります。

DNS サーバへの `ping` は、何も返ってこずに `100% packet loss` です。`traceroute` で、どこまで届いているかを見ます。

```bash
sudo ip netns exec pc1 traceroute -n -m 4 -w 1 -q 1 198.51.100.53
```

```text
traceroute to 198.51.100.53 (198.51.100.53), 4 hops max, 60 byte packets
 1  192.168.0.254  0.031 ms
 2  *
 3  *
 4  *
```

1台目の `r1` までは返事がありますが、2台目から先が返ってきません。`r1` の次は `r2` なので、`r2` を調べます。`sysctl net.ipv4.ip_forward` が `0` になっています。

**原因**：`r2` が転送しなくなっていた（第5章）。転送しないルータは、TTL を減らす処理もしないので、`traceroute` の「寿命切れ」の知らせも返しません。
:::

### 問3

```bash
sudo ip netns exec r1 nft flush ruleset
sudo ip netns exec pc1 curl http://www.example.test/
```

:::details 答え
症状は問2とまったく同じです。`curl` は10秒ほどで `Could not resolve host`、`traceroute` は2台目から先が `*` になります。

`r2` の `ip_forward` は `1` のままです。では、`r1` と `r2` の間で、何が流れているかを見てみます。ターミナルを2つ使います。

```bash
sudo ip netns exec r1 tcpdump -n -i r1-wan udp port 53
```

```bash
sudo ip netns exec pc1 dig +short www.example.test
```

```text
IP 192.168.0.1.37311 > 198.51.100.53.53: 34711+ [1au] A? www.example.test. (57)
IP 192.168.0.1.51398 > 198.51.100.53.53: 34711+ [1au] A? www.example.test. (57)
IP 192.168.0.1.39739 > 198.51.100.53.53: 34711+ [1au] A? www.example.test. (57)
```

（時刻は省いています。`dig` は返事が来ないので、5秒おきに3回聞き直しています。）

送信元が **`192.168.0.1` のまま**、インターネット側に出ていっています。`r1` の NAT が効いていません（第9章）。`r1` の `nft list ruleset` を見ると、ルールが空になっています。

**原因**：NAT の設定が消え、プライベートアドレスのまま外に出ていた。`r2` には `192.168.0.0/24` への帰り道が無いので、返事が返ってこない。**症状がまったく同じでも、原因は違うことがある**。`tcpdump` で「実際に何が流れているか」を見ると、区別できます。
:::

### 問4

```bash
sudo pkill dnsmasq
sudo ip netns exec pc1 curl http://www.example.test/
```

:::details 答え
`curl` は**すぐに** `Could not resolve host` で終わります。問1〜3のように待たされません。

DNS サーバへの `ping` は通ります。`traceroute` も最後まで届きます。

```bash
sudo ip netns exec pc1 ping -c 1 198.51.100.53
```

```text
64 bytes from 198.51.100.53: icmp_seq=1 ttl=62 time=0.024 ms
```

機器までは届いているので、`dig` で DNS そのものに聞いてみます。

```bash
sudo ip netns exec pc1 dig www.example.test
```

```text
;; communications error to 198.51.100.53#53: timed out
;; communications error to 198.51.100.53#53: connection refused
;; communications error to 198.51.100.53#53: connection refused
;; no servers could be reached
```

`connection refused`、つまり 53 番で誰も待ち受けていません（第7・10章）。`dns1` の中で `ss -ulnp` を見ると、dnsmasq がいません。

**原因**：DNS サーバのプログラムが止まっていた。すぐにエラーになったのは、`dns1` が「53 番には誰もいない」とすぐに返事をしたから。問1〜3では、返事が何も来ないので時間切れまで待たされていました。
:::

### 問5

```bash
sudo pkill -f "http.server 80 --bind 198.51.100.80"
sudo ip netns exec pc1 curl http://www.example.test/
```

:::details 答え
```text
curl: (7) Failed to connect to www.example.test port 80 after 0 ms: Couldn't connect to server
```

`Could not resolve host` ではないので、名前解決は成功しています（`dig +short www.example.test` で `198.51.100.80` が返ります）。`curl -v` で見ると `Connection refused` です。

**原因**：Web サーバのプログラムが止まっていた（第8・11章）。`web1` の中で `ss -tlnp` を見ると、80 番で誰も待ち受けていません。
:::

### 総合問題から分かること

| 問 | `curl` のエラー | 待たされたか | 本当の原因 |
|---|---|---|---|
| 1 | `Could not resolve host` | 約5秒 | `r1` の道案内 |
| 2 | `Could not resolve host` | 約10秒 | `r2` の転送設定 |
| 3 | `Could not resolve host` | 約10秒 | `r1` の NAT |
| 4 | `Could not resolve host` | すぐ | DNS サーバのプログラム |
| 5 | `Connection refused` | すぐ | Web サーバのプログラム |

- **同じエラーメッセージでも、原因はいろいろ**。エラーメッセージは「どの段階で失敗したか」を教えてくれるが、「なぜか」までは教えてくれない
- **すぐに失敗したか、待たされたか**も手がかりになる。すぐなら誰かが「だめ」と返事をしている。待たされたら、どこかで黙って消えている
- 「どこまで届いているか」を、`ping` → `traceroute` → `tcpdump` と順に確かめれば、壊れている場所に近づける

## 12-6. 片付け

```bash
sudo bash ~/netlab-clean.sh
rm -r ~/cap
```

スクリプト（`~/netlab-build.sh` と `~/netlab-clean.sh`）は、もう一度遊びたくなったときのために残しておいてもかまいません。

## まとめ

- `curl` を1回打つだけで、**ARP → DNS（UDP）→ NAT → ルーティング → TCP の3ウェイハンドシェイク → HTTP → FIN** が順に起きている
- **MAC アドレスはケーブルごとに付け替えられ、IP アドレスは NAT 以外では変わらない**
- サーバからは、家の中の PC のプライベートアドレスは見えない
- 同じエラーでも原因はさまざま。**エラーの種類・待たされたか・どこまで届いているか**を手がかりに、下の層から順に切り分ける
