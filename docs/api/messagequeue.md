# メッセージキュー

ログインユーザー間で短いメッセージを受け渡すためのメソッド群。
WebSocket の代わりに、**送信側がキューへ積み、受信側がポーリングで取り出す**ことで準リアルタイムの通知を実現する。

- 送信先はチャネルのグループのメンバー（UID・アカウント・グループ全員）。
- 受信側がメッセージキューを使わない設定（OFF）のときは、キューに積まずに **Push 通知**へ切り替わる。
- 取り出したメッセージはキューから削除される（1 回だけ受け取れる）。

---

## 概要

### 処理の流れ

```
受信側                                送信側
  │ setMessageQueueStatus(true, ch)     │
  │  （自分の使用設定を ON）             │
  │                                     │ setMessageQueue(entries, ch)
  │                                     │  → 宛先ごとのキューへ登録
  │ getMessageQueue(ch)  ← 数秒おき      │
  │  → 自分宛てのメッセージを取得         │
  │  → 取得したメッセージは削除（非同期）  │
```

### チャネル

`channel` は **`/{チャネル名}/{グループ}`** の形式で指定する。

| 部分 | 説明 |
| --- | --- |
| 1 階層目 | 任意のチャネル名（用途ごとに分ける） |
| 2 階層目以降 | グループ。送信者はこのグループに参加している必要があり、送信先はこのグループのメンバーから選ばれる |

```typescript
const channel = '/chat/_group/sales' // チャネル「chat」・グループ /_group/sales
```

> グループ部分の表記（`_group/` を含めるか）は、利用前に実環境で確認すること。

- **`/` で始める**（SDK が `Key must start with a slash.` の 400 を返す）。
- **`[]` を含めない**。サーバはチャネルを 1 階層のキーに変換する際に `/` を `[]` へ置き換えるため、`[]` を含むチャネルはエラーになる。

### サーバ内部のデータ

| キー | 内容 | ACL |
| --- | --- | --- |
| `/_user/{UID}/mqstatus/{チャネルの変換値}` | ユーザーごと・チャネルごとの使用設定（title=チャネル、summary=flag） | ユーザー自身の領域 |
| `/_mq/{チャネルの変換値}@{UID}` | 宛先ユーザーのキュー（連番は `addids` で採番） | `{グループ},CE` と `{UID},RDE` |
| `/_mq/{チャネルの変換値}@{UID}/{連番}` | メッセージ本体（連番は 20 桁の 0 埋め） | 同上 |

`/_mq` はシステムディレクトリ（ACL は `+,CE`）。アプリの `folderacls.json` に定義する必要はない。

---

## メッセージキュー使用設定

### `setMessageQueueStatus(flag, channel)`

```typescript
setMessageQueueStatus(flag: boolean, channel: string): Promise<boolean>
```

**ログインユーザー自身**のメッセージキュー使用設定を、チャネル単位で登録する。

| 引数 | 型 | 説明 |
| --- | --- | --- |
| `flag` | `boolean` | `true`: 受信メッセージをキューに登録する／`false`: キューに登録せず Push 通知を送る |
| `channel` | `string` | チャネル（`/{チャネル名}/{グループ}`） |

- 未ログインはエラー。
- **設定が無いユーザーは OFF とみなされる**。キューで受け取りたいユーザーは、受信を始める前に自分で `true` を設定する。
- 他のユーザーの設定は変更できない（受信側が自分で呼ぶ）。

```typescript
const channel = '/chat/_group/sales'
await vtecxnext.setMessageQueueStatus(true, channel)
```

---

### `getMessageQueueStatus(channel)`

```typescript
getMessageQueueStatus(channel: string): Promise<MessageResponse>
```

ログインユーザー自身のメッセージキュー使用設定を取得する。値は `/_user/{UID}/mqstatus/{チャネルの変換値}` の summary に保存された flag。

- 未ログインはエラー。
- 画面を開くたびに `setMessageQueueStatus` を呼ぶのではなく、これで確認して未設定のときだけ設定すると書き込みを減らせる。

```typescript
const res = await vtecxnext.getMessageQueueStatus(channel)
```

---

## メッセージ送信

### `setMessageQueue(feed, channel)`

```typescript
setMessageQueue(feed: Entry[], channel: string): Promise<boolean>
```

メッセージエントリを宛先ユーザーのキューへ登録する。

| 引数 | 型 | 説明 |
| --- | --- | --- |
| `feed` | `Entry[]` | メッセージエントリの配列（`{ feed: { entry } }` ではなくエントリの配列を渡す） |
| `channel` | `string` | チャネル（`/{チャネル名}/{グループ}`） |

#### メッセージエントリ

| 項目 | 説明 |
| --- | --- |
| `summary` | メッセージ本文 |
| `link` (`rel="to"`) の `___href` | 送信先。**UID・アカウント・`*`（グループ全員）** のいずれか。複数指定可 |
| `title` | Push 通知に切り替わった場合のタイトル |
| `subtitle` | Push 通知に切り替わった場合のサブタイトル（Expo 用） |
| `content` | Push 通知に切り替わった場合の本文。**空なら Push 通知しない** |
| `category` | Push 通知に切り替わった場合の data（Expo 用・key-value）。`_$scheme` がキー、`_$label` が値 |
| `rights` | `'true'` を指定すると、**Push 通知に切り替わる場合でも Push 通知しない** |

```typescript
await vtecxnext.setMessageQueue(
  [
    {
      link: [{ ___rel: 'to', ___href: targetUid }],
      summary: JSON.stringify({ type: 'message.created', id: '0001' }),
      rights: 'true', // Push 通知は不要（キューだけで通知する）
    },
  ],
  '/chat/_group/sales'
)
```

#### 処理内容

1. 未ログイン、または**チャネルのグループに参加していない**場合はエラー。
2. グループのメンバー（UID・アカウント）を取得する。**送信者自身は除く**。
3. 送信先がグループ（`*`）ならメンバー全員、UID・アカウントならそのユーザーを対象にする。
4. 対象ユーザーごとに使用設定を確認する。
   - **ON**: キュー `/_mq/{チャネルの変換値}@{UID}/{連番}` にメッセージエントリをそのまま登録する。
   - **OFF（未設定を含む）**: Push 通知を送る（下記）。

#### Push 通知に切り替わるケース

1. 送信先ユーザーのメッセージキュー使用設定が OFF（未設定を含む）。
2. グループ管理者の署名があり、送信先ユーザーの署名がされていない（グループ追加直後の通知を想定。送信先ユーザーのエイリアスが無い場合は対象外）。
3. キューに登録されたまま取り出されずに一定時間が過ぎた（[未受信メッセージの期限切れ](#未受信メッセージの期限切れ)）。

ただし次の場合は Push 通知しない。

- メッセージエントリの `rights` が `true`。
- メッセージエントリの `content` が空。
- `{グループ}/{UID}/disable_notification` エントリの `contributor.uri` に `urn:vte.cx:disable.notification:true` がある（受信側の Push 通知 OFF 設定）。

Push 通知は FCM または Expo サーバへ送られる。**キューだけで通知したい場合は `rights: 'true'` を付け、`content` を空にする。**

---

## メッセージ受信

### `getMessageQueue(channel)`

```typescript
getMessageQueue(channel: string): Promise<Entry[] | null>
```

ログインユーザー自身のキュー（`/_mq/{チャネルの変換値}@{UID}`）からメッセージを取得する。

| 引数 | 型 | 説明 |
| --- | --- | --- |
| `channel` | `string` | チャネル（`/{チャネル名}/{グループ}`） |

| 戻り値 | 説明 |
| --- | --- |
| `Entry[]` | メッセージエントリの配列（本文は `summary`） |
| `null` | メッセージが無い（HTTP 204） |

- 続きのデータがあれば、取得件数がシステム設定 `_fetch.limit` 以上になるまで検索を繰り返す。
- **返したメッセージはキューから削除される（非同期）**。フォルダ削除ではなく、受信したメッセージのキーを個別に削除する。

```typescript
const entries = await vtecxnext.getMessageQueue('/chat/_group/sales')
for (const e of entries ?? []) {
  const message = JSON.parse(e.summary ?? '{}')
  // ...
}
```

---

## 未受信メッセージの期限切れ

バッチジョブサーバが **5 分おき**にキューを確認し、`updated` が「現在時刻 −（設定値）分」より古いメッセージについて次の処理を行う。

1. `rights` が `true` でなく、受信側の Push 通知 OFF 設定も無ければ、Push 通知を送る。
2. メッセージを削除する。

| 設定（`/_settings/properties`） | 説明 | 既定値 |
| --- | --- | --- |
| `_messagequeue.expire.min` | 取り出されないメッセージを Push 通知・削除するまでの時間（分） | `5` |

---

## デバッグログ

`/_settings/properties` に `_debuglog.notification=true` を設定すると、次の実行内容がログエントリに出力される（WebSocket・Push 通知のデバッグログと同じ設定）。

- メッセージキュー使用設定
- メッセージ送信
- メッセージ受信
- 未受信メッセージの期限切れによる削除

---

## 実装例（Next.js）

受信はブラウザから直接ではなく、API Route 経由で行う。チャネルはサーバで決め、クライアントから受け取らない。

```typescript
// app/api/notice/route.ts
import { NextRequest } from 'next/server'
import { VtecxNext } from '@vtecx/vtecxnext'

const CHANNEL = '/chat/_group/sales'

// 受信の準備（画面を開いたときに 1 回）
export const POST = async (req: NextRequest) => {
  const vtecxnext = new VtecxNext(req)
  if (!(await vtecxnext.isGroupMember('/_group/sales'))) {
    await vtecxnext.addGroup('/_group/sales')
  }
  await vtecxnext.setMessageQueueStatus(true, CHANNEL)
  return vtecxnext.response(200, { feed: { title: 'ok' } })
}

// 受信（画面から数秒おきに呼ぶ）
export const GET = async (req: NextRequest) => {
  const vtecxnext = new VtecxNext(req)
  const entries = (await vtecxnext.getMessageQueue(CHANNEL)) ?? []
  return vtecxnext.response(200, entries)
}
```

```typescript
// 画面側: 前の取得が終わってから次を始める（setInterval は使わない）
useEffect(() => {
  let timer: ReturnType<typeof setTimeout> | undefined
  let stopped = false
  const poll = async () => {
    if (stopped) return
    if (!document.hidden) {
      try {
        const res = await fetch('/api/notice', { headers: { 'X-Requested-With': 'XMLHttpRequest' } })
        if (res.ok) handleMessages(await res.json())
      } catch {
        // 次の回で取り直す
      }
    }
    timer = setTimeout(poll, 2000)
  }
  poll()
  return () => {
    stopped = true
    clearTimeout(timer)
  }
}, [])
```

---

## 注意事項

- **キューを業務データの正にしない。** 取り出すと消え、取り出されなければ期限切れで消える。キューには「更新があった」という合図（ID など）だけを載せ、受信側はそれを受けて本体を取り直す。画面を開いたとき・タブが前面に戻ったときも本体を取り直す。
- **同じメッセージが 2 回届くことがある。** 取り出したメッセージの削除は非同期なので、短い間隔で続けて取得すると、削除前のメッセージがもう一度返ることがある。受信処理は 2 回実行しても結果が変わらないように作る。
- **キューはユーザー（UID）ごとに 1 つ。** 同じユーザーが複数のタブ・端末で開いていると、メッセージを受け取るのはどれか 1 つだけになる。
- **送信にはグループへの参加が必要。** 異なる組織のユーザー同士でやり取りする場合は、両者が参加する通知専用のグループを用意する。そのグループにデータの ACL を付けなければ、参加してもデータは見えない。
- **送信のたびにグループのメンバーを取得する。** メンバーが非常に多いグループでは送信に時間がかかる。
- **Push 通知が不要なら `rights: 'true'` を付ける。** 付けないと、使用設定が未設定のユーザーや、キューを取り出さないまま 5 分過ぎたユーザーに Push 通知が送られる。
