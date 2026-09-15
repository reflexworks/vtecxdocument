# ユーティリティ

リクエスト処理・レスポンス生成・ログ・プロパティ取得などの汎用メソッド群。

---

## リクエスト処理

### `checkXRequestedWith()`

```typescript
checkXRequestedWith(): Response | undefined
```

`X-Requested-With` ヘッダが正しく設定されているか検証する（CSRF 対策）。
不正な場合は 401 Response を返す。正常の場合は `undefined`。

API ルートの先頭で必ず呼び出す。

```typescript
const result = vtecxnext.checkXRequestedWith()
if (result) return result
```

---

### `getParameter(name)`

```typescript
getParameter(name: string): string | undefined
```

リクエストのクエリパラメータを取得する。存在しない場合は `undefined`。

```typescript
const rxid = vtecxnext.getParameter('_RXID') ?? ''
const page = parseInt(vtecxnext.getParameter('n') ?? '1', 10)
```

---

### `hasParameter(name)`

```typescript
hasParameter(name: string): boolean
```

指定したクエリパラメータが存在するかどうかを返す。値のないフラグ系パラメータの確認に使用。

```typescript
const isDebug = vtecxnext.hasParameter('debug')
```

---

### `buffer(req)`

```typescript
buffer(readable?: Readable): Promise<Uint8Array>
```

リクエストボディ（または指定した `Readable` ストリーム）を `Uint8Array` として取得する。`readable` を省略した場合はコンストラクタに渡したリクエストのボディを読み取る。

---

## レスポンス生成

### `response(status, body?)`

```typescript
response(status?: number, data?: any): Response
```

`Response` オブジェクトを生成して返す。API ルートの最後に使用する。

```typescript
return vtecxnext.response(200, { feed: { entry: entries } })
return vtecxnext.response(401, { feed: { title: 'Unauthorized.' } })
return vtecxnext.response(204)  // データなし
```

---

### `setResponseHeader(name, value)`

```typescript
setResponseHeader(name: string, value: string): void
```

レスポンスヘッダを設定する。`response()` を呼び出す前に設定する。

```typescript
vtecxnext.setResponseHeader('Cache-Control', 'no-store')
return vtecxnext.response(200, data)
```

---

## レスポンス（feed 形式）

### `sendMessage(statusCode, message)`

```typescript
sendMessage(statusCode: number, message: string): Response
```

feed 形式のレスポンスを生成して返す。`response()` の代替で、ステータスコードとメッセージを渡す。

```typescript
return vtecxnext.sendMessage(400, 'Invalid parameter.')
```

---

## 設定・ログ

### `property(key)`

```typescript
property(key: string): Promise<string | null>
```

`/_settings/properties` に設定されたプロパティ値を取得する。

```typescript
const siteKey = await vtecxnext.property('_recaptcha.sitekey')
```

---

### `log(message, title?, subtitle?)`

```typescript
log(message: string, title?: string, subtitle?: string): Promise<boolean>
```

vte.cx にログエントリを登録する。`title` のデフォルトは `'JavaScript'`。

```typescript
await vtecxnext.log('処理完了', 'MyApp', 'INFO')
await vtecxnext.log('エラーが発生しました', 'MyApp', 'ERROR')
```

---

## reCAPTCHA検証

### `verifyRecaptcha(token, action)`

```typescript
verifyRecaptcha(token: string, action: string): Promise<RecaptchaVerifyResult>
```

reCAPTCHA トークンを検証する。

| 引数 | 型 | 説明 |
| --- | --- | --- |
| `token` | `string` | reCAPTCHA トークン |
| `action` | `string` | reCAPTCHA アクション |

戻り値の型:

```typescript
type RecaptchaVerifyResult = {
  valid: boolean
  action?: string
  reason?: string
}
```

```typescript
const result = await vtecxnext.verifyRecaptcha(token, 'submit')
if (!result.valid) {
  return vtecxnext.response(400, { feed: { title: 'reCAPTCHA verification failed.', subtitle: result.reason } })
}
```

---

## 型チェック

### `isVtecxNextError(e)`

```typescript
isVtecxNextError(e: unknown): e is VtecxNextError
```

エラーが `VtecxNextError` かどうかを判定する型ガード。

```typescript
import { isVtecxNextError } from '@vtecx/vtecxnext'

try {
  await vtecxnext.getEntry('/crm/customer/001')
} catch (e) {
  if (isVtecxNextError(e)) {
    console.error(e.status, e.message)
  }
}
```

---

## 文字列ユーティリティ

### `isBlank(str)`

```typescript
isBlank(val: any): boolean
```

値が `null` / `undefined` / 空文字 / 空配列のいずれかかどうかを返す。

```typescript
if (vtecxnext.isBlank(name)) {
  return vtecxnext.response(400, { feed: { title: 'Name is required.' } })
}
```

---

### `null2blank(str)`

```typescript
null2blank(str: string | null | undefined): string
```

`null` または `undefined` を空文字列に変換する。

```typescript
const name = vtecxnext.null2blank(entry?.customer?.name)
```
