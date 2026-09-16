# vte.cx batchjob

この文書は、vte.cx の batchjob を実装・登録・運用するための仕様と実践的な指針をまとめたものです。開発者だけでなく、AI コーディングエージェントが実装判断に利用できることを目的とします。

## 概要

vte.cx の batchjob は、定期処理用の JavaScript モジュールを vte.cx サービスへ登録し、サービスプロパティに cron 形式の起動時刻を設定して実行する仕組みです。

- 実行モジュールは `/_html/batchjob` 配下へ登録する。
- モジュールは `run` 関数を公開する。
- 実行時には、初期化済みの `VtecxNext` インスタンスが引数として渡される。
- 渡されるインスタンスは、対象サービスのサービス管理者権限で各機能を利用できる。
- 起動スケジュールは `/_settings/properties` Entry の `rights` に設定する。

通常は TypeScript で実装し、Rspack で JavaScript にトランスパイルしてアップロードします。

## モジュールの形式

### JavaScript

```javascript
exports.run = async function (vtecxnext) {
  // vtecxnext を使って処理する
};
```

### TypeScript

型だけを参照する場合は `import type` を推奨します。

```typescript
import type { VtecxNext } from '@vtecx/vtecxnext'

exports.run = async function (vtecxnext: VtecxNext) {
  // vtecxnext を使って処理する
}
```

次の点を守ってください。

- 公開する関数名は `run` とする。
- 必要な非同期処理はすべて `await` し、完了前に `run` を終了させない。
- 業務処理は `run` から開始し、モジュールのトップレベルでは実行しない。
- 引数として渡された `vtecxnext` を使用する。
- batchjob 内で `VtecxNext` を再生成したり、APIKey や AccessToken で再認証したりしない。

## ソースファイルの配置

標準的なプロジェクト構成では、TypeScript ファイルを次の場所に置きます。

```text
src/batchjob/{モジュール名}.ts
```

例えば `src/batchjob/daily-report.ts` は、アップロード後に次の JavaScript として登録されます。

```text
/_html/batchjob/daily-report.js
```

スケジュール設定で指定するモジュール名は、通常、パスと拡張子を除いた `daily-report` です。

### `/_html/batchjob` の事前登録

初回アップロードの前に、登録先である `/_html/batchjob` フォルダーが vte.cx サービス上に存在している必要があります。フォルダーがない状態で Rspack のアップロードを実行するとエラーになります。

プロジェクトの `setup/_settings/htmlfolders.xml` に `/_html/batchjob` を定義します。

```xml
<feed>
  <entry>
    <contributor>
      <uri>urn:vte.cx:acl:/_group/$admin,CRUD</uri>
    </contributor>
    <link rel="self" href="/_html/batchjob" />
  </entry>
</feed>
```

実際のファイルでは、プロジェクトの権限設計に応じた `contributor` を設定してください。既存の `/_settings` や `/_html` の定義がある場合は削除せず、`/_html/batchjob` の Entry を追加します。

対象サービスへログインした後、次のコマンドでフォルダー定義をアップロードします。

```bash
pnpm upload:htmlfolders
```

これは通常、サービスごとの初回セットアップ時、または `htmlfolders.xml` を変更したときに実行します。すでに `/_html/batchjob` が登録済みであれば、batchjob モジュールを更新するたびに実行する必要はありません。

## ビルドとアップロード

対象の vte.cx サービスへログインし、初回は `/_html/batchjob` を登録してから、Rspack コマンドを実行します。

```bash
pnpm run login
pnpm upload:htmlfolders
pnpm exec rspack --env entry=/batchjob/{モジュール名}.ts
```

`/_html/batchjob` が登録済みの場合、モジュール更新時は次のコマンドだけで構いません。

```bash
pnpm exec rspack --env entry=/batchjob/{モジュール名}.ts
```

例:

```bash
pnpm exec rspack --env entry=/batchjob/daily-report.ts
```

このコマンドは、プロジェクトの Rspack 設定に従って TypeScript を Node.js 向け JavaScript に変換し、`/_html/batchjob` 配下へアップロードします。

> `rspack` の設定やアップロード用プラグインはプロジェクト側に必要です。コマンドがローカルビルドだけでなく対象サービスへのアップロードも行う構成の場合、本番環境に対して不用意に実行しないでください。

## 起動時間の登録

対象サービスの `/_settings/properties` Entry の `rights` に、次の形式で設定します。

```properties
_batchjob.{ジョブ名}={分} {時} {日} {月} {曜日} {モジュール名}
```

| 項目 | 内容 |
| --- | --- |
| ジョブ名 | 設定を識別する一意な名前 |
| 分 | cron の分フィールド |
| 時 | cron の時フィールド |
| 日 | cron の日フィールド |
| 月 | cron の月フィールド |
| 曜日 | cron の曜日フィールド |
| モジュール名 | `/_html/batchjob` 配下に登録した JavaScript の拡張子を除く名前 |

### 設定例

5 分ごとに `queue-worker` を実行:

```properties
_batchjob.queue_worker=*/5 * * * * queue-worker
```

毎日 2:30 に `daily-report` を実行:

```properties
_batchjob.daily_report=30 2 * * * daily-report
```

毎週月曜日の 9:00 に `weekly-summary` を実行:

```properties
_batchjob.weekly_summary=0 9 * * 1 weekly-summary
```

ジョブ名とモジュール名は別の値です。右端のモジュール名が、登録した JavaScript と一致していることを確認してください。

タイムゾーン、曜日の数値表現、再試行、最大実行時間など、サービスや実行環境に依存する挙動は推測せず、対象環境で確認してください。

## `VtecxNext` の利用

`run` に渡される `vtecxnext` は、対象サービスに接続済みです。Entry の取得、検索、登録、更新、プロパティ取得、ログ出力などに利用します。

```typescript
import type { VtecxNext } from '@vtecx/vtecxnext'

exports.run = async function (vtecxnext: VtecxNext) {
  const entries = await vtecxnext.getFeed(
    '/example/job?f&example.status-eq-queued&l=10'
  )

  for (const entry of entries) {
    // Entry を処理する
  }
}
```

利用できるメソッドと引数は、プロジェクトで使用している `@vtecx/vtecxnext` のバージョンに従ってください。

## 禁止されている構文

batchjobでは以下が禁止されています。
TypeScript AST で判定しています。

- import
- export
- class
- super
- eval
- Function
- AsyncFunction
- GeneratorFunction
- require
- process
- globalThis
- global
- XMLHttpRequest
- WebSocket
- Worker
- WebAssembly
- Proxy

### vte.cx データ操作の注意点

- フィールド検索を行う URL には `f` パラメータを付ける。
- 一度に取得する Entry 数には `l` などで上限を設ける。
- 複数 Entry は、可能であれば一回の `put()` にまとめる。
- 親子 Entry を同時に作成する場合は、親を先に登録する。
- 既存 Entry の更新では、`id` の revision と既存の `contributor` を保持する。
- revision の競合を無視して上書きしない。
- 独自データはテンプレートで定義したフィールドへ保存し、標準 Atom フィールドを代用しない。
- 自動生成された型定義を手作業で編集しない。
- 権限制御は Entry やフォルダーの ACL／contributor を利用し、コード内へ独自のロール判定を重複実装しない。

## サービスプロパティ

batchjob の設定値は、必要に応じて `property()` から取得します。

```typescript
const enabled = await vtecxnext.property('_example.batch_enabled')

if (enabled !== 'true') {
  return
}
```

設定例:

```properties
_example.batch_enabled=true
_example.batch_limit=10
```

API キーなどの秘密情報を扱う場合は、次の原則を守ります。

- 秘密情報をソースコード、Git、テストデータ、ドキュメントへ埋め込まない。
- 実値をログや例外メッセージへ出力しない。
- 設定例にはプレースホルダーだけを記載する。
- `/_settings/properties` を更新するときは既存設定を保持し、関係のない設定を上書きしない。
- 設定 Entry の閲覧・更新権限を適切に制限する。

## 冪等性と多重実行への備え

定期実行では、同じジョブが重なって起動する、途中で失敗する、実行結果が不明なまま再実行される可能性を考慮します。厳密に一回だけ実行されると仮定してはいけません。

キューを処理する場合は、例えば次の状態を Entry に保存します。

```text
queued -> processing -> completed
                     -> failed
```

推奨事項:

- `queued` など、処理対象の状態を明示的に検索する。
- 1 回の起動で処理する件数に上限を設ける。
- 実処理を始める前に `processing` へ更新する。
- revision 競合が発生した Entry を重複処理しない。
- 外部システムへの送信には、可能であれば冪等キーを使用する。
- 完了した処理を自動的に繰り返さない。
- 長い処理は工程ごとの完了状態を保存し、途中から再開できるようにする。
- 失敗時に自動再試行して安全かどうかを処理単位で判断する。

特にメール送信、決済、外部 API による作成処理など、副作用のある操作では、vte.cx 側の状態だけでなく送信先の結果も確認できる設計にします。

## エラー処理とログ

処理結果や障害調査に必要な情報は、`vtecxnext.log()` など、利用中の SDK が提供するログ機能で記録します。

```typescript
await vtecxnext.log(
  'daily-report completed. requested=10 completed=9 failed=1',
  'daily-report',
  'INFO'
)
```

ログに含める情報:

- ジョブ名
- 開始・終了と処理時間
- 対象件数、成功件数、失敗件数
- Entry を特定するための秘密情報を含まない識別子
- 一般化したエラー種別

ログに含めない情報:

- APIKey、AccessToken、Authorization ヘッダー、Cookie
- パスワードやセッショントークン
- 個人情報を含む Entry 全体
- 外部 API のリクエスト／レスポンス全文
- 秘密情報を含む可能性がある例外オブジェクトや HTTP ヘッダー全体

エラーを捕捉する場合も、失敗状態の保存と安全なログ記録を行ったうえで、呼び出し元に失敗を伝える必要があるケースでは例外を握りつぶさないでください。

## 外部 HTTP 通信

batchjob から外部サービスへ通信する場合は、次を実装します。

- 接続タイムアウトと応答タイムアウトを設定する。
- レスポンスサイズと取得件数を制限する。
- HTTP ステータスと Content-Type を検証する。
- リトライ回数と待機時間に上限を設ける。
- 外部から与えられた URL を利用する場合は、許可するスキームと接続先を検証する。
- localhost、ループバック、プライベート IP、リンクローカルなどへの意図しないアクセスを防ぐ。
- リダイレクト先も再検証する。
- 外部 API の戻り値を構造検証してから Entry に保存する。

ブラウザー自動操作にはブラウザーバイナリなどの実行環境が必要です。batchjob の実行イメージがそれらを提供することを確認できない場合は、Playwright などを前提にせず、対応する別の実行環境へ処理を分離してください。

## 実装例

次は、キューから最大 10 件を取得して処理する最小例です。Entry のカスタムフィールド名や `put()` の具体的な引数形式は、対象プロジェクトのテンプレートと SDK バージョンに合わせてください。

```typescript
import type { VtecxNext } from '@vtecx/vtecxnext'

exports.run = async function (vtecxnext: VtecxNext) {
  const enabled = await vtecxnext.property('_example.batch_enabled')
  if (enabled !== 'true') {
    return
  }

  const entries = await vtecxnext.getFeed(
    '/example/job?f&example.status-eq-queued&l=10'
  )

  let completed = 0
  let failed = 0

  for (const entry of entries) {
    try {
      // 1. revision を保持したまま processing へ更新する
      // 2. 必要な処理を実行する
      // 3. completed と処理結果を保存する
      completed += 1
    } catch (error) {
      // 秘密情報を含まない形で failed とエラー概要を保存する
      failed += 1
    }
  }

  await vtecxnext.log(
    `example batch completed. requested=${entries.length} completed=${completed} failed=${failed}`,
    'example-batch',
    failed === 0 ? 'INFO' : 'WARN'
  )
}
```

## デプロイ手順

1. `src/batchjob/{モジュール名}.ts` を実装する。
2. 型チェック、Lint、単体テストを実行する。
3. 対象環境とログイン先を確認する。
4. 初回は `setup/_settings/htmlfolders.xml` に `/_html/batchjob` が定義されていることを確認する。
5. 初回は `pnpm upload:htmlfolders` で登録先フォルダーをアップロードする。
6. Rspack コマンドで JavaScript をビルドし、アップロードする。
7. `/_html/batchjob/{モジュール名}.js` が登録されたことを確認する。
8. 必要なサービスプロパティを設定する。
9. `/_settings/properties` Entry の `rights` に `_batchjob.*` のスケジュールを登録する。
10. 少量のデータまたは副作用を抑えた設定で動作確認する。
11. 実行ログ、Entry の状態遷移、外部処理の結果を確認する。

## AI コーディングエージェント向けチェックリスト

実装またはレビュー時は、少なくとも次を確認してください。

- [ ] `exports.run = async function (vtecxnext)` の形式になっている。
- [ ] 引数の `vtecxnext` を使用し、認証情報を埋め込んでいない。
- [ ] `setup/_settings/htmlfolders.xml` に `/_html/batchjob` が定義されている。
- [ ] 初回アップロード前に `pnpm upload:htmlfolders` を実行している。
- [ ] ソースと登録先が `batchjob/{モジュール名}` で対応している。
- [ ] スケジュール右端のモジュール名が登録ファイルと一致している。
- [ ] 検索 URL に必要な `f` と件数上限がある。
- [ ] 多重実行と revision 競合を考慮している。
- [ ] 処理が冪等、または再実行可否が明示されている。
- [ ] すべての非同期処理を `await` している。
- [ ] タイムアウト、取得件数、レスポンスサイズに上限がある。
- [ ] 秘密情報や個人情報をログへ出力しない。
- [ ] 既存の Entry、`contributor`、サービスプロパティを不用意に破壊しない。
- [ ] SDK や実行環境について未確認の挙動を断定していない。
- [ ] 本番アップロード前に対象サービスを確認している。

## よくある問題

### アップロード時に `/_html/batchjob` が存在しないエラーになる

Rspack のアップロード先フォルダーがサービス上に未登録です。

1. `setup/_settings/htmlfolders.xml` に `/_html/batchjob` の Entry を追加する。
2. 対象サービスへログインする。
3. `pnpm upload:htmlfolders` を実行する。
4. Rspack のアップロードコマンドを再実行する。

```bash
pnpm run login
pnpm upload:htmlfolders
pnpm exec rspack --env entry=/batchjob/{モジュール名}.ts
```

`/_html` だけが存在していても、子フォルダーの `/_html/batchjob` が未登録であればアップロードできません。

### ジョブが起動しない

- JavaScript が `/_html/batchjob` 配下へ登録されているか確認する。
- `exports.run` が公開されているか確認する。
- `_batchjob.*` が `/_settings/properties` Entry の `rights` に設定されているか確認する。
- 設定右端のモジュール名と登録ファイル名を照合する。
- cron 式と対象環境のタイムゾーンを確認する。

### ジョブは起動するがデータを取得できない

- 対象パスとテンプレートのフィールド名を確認する。
- フィールド検索 URL に `f` があるか確認する。
- `VtecxNext` の利用方法が、インストール済み SDK のバージョンに合っているか確認する。

### 同じ処理が複数回実行される

- revision 競合を検出し、競合した側で処理を続行しない。
- 外部処理に冪等キーまたは処理済み判定を追加する。
- 実行間隔より処理時間が長くなっていないか確認する。

### ローカルでは動くが batchjob では失敗する

- Node.js のバージョンと利用可能な Web API を確認する。
- Rspack によるバンドル結果と外部依存関係を確認する。
- ファイルシステム、ブラウザー、ネイティブモジュールなど、実行環境に存在すると仮定した機能を見直す。
- 実行時間、メモリ、外部通信の制限を対象環境で確認する。

## 未確認事項の扱い

次の項目は、実装前または運用開始前に対象サービスの仕様・設定で確認してください。

- cron の基準タイムゾーン
- 同一ジョブの同時実行制御
- 失敗時の自動再試行回数と間隔
- 最大実行時間、メモリ、ペイロードの制限
- 利用可能な Node.js バージョンと組み込み API
- 外部ネットワークへの接続制限
- ログの保存先、保持期間、閲覧権限

不明な挙動をコード側で都合よく仮定せず、安全側の制限と冪等性を実装してください。
