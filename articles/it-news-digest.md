---
title: "毎朝5時に「自分専用のITニュース1ページ」を作る ― 依存ゼロPythonとRedditの403との戦い"
emoji: "📰"
type: "tech"
topics: ["python", "rss", "cron", "個人開発", "hackernews"]
published: false
---

## TL;DR

Hacker News / Reddit / はてなブックマークから毎朝5時に記事を集めて、**1枚の静的HTML**にまとめるツールを作りました。今どのトピックが盛り上がっているかを、サイトを巡回せずに1ページで把握できます。

- **依存ゼロ**（Python 標準ライブラリのみ、`pip install` 不要）— cron に置くだけで動く
- **Reddit の 403 / 429 を、マルチレディット RSS 1リクエスト**で回避
- **部分失敗を許容する設計** — 1ソースが落ちても他は出す、全滅した日は前日のページを壊さない
- 生成物は静的 HTML なので、`python3 -m http.server` と Tailscale だけでスマホからも読める

コードは Python 4ファイル・約550行。以下、実装で判断が要った4点を実コードで解説します。

---

## なぜ作ったか

技術トレンドを追うのに、毎朝 Hacker News を開き、いくつかのサブレディットを回り、はてブのテクノロジーを見る、という巡回をしていました。この「巡回」が続かない。1サイト見て満足して終わる日が増えます。

RSS リーダーも試しましたが、今度は**未読が溜まる**。私が欲しかったのは「全部読む」体験ではなく、**その日に何が話題だったかを30秒で俯瞰する**体験でした。未読管理も同期もいらない。毎朝、上書きされる1ページだけあればいい。

そこで要件をこう決めました。

| 要件 | 理由 |
| --- | --- |
| 出力は静的 HTML 1枚 | サーバーサイドの状態を持たない。壊れても再生成するだけ |
| 未読管理をしない | 「読み残し」というプレッシャーを構造的に作らない |
| 外部ライブラリを使わない | 自分用ツールを数年後に動かすとき、依存が腐っているのが一番つらい |
| 1ソース落ちても動く | 3サイト中1つの API 変更で毎朝が止まるのは許容できない |

構成はシンプルです。

```
config.py  取得元・件数・保持日数の設定
fetch.py   各サイトから収集 → data/YYYY-MM-DD.json
build.py   data/*.json → public/ 以下の HTML
run.sh     cron から呼ばれる入口（fetch → build）
```

`fetch`（収集）と `build`（生成）を分けたのは、**HTML の見た目を変えるたびにサイトを叩き直したくない**からです。JSON が残っていれば `python3 build.py` だけで過去60日分のページを作り直せます。CSS をいじっていた日は、これに何度も助けられました。

---

## 1. 依存ゼロで書く ― `urllib` + `ElementTree` で足りる

`requests` も `feedparser` も使っていません。標準ライブラリの `urllib.request` と `xml.etree.ElementTree` だけです。

自分用ツールで一番よくある死に方は、機能不足ではなく**環境の腐敗**です。数年後に venv が壊れている、ライブラリのメジャーバージョンが上がって動かない。`python3 fetch.py` だけで動くなら、その死に方をしません。

代わりに HTTP まわりは自前で書く必要があります。`http_get()` はこうなりました。

```python
USER_AGENT = "Mozilla/5.0 (X11; Linux x86_64) personal-it-news-digest/1.0"

def http_get(url: str, retries: int = 3) -> str:
    req = urllib.request.Request(
        url,
        headers={
            "User-Agent": USER_AGENT,
            "Accept": "application/json, application/rss+xml, text/xml, */*",
            "Accept-Language": "ja,en;q=0.8",
            "Accept-Encoding": "gzip",
        },
    )
    for attempt in range(retries):
        try:
            with urllib.request.urlopen(req, timeout=config.HTTP_TIMEOUT) as res:
                raw = res.read()
                if res.headers.get("Content-Encoding") == "gzip":
                    raw = gzip.decompress(raw)
            return raw.decode("utf-8", errors="replace")
        except urllib.error.HTTPError as exc:
            # レート制限・一時障害のみ待って再試行。404 などは即座に諦める
            if exc.code not in (429, 500, 502, 503) or attempt == retries - 1:
                raise
            try:
                wait = float(exc.headers.get("Retry-After") or 0)
            except ValueError:
                wait = 0
            wait = max(wait, config.RETRY_BASE_WAIT * (attempt + 1))
            print(f"[retry] HTTP {exc.code} {url} — {wait:.0f}s 待機", file=sys.stderr)
            time.sleep(wait)
```

`requests` を捨てて自分で書くと、普段ライブラリが隠してくれている判断が全部見えてきます。

**再試行する条件を絞る。** 429 / 500 / 502 / 503 だけ待って再試行し、404 や 403 は即座に諦めます。「エラーだからとりあえずリトライ」にすると、URL を打ち間違えただけの日に30秒待たされます。**回復する見込みのないエラーを待つのは、ただの遅延**です。

**`Retry-After` を尊重しつつ、下限を自分で持つ。** サーバーが `Retry-After` を返してきたらそれに従いますが、`max(wait, RETRY_BASE_WAIT * (attempt + 1))` で自前の待機時間と比べて長い方を取ります。ヘッダーが無い場合や `0` の場合に即リトライして、相手をさらに叩かないためです。試行回数に比例して伸ばすので 10秒 → 20秒 → 30秒 になります。

**gzip を自分で展開する。** `Accept-Encoding: gzip` を送るなら、レスポンスの展開も自分の責任です。`requests` が透過的にやってくれていた部分で、忘れるとバイナリを `decode()` して盛大に壊れます。

**`errors="replace"` で decode する。** はてブの RSS には稀に不正なバイト列が混ざります。1文字が化けるだけで20件の記事が全部落ちるのは割に合わないので、例外を投げずに置換文字にします。

---

## 2. Reddit の 403 と 429 ― マルチレディット RSS で1リクエストに畳む

ここが一番てこずりました。

まず、**Reddit の未認証 JSON API は現在 403 Blocked を返します**。`www.reddit.com/r/programming/top.json` も、`old.reddit.com` も、`api.reddit.com` も同じです。User-Agent を変えても通りません。

次に RSS（`.rss`）に切り替えました。こちらは通ります。ところが**サブレディットごとに順番に叩くと、数秒間隔を空けても 429 になる**。5サブレディットを回るだけでレート制限に引っかかります。

解決策は、Reddit が元々持っている**マルチレディット**記法でした。`r/a+b+c` と `+` でつなぐと、複数サブレディットの投稿を1つのフィードで返してくれます。

```python
def fetch_reddit_multi(subreddits, fetch_limit):
    """マルチレディット RSS を 1 回取得し、{サブレディット名: [item, ...]} を返す。"""
    joined = "+".join(subreddits)
    url = f"https://www.reddit.com/r/{joined}/top/.rss?t=day&limit={fetch_limit}"
    root = ET.fromstring(http_get(url))

    grouped = {sub: [] for sub in subreddits}
    # term は大文字小文字が設定どおりとは限らないので小文字で引けるようにしておく
    lookup = {sub.lower(): sub for sub in subreddits}

    for entry in root.findall(f"{ATOM}entry"):
        category_el = entry.find(f"{ATOM}category")
        term = (category_el.get("term") or "") if category_el is not None else ""
        sub = lookup.get(term.lower())
        if sub is None:
            continue
        grouped[sub].append(...)
    return grouped
```

**5リクエストが1リクエストになり、429 が消えました。** どのサブレディット由来かは Atom の `<category term="...">` に入っているので、それを見て振り分けます。

`lookup` を小文字キーで作っているのは実際に踏んだ罠です。設定には `MachineLearning` と書いていますが、フィードが返す `term` の大文字小文字が設定と一致する保証がありません。素直に `grouped[term]` で引くと、あるサブレディットだけ**エラーも出さずに常に0件**になります。落ちてくれないバグは見つけるのに時間がかかります。

### 外部リンク投稿の URL を取り出す

Reddit の RSS の `<link href="...">` は**常に Reddit のパーマリンク**です。「元記事」を読みたいのに Reddit のコメントページに飛ばされる。実際の記事 URL は `<content>` の HTML の中に `[link]` というアンカーとして埋まっています。

```python
EXTERNAL_LINK_RE = re.compile(r'<a href="([^"]+)">\s*\[link\]\s*</a>')

# 外部リンク投稿は content 内の [link] が実際の記事 URL。self post には無い
target = permalink
if content_el is not None and content_el.text:
    match = EXTERNAL_LINK_RE.search(html.unescape(content_el.text))
    if match:
        target = html.unescape(match.group(1))
```

正規表現で HTML を解析するのは一般には悪手ですが、ここは**Reddit が生成する固定フォーマットの一箇所**を抜くだけなので許容しました。self post（テキスト投稿）には `[link]` が無いため、その場合は `target` がパーマリンクのまま残ります。これは正しい挙動です — 本文が Reddit にあるのだから、リンク先も Reddit でいい。

`html.unescape()` を2回呼んでいるのは、`<content>` が**エスケープされた HTML** を含んでいるためです。1回目でタグを取り出せる形にし、2回目で URL 内の `&amp;` を `&` に戻します。クエリパラメータ付き URL でこれを忘れると壊れます。

### 妥協した点

この方式には代償があります。

- **Reddit だけスコアとコメント数が出ません。** RSS に含まれていないためです。JSON API が使えれば取れますが、403 なので諦めました。並び順は Reddit の「今日のトップ」順そのままです。
- **人気サブレディットに枠を取られます。** 1つのフィードから100件取って振り分けるので、`r/programming` が大量に入っている日は `r/selfhosted` が設定した6件に届きません。`REDDIT_FETCH_LIMIT = 100` と多めに取ることで緩和していますが、根本解決ではありません。

「1リクエストに畳む」という設計を選んだ時点で、この偏りは受け入れる前提でした。**毎朝確実に何かが出ること**のほうが、各サブレディットの件数が揃うことより大事だと判断しています。

---

## 3. 部分失敗を許容する ― 「1つ落ちても出す」「全滅なら書かない」

3つの外部サイトに依存している以上、どれかが落ちる日は必ず来ます。設計をこう分けました。

**1ソースの失敗は全体を止めない。**

```python
for key, name, run in jobs:
    entry = {"key": key, "name": name, "error": None, "items": []}
    try:
        fetched = run()
    except Exception as exc:  # 1 サイトの失敗で全体を止めない
        entry["error"] = f"{type(exc).__name__}: {exc}"
        print(f"[warn] {name}: {entry['error']}", file=sys.stderr)
    else:
        # 同じ URL が複数ソースに出たら先に取れた方を残す
        for it in fetched:
            if it["url"] in seen_urls:
                continue
            seen_urls.add(it["url"])
            entry["items"].append(it)
    sources.append(entry)
```

エラーを握りつぶすのではなく、**`error` フィールドとして JSON に残します**。build 側はそれを読んで、ページ上部に警告バナーを出します。

```python
failed = [s["name"] for s in data["sources"] if s.get("error")]
if failed:
    banner = (
        f'<p class="banner">取得できなかったソース: {esc("、".join(failed))}'
        "（他のソースは正常に取得済み）</p>"
    )
```

これが効くのは、**「今日は記事が少ないな」と「今日は Reddit が落ちていた」を区別できる**からです。前者ならそういう日だと思って閉じますが、後者なら `logs/run.log` を見に行きます。サイレントに空になるのが一番たちが悪い。

**全滅した日は何も書かない。**

```python
total = sum(len(s["items"]) for s in sources)
if total == 0:
    print("[error] 全ソースの取得に失敗したため保存を中止", file=sys.stderr)
    return 1
```

`run.sh` 側でも、fetch が失敗したら build を実行しません。

```bash
if ! python3 fetch.py; then
  # 収集が全滅した場合でも、前日分の HTML は残したいのでビルドは実行しない
  echo "=== $(date '+%Y-%m-%d %H:%M:%S') 収集に失敗したため中断 ==="
  exit 1
fi
```

ネットワークが死んでいる朝に**空のページで前日分を上書きしてしまう**のが、この手のツールで最悪の失敗です。「昨日のニュース」は「空白」より確実に価値がある。**書けないときは書かない**ほうが安全側です。

`set -uo pipefail` に `-e` を含めていないのも意図的です。`-e` を付けると `python3 fetch.py` が非ゼロを返した瞬間にスクリプトが死に、終了ログを書けません。明示的に `if !` で分岐して、必ずログの最終行を残すようにしています。

### 重複排除

Hacker News とはてブは、同じ記事を同じ日に取り上げることがよくあります。`seen_urls` で先に取れた方を残す単純な仕組みです。

これは**ジョブの実行順が優先順位になる**設計です。`jobs` リストは Hacker News → Reddit → はてブの順なので、重複した記事は Hacker News 側に表示されます。厳密な「どちらが適切か」の判定はしていませんが、自分用ツールでは十分でした。

---

## 4. 配信 ― 静的 HTML を `http.server` と Tailscale で運ぶ

生成物は完全に静的な HTML です。当初は `file://` で開いていましたが、パスの打ち間違いやブラウザ側の制約で開けないことがあり、HTTP 経由を既定にしました。

Linux PC で systemd のユーザーサービスとして `python3 -m http.server` を常駐させています。

```bash
systemctl --user status it-news-web
systemctl --user restart it-news-web
```

さらに **Tailscale** を入れたことで、Mac からも iPhone からも同じページが見られるようになりました。tailnet 内の MagicDNS 名を叩くだけです。外部に公開せず、ポート開放も DDNS も要りません。朝の電車で iPhone から開けるようになって、初めて「毎日見るもの」になりました。

`--bind 0.0.0.0` にしているので localhost・LAN・tailnet のすべてから見えます。tailnet だけに絞りたければ `--bind <tailscale IP>` にできますが、その場合その PC 自身の `localhost:8765` では開けなくなります。この非対称性は最初ハマったので README に書いてあります。

### アーカイブと prune

過去60日分を残しています（`ARCHIVE_KEEP_DAYS = 60`）。fetch 側で古い JSON を消し、build 側で**対応する JSON が無い HTML** を消します。

```python
# data/ から消えた日のアーカイブ HTML を残さない
valid_names.add("index.html")
for stale in ARCHIVE_DIR.glob("*.html"):
    if stale.name not in valid_names:
        stale.unlink()
```

**削除を build 側に寄せた**のがポイントです。「JSON が正」で「HTML は JSON から導出されるもの」と決めると、prune のロジックが1箇所で済みます。fetch 側と build 側の両方で日付計算をして削除すると、片方だけずれたときにゴミが残ります。

CSS は `prefers-color-scheme` でダークモードに対応させ、`grid-template-columns: repeat(auto-fill, minmax(360px, 1fr))` の1行でレスポンシブにしています。iPhone では1列、デスクトップでは3〜4列。メディアクエリは書いていません。

訪問済みリンクの色（`.title:visited`）は地味に重要でした。毎朝ほぼ同じ顔ぶれのサイトが並ぶので、**既読が視覚的に沈んでいく**と一覧の読み進めがだいぶ楽になります。

---

## 使ってみて分かったこと

**3ソースの性格の違いがはっきり見えます。** Hacker News は英語圏の一次情報とプロダクトローンチ、Reddit は現場の愚痴と議論、はてブは日本語の解説記事と国内の話題。同じ週でも盛り上がる話題がずれるので、**1ページに並べると「世界的に来ている話」と「日本で来ている話」の差が可視化される**のが面白い発見でした。片方にしか出ない話題は、たいていどちらかのコミュニティ固有の関心事です。

**巡回しなくなりました。** 目的だった「30秒で俯瞰する」は達成できています。深く読むのは1日1〜2本で、残りはタイトルだけ見て流します。それでいい、と割り切れたのが RSS リーダーとの一番の違いでした。

### 残っている制約

正直に書いておきます。

- **PC が止まっていた朝はその日の分がスキップされます。** cron なので当然ですが、`anacron` などは入れていません。手動で `./run.sh` を叩けばその時点の内容で作成できます。
- **Reddit のスコアが出ません。**（前述）
- **「流行り」の判定はしていません。** 各サイトのランキングをそのまま並べているだけで、横断的なトレンド抽出や要約はしていません。ここは今後 LLM で「今日のトピック3行まとめ」を足したいところです。

### 次にやるなら

- 3ソース横断で頻出キーワードを抽出し、**その日のテーマ**を上部に出す
- 同じ記事が複数ソースに出たら「HN + はてブ」のように**バッジを重ねて表示**（今は先勝ちで消している）
- 全文取得して要約を添える（ただし依存ゼロの原則とトレードオフになる）

---

## まとめ

自分専用ツールは、**壊れにくさが機能より優先する**と改めて思いました。この記事で書いた判断はほとんど「どう動かすか」ではなく「**どう壊れるか**」の設計です。

- 依存ゼロ → 環境の腐敗で死なない
- 1リクエストに畳む → レート制限で死なない
- 部分失敗を許容 → 1サイトの API 変更で死なない
- 全滅時は書かない → 失敗が既存の成果物を壊さない

外部サイトに依存するツールは必ず壊れます。壊れることを前提に、**壊れたときにどこまで機能を残すか**を先に決めておくと、朝5時に静かに動き続けてくれるようになります。

Python 標準ライブラリだけで週末に組める規模なので、自分の情報源に合わせて作ってみるのをおすすめします。取得元を差し替えるのは `config.py` の数行です。
