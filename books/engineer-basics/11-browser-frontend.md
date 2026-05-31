---
title: "ブラウザとフロントエンドの基本"
---

## この章のゴール

URL を打ってから画面が表示されるまでに、**ブラウザの中で何が起きているか**。
そして、そのブラウザの上で動く「**フロントエンド**」は何をする領域なのか。

この章では HTML/CSS/JavaScript の3点セット、DOM、レンダリング、SPA、モダンフレームワーク — Web フロントエンドの全体像を眺めます。

## 11-1. ブラウザのざっくり構造

ブラウザは小さな OS のような存在で、内部に複数のエンジンを抱えています。

| エンジン | 役割 |
|---------|------|
| ネットワーク | HTTP リクエストを送受信 |
| HTMLパーサ | HTMLを解析してDOM木を作る |
| CSSパーサ | CSSを解析してスタイル情報を作る |
| レンダリングエンジン | DOM + CSS を元にレイアウト・描画 |
| JavaScriptエンジン | JS を実行 (Chrome の V8 など) |
| ストレージ | Cookie, LocalStorage, IndexedDB |

代表的なブラウザとエンジン:

| ブラウザ | レンダリングエンジン | JSエンジン |
|---------|------------------|----------|
| Chrome, Edge | Blink | V8 |
| Safari | WebKit | JavaScriptCore |
| Firefox | Gecko | SpiderMonkey |

## 11-2. Web の3点セット — HTML / CSS / JavaScript

| 言語 | 役割 | 例えるなら |
|------|------|----------|
| **HTML** | 構造 (何があるか) | 家の骨組み |
| **CSS** | 見た目 (どう見えるか) | 壁紙・塗装 |
| **JavaScript** | 動き (何が起きるか) | 電気・水道 |

### HTML — 構造

```html
<!DOCTYPE html>
<html>
  <head><title>My Site</title></head>
  <body>
    <h1>こんにちは</h1>
    <p>世界</p>
  </body>
</html>
```

タグの入れ子で **木構造** を表す。これが DOM の元になる。

### CSS — 見た目

```css
h1 {
  color: blue;
  font-size: 32px;
}
```

「セレクタ → プロパティ:値」のリスト。
レイアウト系のキーは **Flexbox** と **Grid** (古い `float` レイアウトはもう書かない)。

### JavaScript — 動き

```javascript
document.querySelector("button").addEventListener("click", () => {
  alert("clicked!");
});
```

ボタンが押されたら何かする、API を叩く、DOM を書き換える — ユーザー操作と非同期処理を担う。

## 11-3. DOM — ブラウザ内の HTML 木

ブラウザは HTML を読むと、**DOM (Document Object Model)** という木構造のオブジェクトに変換します。

```
document
  └ html
     ├ head
     │  └ title
     └ body
        ├ h1
        └ p
```

JavaScript はこの DOM を操作してページを書き換えます。

```javascript
document.querySelector("h1").textContent = "Hello";
```

`querySelector` で要素を取り、`textContent` を書き換える。**DOM 操作 = JavaScript で UI を変える** の本質。

## 11-4. URL を打ってから画面が出るまで

新人面接でよく聞かれる「Google を開いた瞬間に何が起きるか」を簡略化すると:

1. **DNS で名前解決** (第9章)
2. **TCP/TLS で接続確立** (3-way handshake + TLS handshake)
3. **HTTP リクエスト送信** (第10章)
4. **HTML レスポンス受信**
5. **HTMLをパース → DOM 構築**
6. **CSS をパース → スタイル情報構築**
7. **DOM + CSS → レンダー木 → レイアウト計算 → 描画**
8. **追加リソース取得** (画像、JS、フォント...)
9. **JavaScript 実行 → DOM 書き換え → 再描画**

ここまで現代の Web ではミリ秒〜秒のオーダーで終わります。

## 11-5. レンダリングと再描画

レンダリングは以下のフェーズで段階的に進みます。

1. **Style**: 各要素にどのスタイルが適用されるか計算
2. **Layout**: 大きさと位置を計算
3. **Paint**: ピクセルを描く
4. **Composite**: 複数レイヤを重ねる

JavaScript で DOM を書き換えると、必要に応じてこれらが再実行されます。

| 操作 | 起きること |
|------|----------|
| 色を変える | Paint だけ (軽い) |
| 大きさを変える | Layout から (重い) |
| 要素を追加 | Layout から (重い) |
| transform/opacity を変える | Composite だけ (最も軽い) |

「**Layout を起こさないアニメーション**」が滑らかな UI のコツ。CSS の `transform` と `opacity` だけ動かす、というのが定番です。

## 11-6. クライアントサイドストレージ

ブラウザに状態を保存する手段。

| 仕組み | 容量 | 寿命 | 用途 |
|-------|------|------|------|
| Cookie | 4KB | 設定次第 | セッション、サーバーへ自動送信 |
| LocalStorage | 5〜10MB | 永続 | ユーザー設定 |
| SessionStorage | 5〜10MB | タブを閉じるまで | 一時データ |
| IndexedDB | 大量 (数百MB〜) | 永続 | オフラインアプリ、構造化データ |

**機密情報は LocalStorage に入れない** (JS で読めるので XSS に弱い)。
セッションIDは Cookie + HttpOnly が定石、と覚えておきましょう。

## 11-7. SPA と MPA

### MPA (Multi-Page Application)

リンクをクリックするたびに **サーバーが新しい HTML を返す** 古典的な形式。
Rails、PHP、Django、WordPress の典型例。

### SPA (Single-Page Application)

**最初の HTML を1回だけ取得**、以後は JavaScript が API を叩いて画面を書き換える形式。
Gmail、Twitter、Notion のような「アプリっぽい Web」がこれ。

| | MPA | SPA |
|--|----|----|
| 初回表示 | 速い | 遅い |
| ページ遷移 | サーバー往復 (遅い) | JS で差分描画 (速い) |
| SEO | 強い | 工夫が必要 |
| 開発の複雑さ | 低い | 高い |

### SSR と CSR と SSG

SPA の進化形として、最初の HTML をどう作るかでさらに分類されます。

- **CSR (Client-Side Rendering)**: ブラウザで全部描画
- **SSR (Server-Side Rendering)**: サーバーで HTML を生成して返す
- **SSG (Static Site Generation)**: ビルド時に HTML を作っておく
- **ISR (Incremental Static Regeneration)**: SSG + 定期再生成

Next.js (React)、Nuxt (Vue)、SvelteKit などのフレームワークがこれらをハイブリッドで提供しています。

## 11-8. モダンフロントエンドフレームワーク

純粋な JavaScript で DOM を直接いじるとコードがすぐ複雑化します。それを救うのがフレームワーク。

### React (Meta)

UI を **コンポーネント** に分けて宣言的に書くライブラリ。

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

エコシステムが圧倒的。**現状の業界デファクト**。Next.js とセット。

### Vue.js

学習コストの低さが売り。テンプレート構文が HTML 寄り。

### Svelte / SolidJS

「コンパイルして仮想DOMを使わない」新世代。バンドルが小さく速い。

どれもベースの考え方は **「状態 → UI」を関数的に表現する** こと。1つ深く学べば他は1〜2週間で書けるようになります。

## 11-9. TypeScript

JavaScript に **静的型付け** を足した言語。コンパイル時に JavaScript に変換される。

```typescript
function greet(name: string): string {
  return `Hi, ${name}`;
}
```

大規模化したフロントエンド開発で必須に近い存在。**新規プロジェクトはほぼ TypeScript 一択** が2026年時点の現実です。

## 11-10. パッケージマネージャとビルドツール

- **パッケージマネージャ**: npm, yarn, pnpm
- **ビルドツール**: Vite, Webpack, esbuild, Turbopack

ビルドツールは TypeScript の変換、複数の .ts/.css ファイルの結合、画像の最適化などを担当。Vite が今の主流。

## 11-11. アクセシビリティ (a11y)

スクリーンリーダー利用者、キーボードのみのユーザー、色覚多様性のユーザーも使えるようにする設計。

最低限:
- `<button>` を使うべきところで `<div onclick>` を使わない
- 画像には `alt` を付ける
- フォーム要素には `<label>` をつなぐ
- コントラスト比に注意

これは品質と倫理の両方の話。第17章のセキュリティと並ぶ、現代の必須教養です。

## まとめ

- ブラウザは **ネットワーク + パーサ + レンダリング + JSエンジン** を抱えた小さなOS
- **HTML (構造) / CSS (見た目) / JavaScript (動き)** が3点セット
- ブラウザは HTML を **DOM** という木にしてメモリに保持。JS が DOM を書き換えるとUIが変わる
- レンダリングは **Style → Layout → Paint → Composite**。Layout を避けるとアニメが滑らか
- ストレージは **Cookie / LocalStorage / SessionStorage / IndexedDB**
- **SPA / MPA**、その派生で **SSR / CSR / SSG / ISR** がある
- React がデファクト、TypeScript が新規プロジェクトの標準
- **アクセシビリティ** は最低限の教養

次の章では、フロントエンドとバックエンドをつなぐ — **API 設計** に進みます。
