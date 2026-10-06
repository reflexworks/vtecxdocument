# メール・通知

メール送信・プッシュ通知を行うメソッド群。メッセージキューは [messagequeue.md](messagequeue.md) を参照。

---

## メール送信

### `sendMail(entry, to, cc?, bcc?, attachments?)`

```typescript
sendMail(
  entry: any,
  to: string[],
  cc?: string[],
  bcc?: string[],
  attachments?: string[]
): Promise<boolean>
```

メールを送信する。件名・本文は `entry`（title = 件名、content = 本文など）で指定する。

| 引数 | 型 | 説明 |
| --- | --- | --- |
| `entry` | `any` | メールの件名・本文を含むエントリ |
| `to` | `string[]` | 送信先メールアドレスの配列 |
| `cc` | `string[]?` | CC |
| `bcc` | `string[]?` | BCC |
| `attachments` | `string[]?` | 添付ファイルの URI 配列 |

```typescript
await vtecxnext.sendMail(
  { title: 'お知らせ', summary: 'テキスト本文', content: { ______text: '<p>HTML本文</p>' } },
  ['user@example.com']
)
```

---

## プッシュ通知

### `pushNotification(message, to, title?, subtitle?, imageUrl?, data?)`

```typescript
pushNotification(
  message: string,
  to: string[],
  title?: string,
  subtitle?: string,
  imageUrl?: string,
  data?: any
): Promise<boolean>
```

モバイルデバイスへのプッシュ通知を送信する。

| 引数 | 型 | 説明 |
| --- | --- | --- |
| `message` | `string` | 通知本文 |
| `to` | `string[]` | 送信先（デバイストークン／グループなど）の配列 |
| `title` | `string?` | 通知タイトル |
| `subtitle` | `string?` | サブタイトル |
| `imageUrl` | `string?` | 画像 URL |
| `data` | `any?` | 追加データ |

```typescript
await vtecxnext.pushNotification('メッセージが届きました', [deviceToken], '新しいメッセージ')
```

---

## メッセージキュー

`setMessageQueueStatus` / `getMessageQueueStatus` / `setMessageQueue` / `getMessageQueue` は [messagequeue.md](messagequeue.md) を参照。
