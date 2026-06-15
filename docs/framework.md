# vte.cx フレームワーク仕様

vte.cx BaaS のデータモデル・アクセス制御・スキーマ設計に関するリファレンスです。
このドキュメントは vte.cx を使用するすべてのプロジェクトで横展開できます。

---

## データ構造

### URI 設計

`getFeed` は指定したパス以下の**エントリ一覧を返す**ため、1つのディレクトリに複数種類のデータを混在させてはならない。

#### NG パターン（データ混在）

```
/crm/customer/0000000001
/crm/customer/contact/...   ← 担当者が顧客と同じディレクトリ
/crm/customer/deal/...      ← 商談が顧客と同じディレクトリ
```

`getFeed('/crm/customer')` を呼ぶと顧客一覧だけでなく担当者・商談も返ってくる。

#### OK パターン（種類ごとに独立したディレクトリ）

```
/crm/customer/0000000001
/crm/customer/0000000001/contact   ← 顧客Aの担当者専用
/crm/customer/0000000001/deal      ← 顧客Aの商談専用
/crm/customer/0000000001/activity  ← 顧客Aの対応履歴専用
```

| クエリ                                  | 返るデータ           |
| --------------------------------------- | -------------------- |
| `getFeed('/crm/customer')`              | 顧客一覧             |
| `getEntry('/crm/customer/{id}')`        | 顧客データ（1件）    |
| `getFeed('/crm/customer/{id}/contact')` | その顧客の担当者のみ |
| `getFeed('/crm/customer/{id}/deal')`    | その顧客の商談のみ   |

#### 設計ルール

1. **1ディレクトリ = 1エンティティ種別** — 異なるエンティティを同じディレクトリ配下に置かない
2. **親子関係はパス階層で表現** — 子エンティティのパスに親IDを含める（例: `/crm/contact/{customerId}/{cid}`）
3. **folderacls はエンティティ種別ごとに登録** — データを置くルートパスをすべて登録する

---

### 戻り値の型

SDK メソッドごとに戻り値の型が異なる。`feed.entry` は SDK 内部で解決されるため、呼び出し側は意識しなくてよい。

| メソッド   | 正常時の型         | 204（データなし）    |
| ---------- | ------------------ | -------------------- |
| `getFeed`  | `VtecxApp.Entry[]` | `null` / `undefined` |
| `getEntry` | `VtecxApp.Entry`   | `null` / `undefined` |

エラー時は `{ feed: { title: string } }` 型が返る。

```typescript
const entries = await vtecxnext.getFeed('/crm/customer')
// entries は VtecxApp.Entry[]（配列）。Array.isArray による正規化は不要

const entry = await vtecxnext.getEntry('/crm/customer/0000000001')
// entry は VtecxApp.Entry（単一オブジェクト）または null（204）
```

---

### entry.id の形式

`entry.id` は `{パス},{リビジョン}` の形式で返る。パスを取り出す場合は `,` より前の部分のみ使う。

```typescript
// NG: リビジョン番号が混入する
const id = entry.id.replace('/crm/customer/', '')  // → "0000000001,3"

// OK: パス部分のみ取り出す
const id = entry.id.split(',')[0].replace('/crm/customer/', '')  // → "0000000001"
```

---

## データ取得

### 使い分け

| 状況                                    | 使うメソッド                    |
| --------------------------------------- | ------------------------------- |
| 件数が少なく全件取得で問題ない一覧      | `getFeed(uri)`                  |
| 件数が多くページング必要な一覧          | `getPageWithPagination(uri, n)` |
| 一覧の中から1件を取得（詳細・編集画面） | `getEntry(uri)`                 |

**基本ルール:**

- 一覧データの取得はページングを前提に行う（`getPageWithPagination`）
- 一覧からデータを1件選んで取得する場合は、必ずデータキー（URI）を使って `getEntry` で取得する
- ページング不要な少量データ（マスタ・サブリストなど）は `getFeed` でよい

```typescript
// ページング一覧（顧客一覧など）
const entries = await vtecxnext.getPageWithPagination('/crm/customer?l=25', n)

// ページング不要一覧（件数が少ないサブリストなど）
const feed = await vtecxnext.getFeed('/crm/contact/0000000001')

// 1件取得（一覧→詳細遷移時、編集画面など）
const entry = await vtecxnext.getEntry('/crm/customer/0000000001')
```

---

### メソッド一覧

| メソッド                        | 用途                                                      |
| ------------------------------- | --------------------------------------------------------- |
| `getFeed(uri)`                  | ページング不要な一覧取得                                  |
| `getEntry(uri)`                 | データキーを指定して1件取得                               |
| `getPageWithPagination(uri, n)` | ページング一覧取得（`pagination` + `getPage` を一括処理） |
| `pagination(uri, pagerange)`    | カーソルリスト（pageindex）を作成                         |
| `getPage(uri, n)`               | 指定ページのデータを取得（カーソルリストが必要）          |

---

### ページング

vte.cx のページングは **カーソルリスト（pageindex）** を事前に作成してから、ページ単位でデータを取得する2ステップ方式を取る。

```
① pagination()  ─ 取得キー（URI）とページ範囲を指定してカーソルリストを作成
② getPage()     ─ 作成済みカーソルを使って指定ページのデータを取得
```

`getPageWithPagination(uri, n)` はこの2ステップを内部で自動処理する。n=1 のとき自動的に `pagination()` を呼び出す。データが存在しない場合は `undefined` を返す。

#### URI パラメータ

| パラメータ              | 指定箇所                  | 説明                          |
| ----------------------- | ------------------------- | ----------------------------- |
| `l=件数`                | URI クエリ                | 1ページ当たりの表示件数       |
| `n=ページ番号`          | `getPage()` が自動付加    | 取得するページ番号（1始まり） |
| `_pagination=start,end` | `pagination()` が自動付加 | カーソルを作成するページ範囲  |

> `pagination()` と `getPage()` に渡す URI は完全一致である必要がある（検索条件も含めて同一にすること）。

#### pagerange

`pagination(uri, pagerange)` の `pagerange` は `"開始ページ,終了ページ"` 形式。**endPage は 50 推奨**。

```
"1,50"   ← ページ1〜50 のカーソルを一括作成（推奨）
"51,100" ← ページ51〜100 を追加作成（続きがある場合）
```

#### 実装例

```typescript
const n = parseInt(vtecxnext.getParameter('n') ?? '1', 10)
const uri = `/crm/customer?l=25`

const entries = await vtecxnext.getPageWithPagination(uri, n)
return vtecxnext.response(200, entries ?? null)
```

#### PaginationInfo

`pagination()` の戻り値。

```typescript
type PaginationInfo = {
  lastPageNumber:   number   // 作成済みカーソルの最終ページ番号（0 = データなし）
  countWithinRange: number   // 指定ページ範囲内のエントリ数
  hasNext:          boolean  // true = 指定 endPage を超えるデータが存在する
  isMemorysort:     boolean  // メモリソートモードかどうか
}
```

| 値                     | 意味                         | 対処                                            |
| ---------------------- | ---------------------------- | ----------------------------------------------- |
| `lastPageNumber === 0` | データが1件もない            | null または空を返す                             |
| `hasNext === true`     | endPage 以降にもデータがある | 必要なら `pagination(uri, '51,100')` を追加実行 |

---

## アクセス制御

### 概念

vte.cx のアクセス制御はすべて **フレームワークが担う**。API ルートはアクセス制御を実装しない。

```
① クライアント（APIルート）がデータにアクセス権限を付与する
② フレームワークがそのアクセス権限を参照してアクセス制御を行う
③ アクセス権限がない場合はフレームワークが HTTP 403 を返す
```

**API ルートではロールチェック・オーナーチェックを実装しない。** folderacls とエントリの contributor 設定がアクセス制御のすべてである。

---

### folderacls.json

アプリが使用するデータパスは、事前に vte.cx サーバーに登録する必要がある。

```json
[
  {
    "contributor": [
      { "uri": "urn:vte.cx:acl:/_group/$admin,CURD" },
      { "uri": "urn:vte.cx:acl:+,CURDE" }
    ],
    "link": [{ "___rel": "self", "___href": "/your/path" }]
  }
]
```

#### 主体と権限

| 主体             | 表記             | 用途                                      |
| ---------------- | ---------------- | ----------------------------------------- |
| システム管理者   | `/_group/$admin` | サービス作成者。必ず `CURD` を付与する    |
| ログインユーザー | `+`              | 認証済みユーザー全員。通常 `CURDE` を付与 |
| 全員（匿名含む） | `*`              | 公開データ（`R` のみ推奨）                |
| カスタムグループ | `/_group/{name}` | 独自ロール（例: `/_group/sales`）         |

| 権限 | 意味                                 |
| ---- | ------------------------------------ |
| `C`  | 作成（POST）                         |
| `R`  | 読取（GET）                          |
| `U`  | 更新（PUT）                          |
| `D`  | 削除（DELETE）                       |
| `E`  | API 経由のみ許可（直接アクセス禁止） |

- `/_group/$admin` には必ず `CURD` を付与する（`E` を付けると直接アクセスが制限される）
- ログインユーザー（`+`）には `CURDE` を付与し、API 経由のみに制限する
- 子パスを登録する場合、親パスも必ず登録する

#### システムディレクトリ

先頭が `_` のパス（`/_group`、`/_html` など）はシステムディレクトリで、サービス作成時にフレームワークが自動生成する。folderacls.json への定義は不要。

---

### contributor によるアクセス権限付与

エントリへのアクセス権限は `contributor` フィールドで設定する。フォーマットは folderacls.json と同一。

```
urn:vte.cx:acl:{グループキーまたはUID},{権限}
```

```typescript
const uid = await vtecxnext.uid()
const entry = {
  link: [{ ___rel: 'self', ___href: '/crm/customer/0000000001' }],
  customer: { ... },
  contributor: [
    { uri: 'urn:vte.cx:acl:/_group/$admin,CURD' },
    { uri: `urn:vte.cx:acl:${uid},CURD` },
    { uri: 'urn:vte.cx:acl:/_group/viewer,R' },
  ],
}
```

エントリを更新する際は `existing.contributor` を引き継ぐことで、元のアクセス権限が失われない。

```typescript
const existing = await vtecxnext.getEntry(uri)
const entry = {
  link: [{ ___rel: 'self', ___href: uri }],
  id: existing?.id,
  myData: { ... },
  contributor: existing?.contributor,
}
await vtecxnext.put({ feed: { entry: [entry] } })
```

---

### 403 レスポンスへの対応

フレームワークが 403 を返した場合、クライアントはセッション切れまたは権限不足として処理する。

```typescript
if (error?.status === 403) {
  router.push('/login')
}
```

---

## スキーマ（template.xml）

### 構文

エンティティのフィールドは `<content>` 内に DSL で定義する。

```
エンティティ名
 フィールド名(型){最大値}
 フィールド名(型)!
```

- 1スペースインデントで子フィールド（ネスト）
- `!` で必須フィールド、`{n}` で string の最大文字数・数値の最大値

#### 型

| 型                 | 用途                   |
| ------------------ | ---------------------- |
| `string`           | 文字列                 |
| `int`              | 整数                   |
| `long`             | 大きな整数（金額など） |
| `date`             | 日付（ISO 8601）       |
| `boolean`          | 真偽値                 |
| `float` / `double` | 小数                   |

### スキーマ変更の制約

- **フィールドの削除は不可** — 一度定義したフィールドは恒久的に残る
- **追加フィールドは必ず末尾に追記する** — 途中に挿入するとスキーマが壊れる
- **`entry.title`, `entry.subtitle` は使用しない** — 全フィールドを専用スキーマで定義する
- **フィールド名は `snake_case`** — 小文字英字・数字・アンダースコアのみ（先頭は小文字英字）

```
// ❌ NG: 既存フィールドの途中に挿入
activity
 next_action(string){500}
 next_action_date(date)     ← 途中に挿入
 contact_uri(string){500}   ← 既存フィールド

// ✅ OK: 既存フィールドをすべて保持し、末尾に追記
activity
 next_action(string){500}
 contact_uri(string){500}   ← 既存フィールドはそのまま
 next_action_date(date)     ← 末尾に追加
```

### template.xml ファイル全体の構造

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<feed>
<entry>
<content>customer
 name(string){255}
 address(string){500}
 phone(string){20}
 status(string){20}
deal
 name(string){255}
 amount(long)
 probability(int){100}
 scheduled_date(date)
</content>
<link href="/_settings/template" rel="self"></link>
<rights>
customer.status:/crm/customer
customer.assigned_uid:/crm/customer
deal.stage:/crm/deal
name;/crm/customer|/crm/deal
password#
</rights>
</entry>
<entry>
  <link href="/_settings/template_property" rel="self"/>
</entry>
</feed>
```

---

### インデックス設定（`<rights>` タグ）

フィールドに検索インデックスを設定することで、そのフィールドを使った絞り込み・ソートが可能になる。

#### 構文

```
{フィールドパス}:{フォルダパスの正規表現}
```

| 構文                  | 種類                   | 用途                             |
| --------------------- | ---------------------- | -------------------------------- |
| `field:path`          | 通常インデックス       | 完全一致検索・ソート・範囲検索   |
| `field;path`          | 全文検索インデックス   | 部分一致検索（`like` 検索）      |
| `field1\|field2;path` | 複数フィールド全文検索 | 複数フィールドをまとめて全文検索 |
| `field:path=role`     | フィールド ACL         | 指定ロールのみ読み書き可         |
| `field#`              | 暗号化                 | フィールド値をサーバー側で暗号化 |

#### 注意事項

- **同一フィールドへの複数パス設定は `|` で1行にまとめる**（別々の行に分けると "Already specified" エラー）
- インデックス追加後は必ず `pnpm upload:template` でサーバーに反映する
- **⚠️ インデックス設定前に登録済みのデータにはインデックスが適用されない** — 既存データを再 PUT する必要がある
- フォルダパスは正規表現のため `/crm/customer` は `/crm/customer` で始まるすべてのパスにマッチする

---

### インデックスを使った検索クエリ

```
/path?f&entity.field-eq-value&l=25
```

`f` パラメータ必須。インデックス検索が有効なのは **先頭の検索条件パラメータのみ**（2番目以降はメモリフィルタ）。

| 演算子 | 意味                      |
| ------ | ------------------------- |
| `-eq-` | 等しい（完全一致）        |
| `-ne-` | 等しくない                |
| `-lt-` | より小さい                |
| `-le-` | 以下                      |
| `-gt-` | より大きい                |
| `-ge-` | 以上                      |
| `-fm-` | 前方一致（LIKE 'value%'） |
| `-bm-` | 後方一致（LIKE '%value'） |
| `-rg-` | 正規表現                  |
| `-ft-` | 全文検索                  |

OR 条件: `?f&|field-eq-a&|field-eq-b`

---

## ユーザーと UID

### UID の発行

ユーザーがサービスに登録（サインアップ）すると、vte.cx フレームワークが自動的に **UID** を発行する。同時にユーザー専用ディレクトリ `/_user/{uid}` が生成される（folderacls.json への定義は不要）。

### UID の取得

```typescript
const vtecxnext = new VtecxNext(req)
const uid = await vtecxnext.uid()
```

### ログイン中のユーザー情報の取得

ログイン中ユーザーの詳細情報は `/_user/{uid}` エントリから取得する。

```typescript
const uid = await vtecxnext.uid()
const userEntry = await vtecxnext.getEntry(`/_user/${uid}`)
const email: string = userEntry?.contributor?.[0]?.email ?? ''
```

#### ユーザーエントリの構造（`/_user/{uid}`）

| フィールド | 内容 |
| --- | --- |
| `title` | UID |
| `subtitle` | ニックネーム |
| `summary` | ステータス（`Activated` / `Interim`） |
| `contributor[0].email` | メールアドレス |

### アカウントの状態

| 状態 | summary 値 | ログイン |
| --- | --- | --- |
| 仮登録 | `Interim` | 不可（確認メール未クリック） |
| 本登録 | `Activated` | 可 |

### サービス名の取得

```typescript
const serviceName = await vtecxnext.service()
```

### ユーザー専用データの保存

```typescript
const uid = await vtecxnext.uid()
const entry = {
  link: [{ ___rel: 'self', ___href: `/crm/user/${uid}` }],
  user: { displayName: 'John Doe', role: 'sales' },
}
await vtecxnext.put({ feed: { entry: [entry] } })
```

---

## パスワード変更

vtecxnext のパスワード変更には **2 つのフロー** がある。いずれも内部で `vtecxnext.changepass()` を呼び出し、vte.cx の `PUT /_changephash` エンドポイントに送信する。

### SDK メソッド

```typescript
vtecxnext.changepass(newpswd: string, oldpswd?: string, passresetToken?: string)
```

| 引数 | 説明 |
| --- | --- |
| `newpswd` | 新しいパスワード（ハッシュ済み） |
| `oldpswd` | 現在のパスワード（ハッシュ済み）。ログイン済みフローで使用 |
| `passresetToken` | パスワードリセットトークン。メールリセットフローで使用 |

パスワードは送信前に `getHashpass(password)`（`@vtecx/vtecxauth`）でハッシュする。

### フロー 1：メールリセット（未ログイン）

```
/forgot-password でメールアドレスを入力
  → vtecxnext.passreset() でリセットメール送信
  → メール内リンク（_passreset_token & _RXID 付き URL）をクリック
  → /change-password ページで新パスワードを入力
  → POST /api/changepass?_RXID={rxid} に { newpswd, passresetToken } を送信
  → vtecxnext.loginWithRxid(rxid) でセッション確立後
  → vtecxnext.changepass(newpswd, undefined, passresetToken) を実行
```

### フロー 2：現在のパスワードで変更（ログイン済み）

```
/account/change-password を開く
  → 現在のパスワード・新しいパスワードを入力
  → POST /api/changepass に { newpswd, oldpswd } を送信（_RXID なし）
  → セッションクッキーで認証済みのまま
  → vtecxnext.changepass(newpswd, oldpswd) を実行
```

### API ルートの振り分けロジック

```typescript
const rxid = vtecxnext.getParameter('_RXID') ?? ''
if (rxid) {
  await vtecxnext.loginWithRxid(rxid)
} else if (!data.oldpswd) {
  return vtecxnext.response(401, { feed: { title: 'Authentication error.' } })
}
await vtecxnext.changepass(data.newpswd, data.oldpswd, data.passresetToken)
```

---

## グループ管理

### グループディレクトリの事前登録

グループ（`/_group/admin` など）は folderacls.json に定義し、`pnpm upload:folderacls` でサーバーに登録する。

```json
{
  "contributor": [
    { "uri": "urn:vte.cx:acl:/_group/$admin,CURD" },
    { "uri": "urn:vte.cx:acl:+,CURDE" }
  ],
  "link": [{ "___rel": "self", "___href": "/_group/admin" }]
}
```

### メソッド一覧

| メソッド                         | 用途                                           |
| -------------------------------- | ---------------------------------------------- |
| `addGroup(group)`                | 自分をグループに登録（グループがなければ作成） |
| `joinGroup(group)`               | 管理者に招待されたグループへの参加確認         |
| `addGroupByAdmin(uids, group)`   | 管理者が指定ユーザーをグループに追加           |
| `leaveGroup(group)`              | グループから脱退                               |
| `leaveGroupByAdmin(uids, group)` | 管理者がユーザーをグループから削除             |
| `isGroupMember(uri)`             | ログインユーザーのグループ所属確認             |
| `getGroups()`                    | サービス内の全グループ一覧取得                 |

```typescript
await vtecxnext.addGroup('/_group/sales')
await vtecxnext.joinGroup('/_group/sales')
await vtecxnext.addGroupByAdmin([uid], '/_group/sales')
const isAdmin = await vtecxnext.isGroupMember('/_group/admin')
const groups = await vtecxnext.getGroups()
```

---

## エイリアス（横断検索）

エイリアスは、同じエントリに対して複数のパスから参照する仕組み。`link` 配列に `rel="alternate"` を追加することで設定する。

**エイリアスの主な用途は「逆引き」の実現です。** 例えば「あるユーザーが担当する顧客一覧」を取得したいとき、顧客側にインデックスを設けなくてもエイリアスパスで `getFeed` すれば顧客エントリが直接返る。

```
一次パス（正引き）:  /crm/customer/{cid}/member/{uid}  ← getFeed('/crm/customer/{cid}/member')
エイリアス（逆引き): /crm/member/{uid}/{cid}           ← getFeed('/crm/member/{uid}')
```

### 登録

エイリアスは別エントリを作るのではなく、**既存エントリの `link` 配列に `rel="alternate"` を追加**する。親パスの登録とエントリ更新を同一 `put()` にまとめる（親パスを先頭に配置すること）。

```typescript
const customer = await vtecxnext.getEntry(`/crm/customer/${cid}`)
const existingLinks = (customer as any)?.link ?? []

await vtecxnext.put({
  feed: {
    entry: [
      { link: [{ ___rel: 'self', ___href: `/crm/member/${uid}` }], contributor },  // 親パス（先頭）
      {
        ...(customer as any),
        link: [...existingLinks, { ___rel: 'alternate', ___href: `/crm/member/${uid}/${cid}` }],
      },
    ]
  }
})
```

### 逆引き検索

```typescript
const feed = await vtecxnext.getFeed(`/crm/member/${uid}`)
// feed は VtecxApp.Entry[]（顧客エントリが直接返る。別途 getEntry 不要）
```

### 担当者一覧の取得

```typescript
const customer = await vtecxnext.getEntry(`/crm/customer/${cid}`)
const memberUids = ((customer as any)?.link ?? [])
  .filter((l: any) => l.___rel === 'alternate' && l.___href?.startsWith('/crm/member/'))
  .map((l: any) => (l.___href as string).split('/')[3])
```

### 削除

```typescript
const customer = await vtecxnext.getEntry(`/crm/customer/${cid}`)
const updatedLinks = ((customer as any)?.link ?? [])
  .filter((l: any) => !(l.___rel === 'alternate' && l.___href === `/crm/member/${uid}/${cid}`))
await vtecxnext.put({ feed: { entry: [{ ...(customer as any), link: updatedLinks }] } })
```

### エイリアスパスの権限設定

エイリアスパスも `folderacls.json` に登録する。

```json
{ "contributor": [{ "uri": "urn:vte.cx:acl:/_group/$admin,CURD" }, { "uri": "urn:vte.cx:acl:+,CURDE" }],
  "link": [{ "___rel": "self", "___href": "/crm/member" }] }
```

---

## データ登録・更新

> **可能な限り `put()` の `feed.entry` にまとめて、API 呼び出しを1回にすること。**  
> これは vte.cx を使った実装における最重要の設計・実装方針である。

| 観点           | 個別 PUT（複数回）                         | まとめた PUT（1回）  |
| -------------- | ------------------------------------------ | -------------------- |
| パフォーマンス | N回のネットワーク往復                      | 1回のみ              |
| 原子性         | 途中でエラーが起きると中途半端な状態になる | 全件成功 or 全件失敗 |
| サーバー負荷   | リクエスト数が多い                         | 少ない               |

```typescript
// ✅ 複数エントリを1回の put() にまとめる
await vtecxnext.put({
  feed: {
    entry: [
      { link: [{ ___rel: 'self', ___href: '/crm/customer/001' }], customer: {...}, contributor },
      { link: [{ ___rel: 'self', ___href: '/crm/customer/002' }], customer: {...}, contributor },
    ]
  }
})
```

### ⚠️ 親子データは必ず親を先に配置する

親パスが存在しない状態で子エントリを登録しようとすると `Parent path is required` エラーになる。

```typescript
// ✅ 正しい順序: 親 → 子パス
await vtecxnext.put({
  feed: {
    entry: [
      { link: [{ ___rel: 'self', ___href: '/crm/customer/001' }], customer: {...}, contributor },
      { link: [{ ___rel: 'self', ___href: '/crm/customer/001/contact' }], contributor },
      { link: [{ ___rel: 'self', ___href: '/crm/customer/001/activity' }], contributor },
    ]
  }
})
// ❌ 誤った順序: 子パスを先に置くと Parent path is required エラー
```

---

## メール設定（properties.xml）

`setup/_settings/properties.xml` を編集して `pnpm upload:properties` を実行することで、ユーザー登録メールとパスワードリセットメールが有効になる。

### `/_settings/properties` — サーバー設定

```xml
<entry>
  <link href="/_settings/properties" rel="self"/>
  <rights>
_recaptcha.sitekey=your-recaptcha-site-key
_mail.user=apikey
_mail.password=SG.your-sendgrid-api-key
_mail.from=no-reply@your-domain.com
_mail.from.personal=Your Service Name
_mail.transport.protocol=smtps
_mail.smtp.host=smtp.sendgrid.net
_mail.smtp.port=587
_mail.smtp.auth=true
  </rights>
</entry>
```

| キー                  | 説明                                         |
| --------------------- | -------------------------------------------- |
| `_recaptcha.sitekey`  | reCAPTCHA v2 サイトキー                      |
| `_mail.user`          | SMTPユーザー名（SendGrid は固定値 `apikey`） |
| `_mail.password`      | SMTPパスワード（SendGrid APIキー）           |
| `_mail.from`          | 送信元メールアドレス                         |
| `_mail.from.personal` | 送信者表示名                                 |
| `_mail.smtp.host`     | SMTPホスト（SendGrid: `smtp.sendgrid.net`）  |
| `_mail.smtp.port`     | SMTPポート（`587`）                          |

### `/_settings/adduser` — ユーザー登録確認メール

```xml
<entry>
  <link href="/_settings/adduser" rel="self"/>
  <title>ユーザ登録のお申込み確認</title>
  <subtitle>text</subtitle>
  <summary>本登録URLをクリックしてください。

${VTECXNEXT_URL}/signup-completion?_RXID=${RXID}

トップページ ${URL}</summary>
  <content type="text/html"><![CDATA[<html>
<body>
<p>本登録URLをクリックしてください。</p>
<p><a href="${VTECXNEXT_URL}/signup-completion?_RXID=${RXID}">本登録はこちら</a></p>
<p>トップページ: <a href="${URL}">${URL}</a></p>
</body>
</html>]]></content>
</entry>
```

### `/_settings/passreset` — パスワードリセットメール

```xml
<entry>
  <link href="/_settings/passreset" rel="self"/>
  <title>パスワード変更</title>
  <subtitle>text</subtitle>
  <summary>以下のURLをクリックしてパスワード変更を行ってください。

${VTECXNEXT_URL}/change-password?${PASSRESET_TOKEN}</summary>
  <content type="text/html"><![CDATA[<html>
<body>
<p>以下のURLをクリックしてパスワード変更を行ってください。</p>
<p><a href="${VTECXNEXT_URL}/change-password?${PASSRESET_TOKEN}">パスワード変更はこちら</a></p>
</body>
</html>]]></content>
</entry>
```

| 変数                 | 内容                                                             |
| -------------------- | ---------------------------------------------------------------- |
| `${VTECXNEXT_URL}`   | アプリのベースURL（`.env.local` の `NEXT_PUBLIC_VTECXNEXT_URL`） |
| `${RXID}`            | 本登録トークン（フレームワークが自動生成）                       |
| `${PASSRESET_TOKEN}` | パスワードリセットトークン（フレームワークが自動生成）           |

---

## 管理画面

管理画面 URL: https://admin.vte.cx/index.html

> **各サービスの管理画面へのアクセス手順:** https://admin.vte.cx にログイン → サービス一覧から該当サービスの「管理画面」ボタンを押下 → そのサービスの管理画面（エンドポイント管理・ユーザー管理など）が開く。

### エンドポイント管理

`folderacls.json` に相当するディレクトリパスとアクセス権限を管理する。`pnpm upload:folderacls` でアップロードした後、管理画面でも確認・追加編集が可能。

### スキーマ管理

`template.xml` のアップロードで作成されたスキーマを管理画面から確認できる。インデックス・フィールド ACL・暗号化の設定は `template.xml` の `<rights>` タグに記述する。

### データ確認

登録したデータはエンドポイント管理から参照できる。

1. https://admin.vte.cx/index.html にログイン
2. サービス一覧から対象サービスの「管理画面」を押下
3. 左メニューの「エンドポイント管理」をクリック

---

## よくあるエラー

### Parent path is required.

```
VtecxNextError: Parent path is required. /your/path
```

**原因**: 登録しようとしたパスの親ディレクトリが vte.cx サーバーに存在しない。

**対処**: `folderacls.json` に不足している親パスを追加して `pnpm upload:folderacls` を実行する。

例: `/crm/user/${uid}` への登録が失敗する場合 → `/crm` と `/crm/user` の両方が必要。

---

### Key is required.

```
VtecxNextError: Key is required.
```

**原因**: `put()` で送信するエントリに `link` が指定されていない。

**対処**: エントリに `link` を明示する。

```typescript
const entry = {
  link: [{ ___rel: 'self', ___href: '/crm/user/uid123' }],
  user: { ... }
}
```

#### `id` フィールドについて

| フィールド                  | 用途                       | 形式                             | 必須       |
| --------------------------- | -------------------------- | -------------------------------- | ---------- |
| `link[___rel=self].___href` | リソースのパス（キー）指定 | `/path/to/entry`                 | PUT で必須 |
| `id`                        | 楽観的排他制御（競合判定） | `{path},{revision}` 例: `/foo,3` | 任意       |

- `id` を含めると、サーバーが現在のリビジョンと照合し、不一致の場合 **409 Conflict** を返す
- `id` を省略すると競合チェックなしで強制上書き

---

### 403 Forbidden

**原因**: アクセス権限がない、またはセッションが切れている。

**対処**:

- セッション切れ → ログイン画面へ自動遷移
- `folderacls.json` のパスに権限が付与されていない → 権限設定を見直して `pnpm upload:folderacls` を再実行

> localhost と `https://{サービス名}.vte.cx` はセッションが独立している。localhost で確認する場合は localhost でログインし直す必要がある。

---

### 401 Unauthorized

**原因**: 認証情報が間違っている（IDまたはパスワードの誤り）。

---

### 204 No Content

**意味**: エラーではなく、**対象データが存在しない**ことを示す正常レスポンス。

**対処**: エラーとして扱わず、「データなし」の状態として処理する。

```typescript
const entry = await vtecxnext.getEntry('/crm/customer/0000000001')
if (entry == null) {
  // データが存在しない（204）
}
```

---

### 409 Conflict

**原因**: `id`（リビジョン）が不一致（楽観的排他制御）。

**対処**: `getEntry` で最新取得し `id` を更新してから再 PUT する。

---

### Already specified

**原因**: 同一フィールドを複数行でインデックス設定している。

**対処**: `field:/path1|/path2` で1行にまとめる。
