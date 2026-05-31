---
title: "CI/CD の基本"
---

## この章のゴール

「push したら自動でテストが走る」「main にマージしたら本番にデプロイ」 — これを当然の感覚にするのが CI/CD です。

新人時代はピンと来づらいですが、**チーム開発のスピードと品質の根幹**。
この章では CI と CD の違い、GitHub Actions の例、デプロイ戦略、フィーチャーフラグを扱います。

## 21-1. CI と CD と CD

3文字略語が混ざるので整理。

| 略 | 正式名 | 意味 |
|----|-------|------|
| **CI** | Continuous Integration | ビルド・テストを継続的に統合 |
| **CD** | Continuous Delivery | いつでもデプロイ可能な状態に保つ |
| **CD** | Continuous Deployment | 自動でデプロイまで実行 |

Delivery と Deployment はよく混同される。**Deploy が自動か手動かの違い**。

## 21-2. CI で何をするか

push やプルリク作成のたびに自動で:

1. **依存をインストール**
2. **ビルド**
3. **lint / フォーマッタ**
4. **テスト実行** (単体・統合)
5. **セキュリティスキャン**
6. **カバレッジ計測**

これがグリーン (✓) でないと PR がマージできない、というガードを置く。

## 21-3. GitHub Actions の例

最も普及している CI/CD ツール。`.github/workflows/ci.yml` に書く。

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run lint
        run: ruff check .

      - name: Run tests
        run: pytest --cov=app
```

ポイント:
- **on**: いつ走らせるか (push, PR, スケジュール)
- **jobs**: 並列に動く仕事の塊
- **steps**: ジョブ内の手順

他に CircleCI, GitLab CI, Jenkins, AWS CodeBuild など色々あるが、基本構造はだいたい同じ。

## 21-4. CD — 自動デプロイ

CI が通った後、本番 / ステージング環境へ自動で反映する。

```yaml
deploy:
  needs: test
  if: github.ref == 'refs/heads/main'
  runs-on: ubuntu-latest
  steps:
    - name: Deploy to production
      run: ./scripts/deploy.sh
```

deploy の中身はサービスによる:
- AWS S3 + CloudFront に上げる
- Vercel / Netlify に push
- Kubernetes に kubectl apply
- Docker イメージを push してサーバー更新

## 21-5. 環境の分離

通常、最低3つの環境を持つ。

| 環境 | 用途 |
|------|------|
| **開発 (dev)** | 各自のローカル |
| **ステージング (staging)** | 本番と同じ構成、テスト用 |
| **本番 (production)** | 実ユーザー向け |

各環境ごとに DB・APIキー・URLが違う。設定は環境変数や AWS Parameter Store などで切り替える。

## 21-6. デプロイ戦略

「本番にどう新しいコードを反映するか」のパターン。

### ローリングデプロイ

サーバーを **1台ずつ順に更新**。常時誰かが新版・誰かが旧版を見る期間がある。最も普通。

### ブルー・グリーンデプロイ

完全に独立した2つの環境 (Blue=現行, Green=新版) を用意し、**ロードバランサで切り替え**。
切り戻しが速い (ロードバランサを戻すだけ)。コスト2倍。

### カナリアデプロイ

新版をまず **少数のサーバー (5%とか)** にだけ出して、問題なければ徐々に増やす。
大規模サービスの定番。AB テストとも組み合わせやすい。

### Recreate

全停止してから新版を立ち上げる。停止時間あり。レアケースのみ。

## 21-7. ロールバック

新版デプロイで本番が壊れたら **即座に旧版に戻せる** こと。

- ブルーグリーン → ロードバランサを戻す (秒)
- コンテナベース → 旧イメージに切り替え (分)
- データベースのマイグレーションありなら工夫が要る (後方互換性)

「**ロールバックできないデプロイは怖くて誰もしない**」 — 健全なチームの合言葉。

## 21-8. データベースマイグレーションの作法

CI/CD で一番ハマるのが DB。

### 後方互換のあるマイグレーション

旧コードでも新コードでも動くようにする。

```
1. 新カラムを NULL 許容で追加 (旧コードは無視できる)
2. 新コードをデプロイ (新カラムを書く)
3. データを埋めるバッチを流す
4. NOT NULL 制約をつける
5. 旧カラム/ロジックを削除
```

「**いきなり破壊的変更を打たない**」が鉄則。

## 21-9. Infrastructure as Code (IaC)

本番のインフラ (サーバー、DB、ロードバランサなど) を **コードで管理する** こと。

ツール:
- **Terraform** (HashiCorp) — クラウド横断、デファクト
- **AWS CloudFormation** — AWS 専用
- **Pulumi** — TypeScript/Python など普通の言語で書ける
- **Ansible** — サーバー設定の自動化

利点:
- 構成が Git で履歴管理される
- 「クリックで作って二度と再現できない」を防げる
- レビューできる

## 21-10. Linter / Formatter / 型チェック

CI で必ず入れたい3点セット。

| 用途 | Python | JavaScript |
|------|--------|-----------|
| Linter | ruff, flake8 | ESLint |
| Formatter | black, ruff format | Prettier |
| 型チェッカ | mypy, pyright | tsc (TypeScript) |

「コミット時に自動整形」+「PRで自動チェック」を組み合わせると、**コードレビューが「中身」の議論だけになる** ので生産性が一気に上がる。

## 21-11. フィーチャーフラグ

「**コードはデプロイするけど機能はOFF**」にしておき、後から有効化できる仕組み。

```python
if feature_flag_enabled("new_search", user):
    return new_search()
else:
    return old_search()
```

ツール:
- LaunchDarkly
- Unleash
- GrowthBook (OSS)
- 自前テーブル

利点:
- 大きな機能をマージしても安全
- 一部ユーザーだけに先行公開できる (A/Bテスト)
- 問題があれば一瞬で OFF にできる

「**デプロイとリリースを分離する**」現代的なアプローチ。

## 21-12. シークレットを CI/CD でどう扱うか

CI のスクリプトに API キーを書くと **ログに出る、リポジトリに残る、絶対漏れる**。

正しいやり方:
- **GitHub Secrets** や CI が提供する暗号化変数
- AWS Secrets Manager / GCP Secret Manager から実行時に取得
- **OIDC連携** で長期的な静的キーをそもそも持たない (近年の主流)

OIDC: CI 実行ごとに **一時的なクレデンシャルを発行** する仕組み。AWS / GCP / GitHub Actions が標準対応。

## 21-13. パイプラインを健全に保つ

「**CI が遅い、壊れている、緑が信用できない**」と、チーム全体が CI を無視し始めて崩壊します。

- テストを並列実行
- キャッシュ (依存・ビルド成果物) を効かせる
- フレーキー (たまに落ちる) テストは即修正 or 隔離
- 失敗は無視せず、その日中に原因究明
- 緑じゃないとマージできない設定 (Branch Protection)

## まとめ

- **CI** = 統合の自動化、**CD** = デリバリ/デプロイの自動化
- push のたびに **ビルド・lint・テスト・スキャン** が走る
- **GitHub Actions** が現代の主流。YAML で書く
- 環境は **dev / staging / production** を分ける
- デプロイ戦略は **ローリング / ブルーグリーン / カナリア**
- **ロールバック可能** にしておくのが鉄則
- DB マイグレーションは **後方互換** を意識
- インフラは **コード化 (IaC: Terraform)**
- Linter / Formatter / 型チェッカで PR を「中身」だけの議論に
- **フィーチャーフラグ** でデプロイとリリースを分離
- シークレットは **GitHub Secrets / OIDC** で安全に
- CI を遅い・壊れた状態で放置しない

次の章では、現代のデプロイ先 — **クラウドとコンテナ** に進みます。
