---
title: "SQL の基本"
---

## この章のゴール

SQL は ORM を使う時代でも **読める・書けることが必須スキル**。
パフォーマンス問題のデバッグ、データ調査、レポート、移行 — どれも結局 SQL を書きます。

この章では `SELECT` から `JOIN`、集約、サブクエリ、ウィンドウ関数、トランザクション制御まで、現場で使う SQL の全景を見ます。

サンプルとして次のテーブルを想定:

```sql
users (id, name, email, age, created_at)
posts (id, user_id, title, body, created_at)
comments (id, post_id, user_id, body, created_at)
```

## 14-1. SELECT — 取得の基本

```sql
SELECT name, email FROM users;
SELECT * FROM users;  -- 全列 (本番では避ける)
```

**`SELECT *` は本番コードで使わない**。テーブルに列が増えた瞬間にバグの温床になるので、必要な列だけ書く癖をつけてください。

### WHERE で絞り込み

```sql
SELECT * FROM users WHERE age > 20;
SELECT * FROM users WHERE name = 'Hikaru' AND age < 30;
SELECT * FROM users WHERE email LIKE '%@gmail.com';
SELECT * FROM users WHERE id IN (1, 2, 3);
SELECT * FROM users WHERE age BETWEEN 20 AND 30;
SELECT * FROM users WHERE email IS NULL;
```

### ORDER BY / LIMIT / OFFSET

```sql
SELECT * FROM users ORDER BY created_at DESC LIMIT 10 OFFSET 20;
```

新着10件、21件目から、という指定。ページネーションに必須。

## 14-2. INSERT / UPDATE / DELETE

```sql
INSERT INTO users (name, email, age)
VALUES ('Hikaru', 'h@example.com', 25);

UPDATE users SET age = 26 WHERE id = 1;

DELETE FROM users WHERE id = 999;
```

:::message alert
**WHERE 抜きの UPDATE と DELETE は全行に効きます**。本番DBで `WHERE` を書き忘れて全レコード消した事故は枚挙にいとまがありません。トランザクションで囲む or `LIMIT` を付けるなどの自衛を。
:::

## 14-3. JOIN — テーブルをつなぐ

複数のテーブルから関連データをまとめて取る。SQL の心臓部。

### INNER JOIN — 両方にあるもの

```sql
SELECT u.name, p.title
FROM users u
INNER JOIN posts p ON p.user_id = u.id;
```

ユーザーと投稿を結合。投稿のないユーザーは出てこない。

### LEFT (OUTER) JOIN — 左側を全部残す

```sql
SELECT u.name, p.title
FROM users u
LEFT JOIN posts p ON p.user_id = u.id;
```

投稿のないユーザーは `title` が NULL で出てくる。「全ユーザー + 投稿件数」のようなレポートで使う。

### RIGHT JOIN / FULL OUTER JOIN

逆方向、両方残す。実務では LEFT JOIN だけで足りることが多い。

### 図でイメージ

```
INNER:      [A∩B]
LEFT OUTER: [A∪(A∩B)]  ← Aの全部 + 一致するB
RIGHT OUTER: [(A∩B)∪B]
FULL OUTER: [A∪B]
```

## 14-4. 集約関数と GROUP BY

データを **まとめて統計を取る** クエリ。

```sql
SELECT COUNT(*) FROM users;
SELECT AVG(age) FROM users;
SELECT MAX(created_at) FROM posts;
```

### GROUP BY — グルーピング

「ユーザーごとの投稿数」を出す:

```sql
SELECT user_id, COUNT(*) AS post_count
FROM posts
GROUP BY user_id;
```

### HAVING — 集約後の絞り込み

```sql
SELECT user_id, COUNT(*) AS post_count
FROM posts
GROUP BY user_id
HAVING COUNT(*) > 10;
```

**WHERE は集約前、HAVING は集約後** の絞り込み。

### 実行順序の暗記

SQL は書く順と実行順が違います:

```
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

これを覚えておくと、エラーメッセージが急に読めるようになります (「`HAVING` で `SELECT` のエイリアスが使えない」など)。

## 14-5. サブクエリ

クエリの中にクエリを入れる。

```sql
-- 平均年齢より上のユーザー
SELECT name, age FROM users
WHERE age > (SELECT AVG(age) FROM users);

-- 投稿があるユーザーだけ
SELECT * FROM users WHERE id IN (SELECT user_id FROM posts);
```

### CTE (Common Table Expression) — WITH句

サブクエリを名前付きで上に書く形式。読みやすい。

```sql
WITH active_users AS (
  SELECT * FROM users WHERE created_at > '2026-01-01'
)
SELECT name FROM active_users WHERE age > 20;
```

複雑なクエリは CTE で段階的に組み立てると読み手に優しい。

## 14-6. ウィンドウ関数 — グループの中で順位や累積

**集約せずに、各行に対してグループ単位の計算結果を付与する** 機能。

```sql
-- ユーザーごとの投稿に1から番号を振る
SELECT user_id, title,
  ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at) AS post_no
FROM posts;

-- 累積金額
SELECT id, amount,
  SUM(amount) OVER (ORDER BY created_at) AS running_total
FROM orders;
```

代表的なウィンドウ関数:
- `ROW_NUMBER()` — 1から連番
- `RANK()` / `DENSE_RANK()` — 順位
- `LAG()` / `LEAD()` — 前/次の行の値
- `SUM() OVER ()` — 累積

「**最初は難しいが分かると一気に強くなる**」機能。データ分析やランキング系の機能で必須。

## 14-7. NULL の罠

SQL の `NULL` は「値がない」という意味で、**通常の値と比較できません**。

```sql
SELECT * FROM users WHERE email = NULL;   -- ❌ 何もヒットしない
SELECT * FROM users WHERE email IS NULL;  -- ✅ 正解
```

```sql
SELECT NULL = NULL;        -- → NULL (FALSE ですらない)
SELECT NULL + 1;           -- → NULL
SELECT COUNT(email) FROM users; -- NULL を除いた件数
SELECT COUNT(*) FROM users;     -- NULL も含む全行
```

新人が SQL で最初にハマるのが NULL の挙動。**「= NULL じゃなくて IS NULL」** を呪文として覚えてください。

## 14-8. トランザクション制御

第13章で出てきた ACID を SQL で扱う。

```sql
BEGIN;  -- または START TRANSACTION
  UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
  UPDATE accounts SET balance = balance + 1000 WHERE id = 2;
COMMIT;  -- 確定
-- 失敗したら ROLLBACK;
```

### 分離レベル (Isolation Level)

並行実行の独立性をどこまで保証するか。

| レベル | 何が防げる |
|-------|----------|
| READ UNCOMMITTED | (ほぼ何も) |
| READ COMMITTED | ダーティリード |
| REPEATABLE READ | 非反復読み取り |
| SERIALIZABLE | すべて (= 直列実行と同じ) |

レベルが高いほど安全で遅い。PostgreSQL/MySQL の既定はそれぞれ READ COMMITTED / REPEATABLE READ です。

## 14-9. EXPLAIN — クエリの実行計画を読む

「このクエリが遅い」と思ったら最初にやること。

```sql
EXPLAIN SELECT * FROM users WHERE email = 'h@example.com';
```

出力で見るべきポイント:
- **Seq Scan** (全行スキャン) → インデックスが効いていない
- **Index Scan** → インデックスが効いている
- **Nested Loop** vs **Hash Join** → 結合方法
- **rows** → 推定行数

第13章のインデックスがちゃんと使われているかを、ここで確認します。

## 14-10. N+1 問題 — 現場で最頻出の性能バグ

```python
users = User.query.all()           # 1回のクエリ
for u in users:
    print(u.posts)                  # ユーザーごとに1回 → N回
# 合計 N+1 回のクエリ
```

ORM 利用時に必ずぶつかる罠。`JOIN` 1発で済むはずが、データ件数分のクエリが飛んで爆遅になる。

対策:
- **eager loading** (Prisma の `include`, SQLAlchemy の `joinedload`)
- 適切な JOIN を SQL で書く

## 14-11. SQL インジェクション対策

ユーザー入力を **そのまま SQL に文字列結合** すると、攻撃者が DB を読み放題・壊し放題になる脆弱性。

```python
# ❌ 危険 (絶対やらない)
query = "SELECT * FROM users WHERE name = '" + user_input + "'"

# ✅ プレースホルダ (パラメータ化クエリ)
query = "SELECT * FROM users WHERE name = ?"
cursor.execute(query, (user_input,))
```

ほとんどの言語の DB ライブラリと ORM がこれを標準でサポートしています。**「SQL を文字列結合しない」**、これだけで OWASP Top 10 の常連が消えます。

## まとめ

- `SELECT` の WHERE / ORDER BY / LIMIT を体に入れる
- `JOIN` の **INNER / LEFT** を使い分ける
- 集計は **GROUP BY + HAVING**。WHERE と HAVING の違いに注意
- 実行順序は **FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT**
- 複雑なクエリは **CTE (WITH句)** で段階的に
- **ウィンドウ関数** はランキング・累積計算で強力
- **NULL は `= NULL` でなく `IS NULL`** で扱う
- **トランザクション** で原子性を確保、分離レベルでバランスを取る
- 遅いクエリは **EXPLAIN** で計画を読む
- **N+1 問題** は eager loading で対策
- **SQLインジェクション** は必ずプレースホルダで防ぐ

次の章では、RDB の対抗 — **NoSQL とキャッシュ** に進みます。
