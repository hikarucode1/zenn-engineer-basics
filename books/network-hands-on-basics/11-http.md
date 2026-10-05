---
title: "HTTP"
---

## この章のゴール

いよいよ Web です。ブラウザで URL を開くと、裏では **HTTP**（HyperText Transfer Protocol）という決まりに従って、Web サーバとやり取りしています。

この章では、第10章のネットワークの `web1` に Web サーバを立て、次のことを確かめます。

- HTTP は、**TCP の上で、人間が読める文字をやり取りする**だけのもの。`nc` で手打ちもできる
- `curl -v` を使うと、**DNS → TCP 接続 → HTTP のやり取り**の順に何が起きているかが全部見える
- 「Web ページが開かない」ときのエラーは、**どの段階で失敗したか**によって違う。エラーを見れば、調べる場所が分かる

## 11-0. 準備 —— 第10章のネットワークを作り直す

第10章と同じく、スイッチ `sw1` に `pc1`・`dns1`・`web1` をつなぎます。

```bash
sudo ip netns add sw1
sudo ip netns exec sw1 ip link add br0 type bridge
sudo ip netns exec sw1 ip link set br0 up
```

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

DNS サーバを立て、`pc1` に教えます。この章では DNS のログは見ないので、`--no-daemon` を付けずに**裏で動かします**（ターミナルがすぐ戻ってきます）。

```bash
sudo ip netns exec dns1 dnsmasq --no-resolv --no-hosts --bind-interfaces --listen-address=192.168.0.53 --local=/example.test/ --local-ttl=300 --host-record=www.example.test,192.168.0.80
sudo mkdir -p /etc/netns/pc1
echo 'nameserver 192.168.0.53' | sudo tee /etc/netns/pc1/resolv.conf
```

## 11-1. Web サーバを立てる

Web サーバが返すページを作ります。

```bash
mkdir -p /tmp/www
echo '<h1>Hello from web1</h1>' > /tmp/www/index.html
```

**ターミナル2**で、`web1` の中で Web サーバを起動します。Python に付いている簡単な Web サーバを使います。

```bash
sudo ip netns exec web1 python3 -m http.server 80 --bind 192.168.0.80 --directory /tmp/www
```

「`192.168.0.80` の **80 番**で待ち受け、`/tmp/www` の中のファイルを返す」という意味です。80 番は、第7章の表にあった HTTP のウェルノウンポートです。

別のターミナルで、TCP の 80 番で待ち受けていることを確かめます。

```bash
sudo ip netns exec web1 ss -tlnp
```

```text
State  Recv-Q Send-Q Local Address:Port Peer Address:PortProcess
LISTEN 0      5       192.168.0.80:80        0.0.0.0:*    users:(("python3",pid=1863,fd=3))
```

## 11-2. `curl` でページを取ってくる

ブラウザの代わりに、コマンドで Web ページを取ってくる **`curl`** を使います。

```bash
sudo ip netns exec pc1 curl http://www.example.test/
```

```text
<h1>Hello from web1</h1>
```

`web1` に置いたページの中身が返ってきました。ブラウザなら、これを見出しとして表示するところです。

ターミナル2（Web サーバ）には、アクセスの記録（**アクセスログ**）が1行出ます。

```text
192.168.0.1 - - [05/Oct/2026 20:57:01] "GET / HTTP/1.1" 200 -
```

「`192.168.0.1` から `GET /` という要求が来て、`200`（成功）を返した」という記録です。

URL の各部分には、ここまでの章で学んだことが詰まっています。

| 部分 | 意味 | 関係する章 |
|---|---|---|
| `http://` | HTTP で話す。ポート番号を書かなければ **80 番** | 第7章 |
| `www.example.test` | 相手の名前。DNS で IP アドレスに変換する | 第10章 |
| `/` | サーバの中の、どのページか（**パス**） | この章 |

## 11-3. `curl -v` で全部見る

`-v`（verbose、詳しく表示）を付けると、`curl` が裏でやっていることが全部表示されます。

```bash
sudo ip netns exec pc1 curl -v http://www.example.test/
```

```text
* Host www.example.test:80 was resolved.
* IPv6: (none)
* IPv4: 192.168.0.80
*   Trying 192.168.0.80:80...
* Connected to www.example.test (192.168.0.80) port 80
> GET / HTTP/1.1
> Host: www.example.test
> User-Agent: curl/8.5.0
> Accept: */*
>
* HTTP 1.0, assume close after body
< HTTP/1.0 200 OK
< Server: SimpleHTTP/0.6 Python/3.12.3
< Date: Mon, 05 Oct 2026 11:57:01 GMT
< Content-type: text/html
< Content-Length: 25
< Last-Modified: Mon, 05 Oct 2026 11:56:59 GMT
<
<h1>Hello from web1</h1>
* Closing connection
```

行の先頭の記号で、3種類に分かれています。

| 記号 | 意味 |
|---|---|
| `*` | `curl` 自身の動き（説明） |
| `>` | `curl` がサーバに**送った**文字（**リクエスト**） |
| `<` | サーバから**返ってきた**文字（**レスポンス**） |

上から順に、ここまでの章の内容がそのまま並んでいます。

1. **名前解決（第10章）**：`www.example.test` を DNS で引き、`192.168.0.80` を得た
2. **TCP 接続（第8章）**：`192.168.0.80` の 80 番に接続した（`Connected`）。裏では3ウェイハンドシェイクが行われている
3. **HTTP リクエスト（この章）**：`>` の行を送った
4. **HTTP レスポンス（この章）**：`<` の行と、ページの中身が返ってきた
5. **切断（第8章）**：`Closing connection`

### リクエストを読む

```text
GET / HTTP/1.1
Host: www.example.test
User-Agent: curl/8.5.0
Accept: */*
（空行）
```

| 行 | 意味 |
|---|---|
| `GET / HTTP/1.1` | **リクエスト行**。「`/` のページを**ください**（`GET`）。HTTP/1.1 で話します」 |
| `Host: www.example.test` | どの名前のサイトに用があるか。1台のサーバで複数のサイトを動かすときに、これで区別する |
| `User-Agent: curl/8.5.0` | 自分は何者か（ブラウザの種類など） |
| `Accept: */*` | どんな種類のデータでも受け取れる |
| 空行 | **ここでリクエストは終わり**という合図 |

`Host:` のような `名前: 値` の行を**ヘッダ**と呼びます。

### レスポンスを読む

```text
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.12.3
Content-type: text/html
Content-Length: 25
（空行）
<h1>Hello from web1</h1>
```

| 行 | 意味 |
|---|---|
| `HTTP/1.0 200 OK` | **ステータス行**。`200` は「成功」を表す**ステータスコード** |
| `Content-type: text/html` | 中身の種類は HTML |
| `Content-Length: 25` | 中身は 25 バイト |
| 空行 | ここからが**中身（ボディ）** |
| `<h1>Hello from web1</h1>` | ページの中身 |

（Python の簡単なサーバは `HTTP/1.0` で返事をするので、`curl` は `HTTP 1.0, assume close after body`、つまり「中身を送り終えたら接続を閉じる前提で読む」と表示しています。）

## 11-4. HTTP はただの文字 —— 手で打ってみる

11-3 で見たとおり、HTTP のリクエストは**ただの文字の並び**です。だったら、`curl` を使わずに、第8章の `nc` で同じ文字を送っても Web ページが取れるはずです。

```bash
printf 'GET / HTTP/1.0\r\nHost: www.example.test\r\n\r\n' | sudo ip netns exec pc1 nc www.example.test 80
```

`printf` で、リクエスト行・`Host` ヘッダ・空行を作って `nc` に渡しています。`\r\n` は HTTP で決められている改行の書き方です。最後の `\r\n\r\n` が「ヘッダの終わりの改行」と「空行」になります。

```text
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.12.3
Date: Mon, 05 Oct 2026 11:57:01 GMT
Content-type: text/html
Content-Length: 25
Last-Modified: Mon, 05 Oct 2026 11:56:59 GMT

<h1>Hello from web1</h1>
```

**Web サーバから、ちゃんとページが返ってきました**。`curl` もブラウザも、やっていることの基本はこれと同じです。TCP で接続して、決まった形の文字を送り、決まった形の文字を受け取っています。

## 11-5. 観察する —— ケーブルの上の HTTP

`tcpdump -A` で、ケーブルの上を流れる HTTP を見てみます。

**ターミナル3**：

```bash
sudo ip netns exec web1 tcpdump -n -A -i veth-web1 tcp port 80
```

**ターミナル1**（`-s` は、エラー以外の余計な表示をしない指定）：

```bash
sudo ip netns exec pc1 curl -s http://www.example.test/
```

ターミナル3には十数個のパケットが表示されます。全部載せると長いので、中身のあるところだけを抜き出します（時刻と `options` 以降、`-A` で表示される読めない文字は省いています）。

```text
IP 192.168.0.1.44770 > 192.168.0.80.80: Flags [S], ...
IP 192.168.0.80.80 > 192.168.0.1.44770: Flags [S.], ...
IP 192.168.0.1.44770 > 192.168.0.80.80: Flags [.], ...
IP 192.168.0.1.44770 > 192.168.0.80.80: Flags [P.], seq 1:80, ..., length 79: HTTP: GET / HTTP/1.1
GET / HTTP/1.1
Host: www.example.test
User-Agent: curl/8.5.0
Accept: */*

IP 192.168.0.80.80 > 192.168.0.1.44770: Flags [.], ack 80, ...
IP 192.168.0.80.80 > 192.168.0.1.44770: Flags [P.], seq 1:186, ..., length 185: HTTP: HTTP/1.0 200 OK
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.12.3
Date: Mon, 05 Oct 2026 11:57:02 GMT
Content-type: text/html
Content-Length: 25
Last-Modified: Mon, 05 Oct 2026 11:56:59 GMT

IP 192.168.0.80.80 > 192.168.0.1.44770: Flags [P.], seq 186:211, ..., length 25: HTTP
<h1>Hello from web1</h1>
（この後、FIN による切断が続く）
```

- 最初の3行は、第8章で見た **3ウェイハンドシェイク**
- 4つ目のパケット（`P.`、データ）の中身が、**リクエストの文字そのもの**
- その後、レスポンスのヘッダと中身が、**読める文字のまま**流れている

ここで大事なのは、**HTTP は中身を一切暗号化していない**ことです。同じケーブル（や Wi-Fi）の途中で `tcpdump` されれば、どのページを見たか、フォームに何を入力したか、ログインのパスワードまで、すべて読めてしまいます。

これを防ぐのが **HTTPS** です。HTTPS は、HTTP の文字を **TLS** という仕組みで暗号化してから TCP に流します。中でやり取りしている HTTP の形はこの章と同じですが、`tcpdump` では意味のない文字の並びにしか見えなくなります。TLS は応用編で扱います。

## 11-6. ステータスコード

存在しないページを要求してみます。

```bash
sudo ip netns exec pc1 curl -v http://www.example.test/nothing.html
```

```text
* Host www.example.test:80 was resolved.
* IPv6: (none)
* IPv4: 192.168.0.80
*   Trying 192.168.0.80:80...
* Connected to www.example.test (192.168.0.80) port 80
> GET /nothing.html HTTP/1.1
> Host: www.example.test
> User-Agent: curl/8.5.0
> Accept: */*
>
* HTTP 1.0, assume close after body
< HTTP/1.0 404 File not found
< Server: SimpleHTTP/0.6 Python/3.12.3
< Date: Mon, 05 Oct 2026 11:57:06 GMT
< Connection: close
< Content-Type: text/html;charset=utf-8
< Content-Length: 335
<
<!DOCTYPE HTML>
（エラーページの HTML が続く）
```

**`404 File not found`**（ファイルが見つからない）です。注目してほしいのは、**DNS も TCP 接続も成功している**ことです。サーバとの会話はうまくいっていて、そのうえで「そのページはありません」と答えられたのです。

ステータスコードは、百の位で大まかな意味が決まっています。

| コード | 意味 | 例 |
|---|---|---|
| **2xx** | 成功 | `200 OK` |
| **3xx** | 別の場所へ行って（リダイレクト） | `301 Moved Permanently` |
| **4xx** | **頼んだ側（クライアント）の問題** | `403 Forbidden`（見る権限が無い）、`404 Not Found`（無い） |
| **5xx** | **サーバ側の問題** | `500 Internal Server Error`（サーバ内部のエラー）、`503 Service Unavailable`（今は応答できない） |

ステータスコードだけを知りたいときは、次のようにします。`-o /dev/null` は中身を捨てる、`-w '%{http_code}\n'` は最後にステータスコードを表示する指定です。

```bash
sudo ip netns exec pc1 curl -s -o /dev/null -w '%{http_code}\n' http://www.example.test/
```

```text
200
```

## 11-7. 壊してみる —— 「ページが開かない」を切り分ける

この本の総仕上げとして、「Web ページが開かない」をいろいろな原因で起こし、`curl` のエラーを比べます。

### ① 名前を打ち間違えた

```bash
sudo ip netns exec pc1 curl -v http://ww.example.test/
```

```text
* Could not resolve host: ww.example.test
* Closing connection
curl: (6) Could not resolve host: ww.example.test
```

**`Could not resolve host`**（名前を解決できない）。DNS の段階で失敗しているので、TCP 接続は試みてもいません（第10章）。

### ② ポート番号を間違えた

```bash
sudo ip netns exec pc1 curl -v http://www.example.test:8080/
```

```text
* Host www.example.test:8080 was resolved.
* IPv6: (none)
* IPv4: 192.168.0.80
*   Trying 192.168.0.80:8080...
* connect to 192.168.0.80 port 8080 from 192.168.0.1 port 34620 failed: Connection refused
* Failed to connect to www.example.test port 8080 after 0 ms: Couldn't connect to server
* Closing connection
curl: (7) Failed to connect to www.example.test port 8080 after 0 ms: Couldn't connect to server
```

URL の `:8080` で、80 番以外のポートを指定できます。名前解決は成功し、`192.168.0.80` には届きましたが、**`Connection refused`** です。第8章で見た `RST` が返ってきて、8080 番には誰もいないと分かりました。

### ③ Web サーバが止まっている

ターミナル2の Web サーバを **`Ctrl+C` で止めて**から、アクセスします。

```bash
sudo ip netns exec pc1 curl -v http://www.example.test/
```

```text
* Host www.example.test:80 was resolved.
* IPv6: (none)
* IPv4: 192.168.0.80
*   Trying 192.168.0.80:80...
* connect to 192.168.0.80 port 80 from 192.168.0.1 port 53934 failed: Connection refused
* Failed to connect to www.example.test port 80 after 0 ms: Couldn't connect to server
* Closing connection
curl: (7) Failed to connect to www.example.test port 80 after 0 ms: Couldn't connect to server
```

②と同じ **`Connection refused`** です。機器（`web1`）は生きているけれど、80 番で待ち受けているプログラムがいない、という状態です。

確認したら、**ターミナル2で Web サーバを起動し直して**ください。

```bash
sudo ip netns exec web1 python3 -m http.server 80 --bind 192.168.0.80 --directory /tmp/www
```

### ④ ファイアウォールで黙って捨てられている

最後に、`web1` に**ファイアウォール**を設定して、80 番宛ての TCP を**黙って捨てる**ようにします。第9章の nftables で、今度は書き換えではなく「捨てる」ルールを書きます。

```bash
sudo ip netns exec web1 nft add table ip filter
sudo ip netns exec web1 nft add chain ip filter input '{ type filter hook input priority filter ; }'
sudo ip netns exec web1 nft add rule ip filter input tcp dport 80 drop
```

`hook input` は「この機器宛てに入ってきたパケット」にルールを当てる指定、`tcp dport 80 drop` は「宛先ポートが 80 番の TCP なら捨てる（**drop**）」というルールです。

```bash
sudo ip netns exec pc1 curl -v --connect-timeout 5 http://www.example.test/
```

`--connect-timeout 5` は「接続を5秒待ってだめならあきらめる」という指定です。付けないと、`curl` はかなり長い間（第8章で見た再送を繰り返しながら）待ち続けます。

```text
* Host www.example.test:80 was resolved.
* IPv6: (none)
* IPv4: 192.168.0.80
*   Trying 192.168.0.80:80...
* ipv4 connect timeout after 5000ms, move on!
* Failed to connect to www.example.test port 80 after 5002 ms: Timeout was reached
* Closing connection
curl: (28) Failed to connect to www.example.test port 80 after 5002 ms: Timeout was reached
```

**`Timeout was reached`**（時間切れ）です。`Trying` の後、5秒間何も起きませんでした。`Connection refused` のように「誰もいない」という返事すらなく、`pc1` の「接続していいですか？」（SYN）が**黙って捨てられた**のです。

このとき、`ping` は通ります。

```bash
sudo ip netns exec pc1 ping -c 1 www.example.test
```

```text
PING www.example.test (192.168.0.80) 56(84) bytes of data.
64 bytes from www.example.test (192.168.0.80): icmp_seq=1 ttl=64 time=0.015 ms

--- www.example.test ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.015/0.015/0.015/0.000 ms
```

ファイアウォールが捨てているのは 80 番の TCP だけなので、`ping`（ICMP）は通るのです。「**`ping` は通るのに Web が開かない**」は、現場でよくある状況です。`ping` が通ることは、そのポートに届くことを保証しません。

ファイアウォールの設定を消して、元に戻ることを確かめます。

```bash
sudo ip netns exec web1 nft delete table ip filter
sudo ip netns exec pc1 curl -s http://www.example.test/
```

```text
<h1>Hello from web1</h1>
```

### 切り分けの表

ここまでの結果を、この本で学んだことと合わせて整理します。

- **`(6) Could not resolve host`**
  - 失敗した段階：**DNS**（第10章）。名前を IP アドレスにできない
  - 調べること：名前の綴り、`/etc/resolv.conf`、`dig`
- **`(7) ... Connection refused`**
  - 失敗した段階：**TCP 接続**（第8章）。相手には届いたが、そのポートで誰も待っていない
  - 調べること：ポート番号、サーバのプログラムが動いているか（`ss -tlnp`）
- **`(28) ... Timeout was reached`**
  - 失敗した段階：**TCP 接続**（第8章）。何も返ってこない
  - 調べること：ファイアウォール、経路（`ping`、`traceroute`）、相手の機器が生きているか
- **`404` などの 4xx**
  - 失敗した段階：**HTTP**（この章）。会話は成功。頼んだものが無い・権限が無い
  - 調べること：URL のパス
- **`500` などの 5xx**
  - 失敗した段階：**HTTP**（この章）。会話は成功。サーバの中で問題が起きた
  - 調べること：サーバのログ

**エラーメッセージは、どこまで進めたかを教えてくれています**。下の段階（DNS → TCP → HTTP）から順に確かめていけば、闇雲に調べずに済みます。

## 11-8. 片付け

ターミナル2の Web サーバを `Ctrl+C` で止めてから、片付けます。

```bash
sudo pkill dnsmasq
sudo ip -all netns delete
sudo rm -r /etc/netns/pc1
rm -r /tmp/www
```

## まとめ

- **HTTP** は、TCP の上で**決まった形の文字**をやり取りする決まり。リクエスト（`GET / HTTP/1.1` + ヘッダ + 空行）を送り、レスポンス（ステータス行 + ヘッダ + 空行 + 中身）を受け取る
- `nc` で HTTP を手打ちしても Web ページが取れる。`curl` もブラウザも基本は同じ
- **`curl -v`** で、名前解決 → TCP 接続 → リクエスト → レスポンスの流れが全部見える
- **ステータスコード**：2xx 成功、3xx リダイレクト、**4xx クライアントの問題**、**5xx サーバの問題**
- HTTP は暗号化されていないので、途中で `tcpdump` されると中身が全部読める。それを防ぐのが **HTTPS**（TLS）
- 「開かない」ときは、`curl` のエラーで段階を切り分ける：**`Could not resolve host`（DNS）→ `Connection refused` / `Timeout`（TCP）→ ステータスコード（HTTP）**
- `ping` が通っても、Web が開くとは限らない

これで、この本で作る部品はすべてそろいました。次の章では総まとめとして、ここまで作ってきたスイッチ・ルータ・NAT・DNS・Web サーバを1つのネットワークに組み上げ、**「ブラウザで URL を開いてからページが表示されるまで」に起きることを、最初から最後まで1本通します**。
