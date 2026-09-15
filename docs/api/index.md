# vtecxnext API インデックス

`@vtecx/vtecxnext` v2.3.1 の全メソッド一覧です。

公式リファレンス: https://zenn.dev/vtecx/articles/6a02ab2440ef05

---

## カテゴリ一覧

| カテゴリ | ファイル | 主なメソッド |
| --- | --- | --- |
| 認証・セッション | [auth.md](auth.md) | `uid`, `login`, `logout`, `loginWithRxid`, `isLoggedin` |
| データ操作 | [data.md](data.md) | `getEntry`, `getFeed`, `put`, `post`, `deleteEntry`, `pagination` |
| ユーザー管理 | [user.md](user.md) | `adduser`, `changepass`, `revokeuser`, `deleteuser` |
| グループ管理 | [group.md](group.md) | `addGroup`, `joinGroup`, `leaveGroup`, `isGroupMember`, `isAdmin` |
| 採番・カウンタ | [counter.md](counter.md) | `allocids`, `addids`, `getids`, `setids` |
| サーバーセッション | [session.md](session.md) | `setSessionFeed`, `getSessionFeed`, `setSessionString` |
| コンテンツ・ファイル | [content.md](content.md) | `putcontent`, `getcontent`, `savefiles`, `getSignedUrlToPutContent` |
| メール・通知 | [notify.md](notify.md) | `sendMail`, `pushNotification`, `setMessageQueue` |
| 外部DB連携 | [db.md](db.md) | `getBQ`, `execBQ`, `queryRDB`, `execRDB`, `postBDBQ` |
| PDF・署名 | [pdf.md](pdf.md) | `toPdf`, `putSignature`, `checkSignature` |
| OAuth・TOTP | [oauth.md](oauth.md) | `oauthLine`, `getTotpLink`, `createTotp`, `loginWithTotp` |
| ACL・エイリアス | [acl.md](acl.md) | `addacl`, `removeacl`, `addalias`, `removealias` |
| ユーティリティ | [util.md](util.md) | `checkXRequestedWith`, `response`, `getParameter`, `log`, `property`, `verifyRecaptcha` |

---

## 全メソッド索引

### 認証・セッション
`isLoggedin` / `uid` / `account` / `service` / `now` / `rxid` / `whoami` /
`login` / `loginWithRxid` / `loginWithTotp` / `logout`

### データ操作
`getEntry` / `getFeed` / `getFeedResponse` / `count` / `countResponse` /
`post` / `put` / `deleteEntry` / `deleteEntries` / `deleteFolder` / `clearFolder` /
`pagination` / `getPage` / `getPageWithPagination`

### ユーザー管理
`adduser` / `adduserByAdmin` / `adduserByGroupadmin` /
`passreset` / `changepass` / `changepassByAdmin` /
`changeaccount` / `changeaccount_verify` /
`userstatus` / `revokeuser` / `revokeusers` / `activateuser` / `activateusers` /
`canceluser` / `deleteuser` / `deleteusers`

### グループ管理
`addGroup` / `addGroupByAdmin` / `joinGroup` / `leaveGroup` / `leaveGroupByAdmin` /
`noGroupMember` / `getGroups` / `isGroupMember` / `isAdmin` /
`createGroupadmin` / `deleteGroupadmin`

### 採番・カウンタ
`allocids` / `addids` / `getids` / `setids` / `rangeids` / `getRangeids`

### サーバーセッション
`setSessionFeed` / `getSessionFeed` / `deleteSessionFeed` /
`setSessionEntry` / `getSessionEntry` / `deleteSessionEntry` /
`setSessionString` / `getSessionString` / `deleteSessionString` /
`setSessionLong` / `getSessionLong` / `deleteSessionLong` / `incrementSession`

### コンテンツ・ファイル
`getcontenturl` / `savefiles` / `savefilesBySize` /
`putcontent` / `putcontentBySize` / `postcontent` / `deletecontent` / `getcontent` /
`getSignedUrlToPutContent` / `getSignedUrlToPostContent` / `getSignedUrlToGetContent`

### メール・通知
`sendMail` / `pushNotification` /
`setMessageQueueStatus` / `getMessageQueueStatus` / `setMessageQueue` / `getMessageQueue`

### 外部DB連携
`postBQ` / `deleteBQ` / `getBQ` / `execBQ` / `getBQCsv` /
`postBDBQ` / `putBDBQ` / `deleteBDBQ` /
`queryRDB` / `queryRDBCsv` / `execRDB`

### PDF・署名
`toPdf` / `putSignature` / `putSignatures` / `deleteSignature` / `checkSignature`

### OAuth・TOTP
`oauthLine` / `oauthCallbackLine` / `mergeOAuthUserLine` /
`getTotpLink` / `createTotp` / `deleteTotp` / `loginWithTotp` / `changeTdid`

### ACL・エイリアス
`addacl` / `removeacl` / `addalias` / `removealias`

### ユーティリティ
`getParameter` / `hasParameter` / `buffer` / `isBlank` / `null2blank` /
`checkXRequestedWith` / `setResponseHeader` / `response` / `sendMessage` /
`property` / `log` / `isVtecxNextError` / `verifyRecaptcha`

---

## 初期化

```typescript
import { VtecxNext } from '@vtecx/vtecxnext'

// Next.js API Route 内で初期化
const vtecxnext = new VtecxNext(req)

// バッチ処理（アクセストークン使用）
const vtecxnext = new VtecxNext(undefined, accessToken)
```

## エラークラス

| クラス | 説明 |
| --- | --- |
| `VtecxNextError` | vte.cx 関連エラー |
| `FetchError` | fetch 処理エラー（VtecxNextError を継承） |

```typescript
import { isVtecxNextError } from '@vtecx/vtecxnext'

if (isVtecxNextError(e)) {
  // VtecxNextError の処理
}
```
