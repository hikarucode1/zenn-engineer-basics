---
title: "DNS"
---

## この章のゴール

ここまで、通信相手はずっと `192.168.0.2` や `203.0.113.100` のような**数字の住所**で指定してきました。でも普段、ブラウザに打ち込むのは `example.com` のような**名前**です。名前を IP アドレスに変換してくれる仕組みが **DNS**（Domain Name System）です。

この章では、自分の DNS サーバを立てて、次のことを確かめます。

- 名前を IP アドレスに変えるとき、PC は **`/etc/hosts`** と **`/etc/resolv.conf`** を見る
- DNS の問い合わせは、第7章で見た **UDP の 53 番**で行われる
- 「名前が無い」「DNS サーバに聞けない」「名前は引けたが住所が間違っている」は、**それぞれ違う症状**になる

## 10-1. 準備 —— 3台をスイッチでつなぐ

第4章と同じく、スイッチ `sw1` に3台をつなぎます。

```text
                 +------------ sw1 (br0) ------------+
                 |                |                  |
               pc1              dns1               web1
           192.168.0.1      192.168.0.53       192.168.0.80
                          （DNS サーバ）     （Web サーバ役）
```

`dns1` には DNS サーバを置きます。`web1` は、第11章で Web サーバにする機器です。この章では「`www.example.test` という名前で呼ばれる機器」として使います。

:::message
`.test` は、テストや説明のために使ってよいと決められている名前の末尾（トップレベルドメイン）です。本物のインターネットには存在しないので、誤って本物のサイトに問い合わせることがありません。
:::

スイッチを作ります。

```bash
sudo ip netns add sw1
sudo ip netns exec sw1 ip link add br0 type bridge
sudo ip netns exec sw1 ip link set br0 up
```

`pc1` をつなぎます。

```bash
sudo ip netns add pc1
sudo ip link add veth-pc1 type veth peer name sw1-pc1
sudo ip link set veth-pc1 netns pc1
sudo ip link set sw1-pc1 netns sw1
sudo ip netns exec sw1 ip link set sw1-pc1 master br0
sudo ip netns exec sw1 ip link set sw1-pc1 up
sudo ip netns exec pc1 ip addr add 192.168.0.1/24 dev veth-pc1
sudo ip netns exec pc1 ip link set veth-pc1 up
```

`dns1` をつなぎます。

```bash
sudo ip netns add dns1
sudo ip link add veth-dns1 type veth peer name sw1-dns1
sudo ip link set veth-dns1 netns dns1
sudo ip link set sw1-dns1 netns sw1
sudo ip netns exec sw1 ip link set sw1-dns1 master br0
sudo ip netns exec sw1 ip link set sw1-dns1 up
sudo ip netns exec dns1 ip addr add 192.168.0.53/24 dev veth-dns1
sudo ip netns exec dns1 ip link set veth-dns1 up
```

`web1` をつなぎます。

```bash
sudo ip netns add web1
sudo ip link add veth-web1 type veth peer name sw1-web1
sudo ip link set veth-web1 netns web1
sudo ip link set sw1-web1 netns sw1
sudo ip netns exec sw1 ip link set sw1-web1 master br0
sudo ip netns exec sw1 ip link set sw1-web1 up
sudo ip netns exec web1 ip addr add 192.168.0.80/24 dev veth-web1
sudo ip netns exec web1 ip link set veth-web1 up
```

数字の住所なら、`pc1` から `web1` に届きます。

```bash
sudo ip netns exec pc1 ping -c 1 192.168.0.80
```

```text
PING 192.168.0.80 (192.168.0.80) 56(84) bytes of data.
64 bytes from 192.168.0.80: icmp_seq=1 ttl=64 time=0.038 ms

--- 192.168.0.80 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.038/0.038/0.038/0.000 ms
```

## 10-2. 名前では届かない

では、名前で指定するとどうなるでしょうか。

```bash
sudo ip netns exec pc1 ping -c 1 www.example.test
```

```text
ping: www.example.test: Temporary failure in name resolution
```

**`Temporary failure in name resolution`**（名前解決に一時的に失敗した）です。`ping` は、パケットを1つも送る前に、**名前を IP アドレスに変換する段階**（**名前解決**）で失敗しています。

Linux は名前解決のとき、次の2つのファイルを見ます。

| ファイル | 中身 |
|---|---|
| `/etc/hosts` | 名前と IP アドレスの対応表を、直接書いておくファイル |
| `/etc/resolv.conf` | 名前を問い合わせる **DNS サーバの住所** |

`pc1` の中から `/etc/resolv.conf` を見てみます。

```bash
sudo ip netns exec pc1 cat /etc/resolv.conf
```

```text
（コメント行は省略）
nameserver 127.0.0.53
options edns0 trust-ad
search multipass
```

`nameserver 127.0.0.53` は、Ubuntu が普段使っている名前解決の窓口（systemd-resolved）の住所です。netns が分けているのは**ネットワークだけ**で、ファイルは元の Ubuntu と共有しているので、`pc1` も同じ設定を読んでいます。ところが `pc1` のネットワークからは、その窓口に届きません。だから失敗したのです。

（`search` の行は環境によって違います。）

## 10-3. いちばん素朴な方法 —— `/etc/hosts`

DNS を使わない方法から試します。`/etc/hosts` に「`www.example.test` は `192.168.0.80`」と書けば、名前解決できるはずです。

ただし、元の Ubuntu の `/etc/hosts` を書き換えると、`pc1` 以外にも影響してしまいます。`ip netns exec` には、**`/etc/netns/<netns名>/` に置いたファイルを、その netns の中では `/etc/` のファイルの代わりに見せる**機能があるので、これを使います。

```bash
sudo mkdir -p /etc/netns/pc1
echo '192.168.0.80 www.example.test' | sudo tee /etc/netns/pc1/hosts
sudo ip netns exec pc1 cat /etc/hosts
```

```text
192.168.0.80 www.example.test
192.168.0.80 www.example.test
```

1行目は `tee` の表示、2行目が `pc1` の中から見た `/etc/hosts` です。`pc1` には、今書いた1行だけの `/etc/hosts` が見えています。

```bash
sudo ip netns exec pc1 ping -c 1 www.example.test
```

```text
PING www.example.test (192.168.0.80) 56(84) bytes of data.
64 bytes from www.example.test (192.168.0.80): icmp_seq=1 ttl=64 time=0.012 ms

--- www.example.test ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.012/0.012/0.012/0.000 ms
```

名前で届きました。`PING www.example.test (192.168.0.80)` の括弧の中が、名前解決の結果です。

`/etc/hosts` は簡単ですが、**名前を使う機器すべてに同じ行を書く**必要があります。`web1` の住所が変わったら、全部の機器を書き換えなければいけません。インターネット全体でこれをやるのは無理です。昔は実際に、1つの大きな hosts ファイルを全員で配っていましたが、規模が大きくなって破綻し、DNS が作られました。

`/etc/hosts` は消しておきます。

```bash
sudo rm /etc/netns/pc1/hosts
```

## 10-4. DNS サーバを立てる

`dns1` に DNS サーバを立てます。小さくて設定が簡単な **dnsmasq** を使います。あわせて、DNS に直接問い合わせるコマンド **`dig`** も入れます（Ubuntu 24.04 には最初から入っていることが多いです）。

```bash
sudo apt install -y dnsmasq-base bind9-dnsutils
```

:::message
`dnsmasq`（`-base` の付かないほう）を入れると、元の Ubuntu で自動的に起動しようとして、Ubuntu が普段使っている名前解決の窓口とぶつかります。ここでは自動起動しない **`dnsmasq-base`** を入れて、`dns1` の中で手動で起動します。
:::

**ターミナル2**で、`dns1` の中で dnsmasq を起動します。

```bash
sudo ip netns exec dns1 dnsmasq --no-daemon --no-resolv --no-hosts --bind-interfaces --listen-address=192.168.0.53 --local=/example.test/ --local-ttl=300 --host-record=www.example.test,192.168.0.80 --host-record=dns.example.test,192.168.0.53 --log-queries
```

長いので、オプションを分けて説明します。

- `--no-daemon`：裏に回らず、このターミナルで動き続ける。ログもここに出る
- `--no-resolv`、`--no-hosts`：元の Ubuntu の `/etc/resolv.conf` と `/etc/hosts` を読まない
- `--bind-interfaces`、`--listen-address=192.168.0.53`：`192.168.0.53` 宛ての問い合わせだけを受け付ける
- `--local=/example.test/`：`example.test` の名前は自分が全部知っている。知らない名前は「無い」と答える
- `--local-ttl=300`：答えを「300 秒間は覚えておいてよい」と伝える（後述）
- `--host-record=www.example.test,192.168.0.80`：`www.example.test` は `192.168.0.80`、という対応を登録する
- `--log-queries`：問い合わせが来るたびにログを出す

ターミナル2に、次のように表示されて止まります。

```text
dnsmasq: started, version 2.91 cachesize 150
dnsmasq: compile time options: IPv6 GNU-getopt DBus no-UBus i18n IDN2 DHCP DHCPv6 no-Lua TFTP conntrack ipset nftset auth DNSSEC loop-detect inotify dumpfile
dnsmasq: warning: no upstream servers configured
dnsmasq: using only locally-known addresses for example.test
dnsmasq: cleared cache
```

`no upstream servers configured`（問い合わせを回す先がない）という警告は、この章では気にしなくて大丈夫です（10-10 で説明します）。

別のターミナルで、`dns1` が UDP の 53 番で待ち受けていることを確かめます。

```bash
sudo ip netns exec dns1 ss -ulnp
```

```text
State  Recv-Q Send-Q Local Address:Port Peer Address:PortProcess
UNCONN 0      0       192.168.0.53:53        0.0.0.0:*    users:(("dnsmasq",pid=1944,fd=4))
```

## 10-5. `dig` で問い合わせる

`pc1` から、`dns1` に直接問い合わせます。`@192.168.0.53` は「この DNS サーバに聞く」という指定です。

```bash
sudo ip netns exec pc1 dig @192.168.0.53 www.example.test
```

```text

; <<>> DiG 9.18.39-0ubuntu0.24.04.7-Ubuntu <<>> @192.168.0.53 www.example.test
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 58610
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;www.example.test.		IN	A

;; ANSWER SECTION:
www.example.test.	300	IN	A	192.168.0.80

;; Query time: 0 msec
;; SERVER: 192.168.0.53#53(192.168.0.53) (UDP)
;; WHEN: Mon Oct 05 20:47:10 JST 2026
;; MSG SIZE  rcvd: 61

```

長いですが、見るべきところは4か所です。

| 場所 | 読み方 |
|---|---|
| `status:` | `NOERROR`：問い合わせは成功した |
| `QUESTION SECTION` | 「`www.example.test` の **A レコード**（IPv4 アドレス）は？」と聞いた |
| `ANSWER SECTION` | 答えは `192.168.0.80`。**300 秒間は覚えておいてよい** |
| `SERVER:` | `192.168.0.53` の **53 番に UDP で**聞いた |

ANSWER の行の `300` は **TTL**（Time To Live）です。第5章の IP パケットの TTL とは別物で、こちらは「この答えを**何秒間覚えておいて（キャッシュして）よいか**」を表します。毎回 DNS サーバに聞きに行かずに済むようにするための工夫です。

答えだけが欲しいときは、`+short` を付けます。

```bash
sudo ip netns exec pc1 dig @192.168.0.53 +short www.example.test
```

```text
192.168.0.80
```

ターミナル2（dnsmasq）には、問い合わせのログが出ています。

```text
dnsmasq: query[A] www.example.test from 192.168.0.1
dnsmasq: config www.example.test is 192.168.0.80
```

「`192.168.0.1` から `www.example.test` の A レコードを聞かれ、設定どおり `192.168.0.80` と答えた」という記録です。

## 10-6. 観察する —— DNS は UDP の 53 番

ケーブルの上の DNS を見てみます。

**ターミナル3**：

```bash
sudo ip netns exec dns1 tcpdump -n -i veth-dns1 port 53
```

`port 53` は「送信元か宛先が 53 番のものだけ」という絞り込みです。

**ターミナル1**：

```bash
sudo ip netns exec pc1 dig @192.168.0.53 +short dns.example.test
```

```text
192.168.0.53
```

ターミナル3（時刻は省いています）：

```text
IP 192.168.0.1.56882 > 192.168.0.53.53: 46297+ [1au] A? dns.example.test. (57)
IP 192.168.0.53.53 > 192.168.0.1.56882: 46297* 1/0/1 A 192.168.0.53 (61)
```

- 1行目：`pc1` の 56882 番から、`dns1` の **53 番**へ。`A? dns.example.test.` は「`dns.example.test` の A レコードは？」という質問
- 2行目：`dns1` の 53 番から、`pc1` の 56882 番へ。`A 192.168.0.53` が答え

第7章で見た UDP の形そのものです。質問1つ、答え1つ。3ウェイハンドシェイクも無く、**たった2つのパケットで終わります**。名前解決は Web ページを開くたびに何度も行われるので、軽い UDP が向いているのです。答えが返ってこなければ、もう一度聞けば済みます。

ターミナル3は `Ctrl+C` で止めておきます。

## 10-7. `pc1` に DNS サーバを教える

`dig @...` で直接聞くのではなく、`ping` やブラウザが普通に名前を使えるように、`pc1` の `/etc/resolv.conf` に DNS サーバを書きます。10-3 と同じく、`/etc/netns/pc1/` に置きます。

```bash
echo 'nameserver 192.168.0.53' | sudo tee /etc/netns/pc1/resolv.conf
sudo ip netns exec pc1 ping -c 1 www.example.test
```

```text
nameserver 192.168.0.53
PING www.example.test (192.168.0.80) 56(84) bytes of data.
64 bytes from www.example.test (192.168.0.80): icmp_seq=1 ttl=64 time=0.011 ms

--- www.example.test ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.011/0.011/0.011/0.000 ms
```

（1行目は `tee` の表示です。）

`/etc/hosts` には何も書いていないのに、名前で届きました。`ping` の中では次のことが起きています。

1. `/etc/hosts` を見る → `www.example.test` は書かれていない
2. `/etc/resolv.conf` を見る → DNS サーバは `192.168.0.53`
3. `192.168.0.53` の 53 番に UDP で「`www.example.test` の A レコードは？」と聞く
4. `192.168.0.80` という答えが返ってくる
5. `192.168.0.80` に `ping` を送る

:::message
このときターミナル2（dnsmasq）のログを見ると、`ping` は `query[A]` のほかに、`query[AAAA]`（IPv6 アドレスは？）や `query[PTR] 80.0.168.192.in-addr.arpa`（`192.168.0.80` の名前は？という**逆引き**）も聞いています。`ping` が結果の行に `from www.example.test (192.168.0.80)` と名前を添えて表示するのは、この逆引きの答えを使っているからです。

```text
dnsmasq: query[A] www.example.test from 192.168.0.1
dnsmasq: config www.example.test is 192.168.0.80
dnsmasq: query[AAAA] www.example.test from 192.168.0.1
dnsmasq: config www.example.test is NODATA-IPv6
dnsmasq: query[PTR] 80.0.168.192.in-addr.arpa from 192.168.0.1
dnsmasq: config 192.168.0.80 is www.example.test
```
:::

`dig` も、`@` を付けなければ `/etc/resolv.conf` の DNS サーバに聞きます。

```bash
sudo ip netns exec pc1 dig +short www.example.test
```

```text
192.168.0.80
```

`web1` の住所が変わっても、`dns1` の登録を1か所直すだけで、全員が新しい住所を引けるようになります。これが hosts ファイルとの大きな違いです。

## 10-8. 壊してみる① —— 名前を打ち間違えたら

`www` を `ww` と打ち間違えてみます。

```bash
sudo ip netns exec pc1 dig ww.example.test
```

```text

; <<>> DiG 9.18.39-0ubuntu0.24.04.7-Ubuntu <<>> ww.example.test
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 1723
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;ww.example.test.		IN	A

;; Query time: 0 msec
;; SERVER: 192.168.0.53#53(192.168.0.53) (UDP)
;; WHEN: Mon Oct 05 20:47:45 JST 2026
;; MSG SIZE  rcvd: 44

```

`status:` が **`NXDOMAIN`**（Non-Existent Domain、その名前は存在しない）になり、`ANSWER: 0` です。DNS サーバにはちゃんと聞けていて、そのうえで「そんな名前は無い」と言われました。

`ping` ではこう表示されます。

```bash
sudo ip netns exec pc1 ping -c 1 ww.example.test
```

```text
ping: ww.example.test: Name or service not known
```

**`Name or service not known`**（その名前は知られていない）です。10-2 の `Temporary failure in name resolution` とは**メッセージが違う**ことに注目してください。

## 10-9. 壊してみる② —— DNS サーバが止まったら

ターミナル2の dnsmasq を **`Ctrl+C` で止めて**から、もう一度聞きます。

```bash
sudo ip netns exec pc1 dig www.example.test
```

```text
;; communications error to 192.168.0.53#53: connection refused
;; communications error to 192.168.0.53#53: connection refused
;; communications error to 192.168.0.53#53: connection refused

; <<>> DiG 9.18.39-0ubuntu0.24.04.7-Ubuntu <<>> www.example.test
;; global options: +cmd
;; no servers could be reached
```

**`no servers could be reached`**（どのサーバにも届かなかった）です。`connection refused` は、第7章の 7-7 で見た `port unreachable` が返ってきたことを表しています。`dns1` の機器自体はいるけれど、53 番で誰も待ち受けていないのです。`dig` は3回試して、あきらめました。

```bash
sudo ip netns exec pc1 ping -c 1 www.example.test
```

```text
ping: www.example.test: Temporary failure in name resolution
```

さっきまで届いていた `www.example.test` に、`Temporary failure in name resolution` が出るようになりました。

| `ping` のメッセージ | 意味 | 調べること |
|---|---|---|
| `Name or service not known` | DNS サーバに聞けた。**その名前は無い**と言われた（`dig` では `NXDOMAIN`） | 名前の綴り、DNS サーバへの登録 |
| `Temporary failure in name resolution` | **DNS サーバに聞けなかった**（`dig` では `no servers could be reached` など） | `/etc/resolv.conf`、DNS サーバが動いているか、そこまで届くか |

この2つを区別できるだけで、「名前の問題」なのか「DNS サーバへの通信の問題」なのかが分かります。

## 10-10. 壊してみる③ —— 名前は引けたが、住所が間違っていたら

最後は、DNS サーバの**登録自体が間違っている**場合です。ターミナル2で、`www.example.test` の住所を、存在しない `192.168.0.81` にして dnsmasq を起動し直します。

```bash
sudo ip netns exec dns1 dnsmasq --no-daemon --no-resolv --no-hosts --bind-interfaces --listen-address=192.168.0.53 --local=/example.test/ --local-ttl=300 --host-record=www.example.test,192.168.0.81 --host-record=dns.example.test,192.168.0.53 --log-queries
```

```bash
sudo ip netns exec pc1 ping -c 2 -W 1 www.example.test
```

```text
PING www.example.test (192.168.0.81) 56(84) bytes of data.

--- www.example.test ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1028ms

```

今度は名前解決のエラーは出ず、`100% packet loss` です。一見、ネットワークの問題に見えます。でも1行目の括弧の中をよく見ると、**`192.168.0.81`** になっています。`web1` は `192.168.0.80` なので、そもそも違う相手に送っていたのです。

```bash
sudo ip netns exec pc1 dig +short www.example.test
```

```text
192.168.0.81
```

「名前で届かない」ときは、**まず名前がどの住所に変換されているかを確かめる**のが鉄則です。`ping` の1行目の括弧か、`dig +short` で確認できます。ここを飛ばして、ルーティングやファイアウォールを何時間も調べてしまうのは、現場でとてもよくある失敗です。

確認したら、ターミナル2の dnsmasq を `Ctrl+C` で止めておきます。

## 10-11. 本物のインターネットの DNS

この章の `dns1` は、`example.test` の名前しか知りません。たとえば `example.com` を聞くと、`status: REFUSED`（答えを拒否）が返ります。

では、本物のインターネットでは、1台の DNS サーバが世界中の名前を全部知っているのでしょうか。そうではありません。DNS は、名前の右から順に**担当を分けて**います。

```text
www.example.com. の場合

  .（ルート）       「.com のことは、.com の担当サーバに聞いて」
     └ com.         「example.com のことは、example.com の担当サーバに聞いて」
         └ example.com.   「www.example.com の IP アドレスはこれです」
```

家の PC の `/etc/resolv.conf` に書かれている DNS サーバ（プロバイダや家庭用ルータのもの）は、自分では答えを持っていません。PC の代わりに、ルート → `.com` → `example.com` と**順番に聞いて回り**、答えを PC に返してくれます。このような DNS サーバを**フルリゾルバ**（キャッシュ DNS サーバ）と呼びます。一方、`example.com` の担当サーバのように、自分の名前の答えを持っているものを**権威 DNS サーバ**と呼びます。この章の `dns1` は、`example.test` の権威 DNS サーバにあたります。

フルリゾルバは、聞いて回った答えを TTL の間だけ覚えておきます。だから、同じ名前を2回目に引くときは速くなります。10-4 の警告 `no upstream servers configured` は、「自分の知らない名前を聞きに行く先が無い」という意味でした。

## 10-12. 片付け

ターミナル2の dnsmasq を止めていなければ `Ctrl+C` で止めてから、片付けます。この章では `/etc/netns/pc1/` も作ったので、それも消します。

```bash
sudo ip -all netns delete
sudo rm -r /etc/netns/pc1
```

## まとめ

- **DNS** は、名前を IP アドレスに変換する仕組み。名前解決のとき、Linux は **`/etc/hosts`** を見てから、**`/etc/resolv.conf`** に書かれた DNS サーバに問い合わせる
- `/etc/hosts` は簡単だが、全部の機器に書く必要がある。DNS なら、サーバ1か所の登録で全員が引ける
- **`dig`** で DNS に直接問い合わせられる。`status`、`ANSWER SECTION`、`SERVER` を見る。答えだけなら `+short`
- DNS の問い合わせは、主に **UDP の 53 番**で、質問1つと答え1つのパケットで終わる
- レコードの **TTL** は、答えを何秒キャッシュしてよいか
- **`Name or service not known`**（NXDOMAIN）は「名前が無い」、**`Temporary failure in name resolution`** は「DNS サーバに聞けない」。調べる場所が違う
- 名前で届かないときは、**まず `dig +short` で、どの住所に変換されているかを確かめる**

これで、名前から住所を引き、住所に向けて TCP で接続する準備がそろいました。次の章では、`web1` に Web サーバを立て、ブラウザが Web ページを取ってくるときの **HTTP** の中身を読みます。
