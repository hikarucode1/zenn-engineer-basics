---
title: "リレーショナルデータベース入門"
---

## この章のゴール

「アプリ」の正体の半分はデータの出し入れです。そのデータを **長期間、安全に、整合性を保って保存する** ための仕組みが **データベース (DB)**。

この章では最も広く使われる **リレーショナルデータベース (RDB)** の基本概念 — テーブル・カラム・キー・トランザクション・ACID・インデックス — を扱います。

## 13-1. なぜ RDB がここまで普及したのか

1970年代の論文に始まる古い技術なのに、2026年現在も新規プロジェクトのデフォルトです。理由は3つ。

1. **データ整合性が強い** (ACID トランザクション)
2. **柔軟なクエリ** ができる (SQL)
3. **業界の理解度が圧倒的** (どの会社でも使われている)

「迷ったら RDB」が今でも正解の場面が多い、と覚えてください。

## 13-2. テーブル・行・列

RDB のデータは **テーブル (= 表)** に保存されます。
Excel をイメージしてください。**行 (row, record)** が1件、**列 (column, field)** が項目。

例: users テーブル

| id | name | email | age |
|----|------|-------|-----|
| 1 | Hikaru | h@example.com | 25 |
| 2 | Aya | a@example.com | 30 |
| 3 | Ken | k@example.com | 28 |

**スキーマ (schema)** = テーブルの構造 (どんな列があるか、各列の型は何か) を定義したもの。
RDB は **スキーマがガッチリ決まっている** のが特徴で、これを **スキーマフル** と呼びます (NoSQL のスキーマレスと対照的)。

## 13-3. データ型

各カラムには型を指定します。代表的な型 (MySQL/PostgreSQL 共通の感覚):

| 型 | 用途 |
|----|------|
| INTEGER, BIGINT | 整数 |
| DECIMAL(10,2) | 正確な小数 (お金など) |
| FLOAT, DOUBLE | 近似小数 (科学計算など) |
| VARCHAR(n) | 可変長文字列 |
| TEXT | 長文 |
| BOOLEAN | true/false |
| DATE, TIMESTAMP | 日付・日時 |
| JSON, JSONB | JSON データ |
| UUID | グローバルユニークID |

:::message
**お金は DECIMAL、日時は TIMESTAMP (タイムゾーン付き)、ID は BIGINT か UUID** — この3つはお守りとして覚えておくと事故が減ります。
:::

## 13-4. 主キーと外部キー

### 主キー (Primary Key)

行を一意に特定する列。1テーブルに1つ。

```sql
CREATE TABLE users (
  id BIGINT PRIMARY KEY,
  name VARCHAR(100),
  email VARCHAR(255)
);
```

通常は `id` という **自動採番される整数** か **UUID** を使う。

### 外部キー (Foreign Key)

他のテーブルの主キーを参照する列。**テーブル間の関係 (リレーション)** を表す。

```sql
CREATE TABLE posts (
  id BIGINT PRIMARY KEY,
  user_id BIGINT REFERENCES users(id),
  title VARCHAR(200)
);
```

`posts.user_id` が `users.id` を指す → 「この投稿はこのユーザーのもの」と分かる。

「リレーショナル」DB の名の由来は、この **テーブル間の関係をキーで表現する** 仕組みからです。

## 13-5. 1対多 / 多対多

実世界の関係をテーブルに落とすパターン。

### 1対多 (One-to-Many)

1人のユーザーが複数の投稿を持つ。
→ posts に user_id を持たせる (上の例)

### 多対多 (Many-to-Many)

ユーザーが複数のタグを付け、タグも複数のユーザーに付く。
→ **中間テーブル** を作る。

```sql
users (id, name)
tags (id, name)
user_tags (user_id, tag_id)  -- 中間テーブル
```

中間テーブルが第三のテーブルに育っていくのが現場の常で、第16章のデータモデリングでもう一度触れます。

## 13-6. SQL — DB と話す言語

RDB を操作する言語が **SQL (Structured Query Language)**。

```sql
SELECT name, email FROM users WHERE age > 20;
INSERT INTO users (name, email) VALUES ('Hikaru', 'h@example.com');
UPDATE users SET age = 26 WHERE id = 1;
DELETE FROM users WHERE id = 999;
```

宣言的言語 (第8章) の代表例で、**「何が欲しいか」だけ書き、どう探すかは DB が決める**。
詳細は次章で。

## 13-7. ACID — トランザクションの4本柱

RDB の最大の武器が **トランザクション**。

「銀行で A から B に1万円送金する」ような、**複数の処理を1つの塊として全部成功か全部失敗にする** 仕組み。

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 10000 WHERE id = 'A';
  UPDATE accounts SET balance = balance + 10000 WHERE id = 'B';
COMMIT;  -- すべて成功なら確定
-- 途中で失敗したら ROLLBACK; で巻き戻し
```

ACID = 以下4つの頭文字。

| 文字 | 意味 |
|------|------|
| **A**tomicity (原子性) | 全部成功 or 全部失敗 |
| **C**onsistency (一貫性) | 整合性を保つ |
| **I**solation (独立性) | 並行実行が互いに影響しない |
| **D**urability (永続性) | コミットしたら消えない |

これが NoSQL との最大の差別化ポイントで、お金や予約などミスが許されない業務で RDB が選ばれ続ける理由です。

## 13-8. インデックス — DB の目次

「100万件の users テーブルから email が `h@example.com` の人を探す」
何も準備しないと、DB は **全行を上から読みます (O(n))**。

これを高速化するのが **インデックス**。
本の巻末の目次のように、「この列の値はここ」というデータ構造を別に持っておきます。

```sql
CREATE INDEX idx_users_email ON users(email);
```

これだけで検索が **O(log n)** に。第7章の二分探索を思い出してください。インデックスの内部は **B-Tree** という木構造です。

### インデックスの代償

ただしインデックスはタダではありません。

- INSERT/UPDATE/DELETE 時にインデックスも更新が必要 (書き込みが遅くなる)
- ディスク容量を食う

**読み込みが多い列だけにインデックス** が原則。「全部の列にインデックス」は逆効果です。

## 13-9. 主な RDBMS

| 製品 | 特徴 |
|------|------|
| **PostgreSQL** | 機能豊富、業界の主流。新規はまずこれ |
| **MySQL** | シェアが大きい、Web 界の伝統的選択 |
| **SQLite** | 1ファイル DB、組み込み・モバイルに |
| **MariaDB** | MySQL のフォーク |
| **Oracle DB** | 大企業向け商用 |
| **SQL Server** | Microsoft 製、Azure と相性◎ |

迷ったら **PostgreSQL**。理由は機能・コミュニティ・将来性のバランス。

## 13-10. ORM (Object-Relational Mapping)

アプリのオブジェクトと DB のテーブルを **自動で変換** する仕組み。

```python
# SQLAlchemy (Python) の例
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(String)

# SQL を書かずに取れる
user = session.query(User).filter(User.name == "Hikaru").first()
```

代表:
- Python: SQLAlchemy, Django ORM
- JavaScript: Prisma, TypeORM
- Java: Hibernate, JPA
- Ruby: ActiveRecord
- Go: GORM

ORM は便利ですが、**裏で発行される SQL を意識しないと N+1 などの性能問題** を起こしがち。「ORM を使うときも SQL は読める」が現場の必須スキル。

## 13-11. マイグレーション — スキーマ変更の記録

「users テーブルに `phone` 列を足す」みたいな変更を、**履歴付きで管理する仕組み** が マイグレーション。

```
20260101_create_users.sql
20260215_add_phone_to_users.sql
20260301_create_posts.sql
```

各変更を SQL ファイルにし、本番・ステージ・開発で **同じ順序で適用** することで、環境を揃えます。

## まとめ

- RDB はデータを **テーブル (行と列)** で持ち、**スキーマでガッチリ型を決める**
- 行を識別するのが **主キー**、テーブル間の関係を表すのが **外部キー**
- データ操作は **SQL**。「何が欲しいか」だけ書く宣言的言語
- **ACID トランザクション** で「全部成功 or 全部失敗」を保証
- **インデックス** で検索を O(log n) に。ただし書き込みが遅くなる
- **PostgreSQL** が新規のデフォルト、**SQLite** は組み込み用
- **ORM** で書きやすく。ただし発行 SQL を読めるように
- **マイグレーション** でスキーマ変更を履歴管理

次の章では SQL そのものを掘り下げます。
