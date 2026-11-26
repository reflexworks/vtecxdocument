# vtecxnext API クイックリファレンス

`@vtecx/vtecxnext` SDK の **よく使うメソッドの抜粋と実装例** です。
アプリ実装でまず参照する入口として使ってください。

> **全メソッドの網羅リファレンスは [api/](api/index.md) を参照してください。**
> シグネチャ・引数・戻り値・型定義はカテゴリ別ファイル（`api/*.md`）に集約しています。
> このファイルは重複を避け、頻出メソッドの使い方に絞っています。

公式リファレンス: https://zenn.dev/vtecx/articles/6a02ab2440ef05

| カテゴリ | 網羅リファレンス |
| --- | --- |
| 認証・セッション | [api/auth.md](api/auth.md) |
| データ操作（取得・登録・削除・ページング） | [api/data.md](api/data.md) |
| ユーザー管理 | [api/user.md](api/user.md) |
| グループ管理 | [api/group.md](api/group.md) |
| 採番・カウンタ | [api/counter.md](api/counter.md) |
| サーバーセッション | [api/session.md](api/session.md) |
| コンテンツ・ファイル | [api/content.md](api/content.md) |
| メール・通知 | [api/notify.md](api/notify.md) |
| 外部DB連携（BigQuery / RDB） | [api/db.md](api/db.md) |
| PDF・署名 | [api/pdf.md](api/pdf.md) |
| OAuth・TOTP | [api/oauth.md](api/oauth.md) |
| ACL・エイリアス | [api/acl.md](api/acl.md) |
| ユーティリティ | [api/util.md](api/util.md) |

---

## 初期化

```typescript
import { VtecxNext } from '@vtecx/vtecxnext'

// Next.js API Route 内で初期化
export const GET = async (req: NextRequest): Promise<Response> => {
  const vtecxnext = new VtecxNext(req)
  // ...
}
```

コンストラクタは `req?: NextRequest` と `accessToken?: string` を受け取る。

---

## 認証・セッション情報の取得

→ 一覧は [api/auth.md](api/auth.md)

```typescript
const uid = await vtecxnext.uid()          // ログインユーザーの UID（未ログインはエラー）
const email = await vtecxnext.account()    // アカウント名（メールアドレス）
const serviceName = await vtecxnext.service()  // サービス名

const loggedIn = await vtecxnext.isLoggedin()
if (!loggedIn) {
  return vtecxnext.response(401, { feed: { title: 'Unauthorized.' } })
}
```

---

## データ取得

→ 一覧・戻り値の型は [api/data.md](api/data.md)

```typescript
// 一覧（ページング不要）。データなしは null / undefined（204）
const entries = await vtecxnext.getFeed('/crm/customer')

// 1件取得。データなしは null（204）
const entry = await vtecxnext.getEntry('/crm/customer/0000000001')

// 件数取得。0件は null
const total = await vtecxnext.count('/crm/customer?f&customer.status-eq-active')
```

**取得の注意**（詳細は [framework.md](framework.md#7-フィールドインデックスの実務的制約インデックス設定の補足)）:

- `getFeed` は既定のページサイズで切り捨てられ得る。全件が要る少量データは `?l=*` を付ける。
- インデックスが効くのは **先頭の検索条件だけ**。絞り込みの強いインデックス項目を先頭に置く。
- インデックスを後から追加した場合、既存データは `_updateindex` で作り直すまで**黙って検索結果から漏れる**。
- メモリでの照合が上限に達すると、件数が足りないまま続きのカーソルと `rights="*"` を付けて返る。0 件に見えても続きがあることがある。

### ページング

`getPageWithPagination` が「カーソル作成 → ページ取得」の2ステップを内部で自動処理する。

```typescript
const n = parseInt(vtecxnext.getParameter('n') ?? '1', 10)
const entries = await vtecxnext.getPageWithPagination('/crm/customer?l=25', n)
// n=1 のとき自動的に pagination() を呼び出す。データなし: undefined
```

カーソルを手動作成する場合は `pagination()`。

```typescript
const info = await vtecxnext.pagination('/crm/customer?l=25', '1,50')
// info.lastPageNumber: 最終ページ番号（0 = データなし）
// info.hasNext: true = 50ページ以降にもデータがある
```

---

## データ登録・更新

→ 一覧は [api/data.md](api/data.md) ／ まとめ方の方針は [framework.md](framework.md#データ登録更新)

`post` はキーを自動採番、`put` はキー（`link` の `___href`）を指定して登録・更新する。
**複数エントリは1回の `put()` にまとめること。**

```typescript
// post: uri 配下に自動採番して登録
await vtecxnext.post({
  feed: {
    entry: [{
      customer: { name: '株式会社テスト' },
      contributor: [{ uri: 'urn:vte.cx:acl:/_group/$admin,CURD' }],
    }]
  }
}, '/crm/customer')

// put（新規登録）: キーを指定して登録。ACL（contributor）は作成時に付ける
await vtecxnext.put({
  feed: {
    entry: [
      {
        link: [{ ___rel: 'self', ___href: '/crm/customer/0000000001' }],
        customer: { name: '株式会社テスト', status: 'active' },
        contributor: [
          { uri: 'urn:vte.cx:acl:/_group/$admin,CURD' },
          { uri: `urn:vte.cx:acl:${uid},CURD` },
        ],
      },
    ]
  }
})

// put（更新）: contributor は送らない。最上位の項目（customer）は丸ごと送る
const existing = await vtecxnext.getEntry('/crm/customer/0000000001')
await vtecxnext.put({
  feed: {
    entry: [
      {
        link: [{ ___rel: 'self', ___href: '/crm/customer/0000000001' }],
        id: existing?.id, // 楽観ロック。他の更新と競合したら 409
        customer: { ...existing?.customer, status: 'inactive' },
      },
    ]
  }
})
```

**更新の注意**（詳細は [framework.md](framework.md#contributor-は作成時に確定させる-更新では送らない)）:

- **更新では contributor を送らない。** contributor を書き換えられるのは作成者と `/_group/$admin` だけで、それ以外のユーザーが送ると、値が同じでも `Property 'contributor' is not writeable.` で失敗する。省略すれば現在の ACL が残る。
- **最上位の項目は丸ごと置き換わる。** `customer: { status }` だけを送ると `customer.name` などが消える。変更しない値も含めて送る。
- `id`（`{キー},{リビジョン}`）を付けると、リビジョンが変わっていれば 409。`id` を省略しても、同じキーへの書き込みが同時に来ると 409 になる。「読んでから書く」処理は `id` を付け、409 なら読み直して再実行する。

---

## データ削除

→ 一覧は [api/data.md](api/data.md)

```typescript
// 1件削除（revision を渡すと競合チェックあり）
await vtecxnext.deleteEntry('/crm/customer/0000000001')

// エイリアスのキーを指定すると、本体は残りエイリアスだけが外れる
await vtecxnext.deleteEntry(`/crm/member/${uid}/0000000001`)

// 複数一括削除
await vtecxnext.deleteEntries({
  feed: {
    entry: [
      { link: [{ ___rel: 'self', ___href: '/crm/customer/0000000001' }] },
      { link: [{ ___rel: 'self', ___href: '/crm/customer/0000000002' }] },
    ]
  }
})
```

- `deleteEntry` / `deleteEntries` / `deleteFolder` は **BDB だけ**を削除する。`putBDBQ` で BQ にも書いたデータは `deleteBDBQ` で削除する（下の「BDB と BigQuery の同時更新」）。

---

## BDB と BigQuery の同時更新

→ 一覧は [api/db.md](api/db.md) ／ 保証範囲の詳細は [framework.md](framework.md#34-bdb-と-bq-の同時更新bdbqの保証範囲)

```typescript
// BDB に書き込み、BQ には後から非同期で反映する
// 引数: (エントリの配列, 親URI?, テーブル名の対応?, async?, isbulk?)
await vtecxnext.putBDBQ([
  {
    link: [{ ___rel: 'self', ___href: '/crm/customer/0000000001' }],
    customer: { name: '株式会社テスト', status: 'active' },
  },
])

// BDB と BQ から削除（BQ には削除を表す行が追加される）
await vtecxnext.deleteBDBQ(['/crm/customer/0000000001'])

// BQ を読む: key ごとに updated が最大の行を取り、deleted = false で絞る
const rows = await vtecxnext.execBQ(
  `SELECT f.* FROM \`dataset.customer\` AS f
   INNER JOIN (SELECT key, MAX(updated) AS updated FROM \`dataset.customer\`
               WHERE key LIKE ? GROUP BY key) AS k
   ON f.key = k.key AND f.updated = k.updated
   WHERE f.deleted = false`,
  ['/crm/customer/%'],
)
```

- **API の成功は「BDB に確定した」ことを意味し、「BQ に反映された」ことは意味しない。** 書き込み直後の表示や確認は BDB（`getEntry` / `getFeed`）で読む。
- BQ への書き込みが失敗すると `/_bdbq` に記録され、定期チェックで **その時点の BDB の内容** が送り直される。計画的な停止では未処理の書き込みを終えてから止まるが、サーバが異常終了した場合に限り、まれに届かないことがある。
- BQ のテーブルは追記専用。同じ `key` の行が複数あるので、**必ず `key` ごとの最新行を選んで読む**。`deleted = false` だけで絞ると古い行も返る。
- 同じキーを短時間に続けて更新すると、BQ の最新行が古い版になることがある。厳密さが必要な場合は `id` のリビジョンが最大の行を選ぶ。
- フォルダ・グループ定義など、実体（項目の値）を持たないエントリは BQ に書けない。これらは `put` で書く。
- まだ 1 件も書いていないエンティティのテーブルは存在しないので、`execBQ` は `Not found: Table` で失敗する。読み取りではこれを「0 件」として扱う。

---

## パスワード変更

→ 一覧は [api/user.md](api/user.md) ／ 2フローの詳細は [framework.md](framework.md#パスワード変更)

```typescript
// ログイン済みフロー（現在のパスワードで変更）
await vtecxnext.changepass(newpswd, oldpswd)

// メールリセットフロー（未ログイン）
await vtecxnext.loginWithRxid(rxid)
await vtecxnext.changepass(newpswd, undefined, passresetToken)
```

パスワードは `getHashpass(password)`（`@vtecx/vtecxauth`）でハッシュしてから渡す。

---

## 採番・カウンタ

→ 一覧は [api/counter.md](api/counter.md)

```typescript
// 連番採番（"開始,終了" 形式の文字列を返す）
const range = await vtecxnext.allocids('/crm/customer/_ids', 1)
const id = range.split(',')[0].padStart(10, '0')  // "0000000042"

// カウンタ加算・参照
const count = await vtecxnext.addids('/crm/counter', 1)
const current = await vtecxnext.getids('/crm/counter')
```

---

## グループ管理

→ 一覧は [api/group.md](api/group.md) ／ グループ設計は [framework.md](framework.md#グループ管理)

```typescript
const isAdmin = await vtecxnext.isAdmin()
// isGroupMember('/_group/$admin') の省略形。true: 所属, false: 非所属

const isSales = await vtecxnext.isGroupMember('/_group/sales')
```

---

## ユーティリティ

→ 一覧は [api/util.md](api/util.md)

```typescript
// クエリパラメータ取得
const rxid = vtecxnext.getParameter('_RXID') ?? ''
const page = parseInt(vtecxnext.getParameter('n') ?? '1', 10)

// CSRF 対策（API ルートの先頭で必ず呼ぶ）
const result = vtecxnext.checkXRequestedWith()
if (result) return result

// レスポンス生成
return vtecxnext.response(200, { feed: { title: 'ok' } })
return vtecxnext.response(401, { feed: { title: 'Unauthorized.' } })

// ログ登録（title デフォルト: 'JavaScript'）
await vtecxnext.log('処理完了', 'MyApp', 'INFO')
```

---

## @vtecx/vtecxauth

クライアントサイド（ブラウザ）向けユーティリティ。

```typescript
import { getHashpass } from '@vtecx/vtecxauth'

const hashedPassword = getHashpass(rawPassword)
// パスワードをハッシュ化（API 送信前に使用）
```

---

## エラーハンドリング

エラー時の戻り値: `{ feed: { title: string } }`

```typescript
try {
  const entries = await vtecxnext.getFeed('/crm/customer')
  return vtecxnext.response(200, { feed: { entry: entries } })
} catch (e) {
  return apiutil.responseError(e, 'api customers')
}
```

エラー型判定:

```typescript
import { isVtecxNextError } from '@vtecx/vtecxnext'

if (isVtecxNextError(e)) {
  // e.status / e.message でステータス・メッセージ取得
}
```

主なステータス（詳細は [framework.md](framework.md#よくあるエラー)）:

| status | 意味 | 対処 |
| --- | --- | --- |
| 204 | データなし（エラーではない） | 戻り値が `null` / `undefined` |
| 400 | 入力エラー。`Property 'contributor' is not writeable.`、template の範囲外（`Minimum number of '…' not met.`）など | 更新で contributor を送っていないか、値の範囲を確認する |
| 403 | 権限なし・セッション切れ | ACL を確認する。セッション切れはログイン画面へ |
| 409 | リビジョン不一致、または同じキーへの同時書き込み | 最新を読み直して再実行する |
