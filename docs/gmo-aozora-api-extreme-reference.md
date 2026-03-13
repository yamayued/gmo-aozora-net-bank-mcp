# GMOあおぞらネット銀行 API 極限リファレンス

最終更新: 2026-03-13  
対象: GMOあおぞらネット銀行の公開APIを使って実装する開発者、設計者、MCPサーバー実装者

## 0. この文書の位置づけ

この文書は、GMOあおぞらネット銀行の公開APIについて、公式の一次情報を横断して実務向けに再構成した高密度リファレンスです。

- 公式API開発者ポータル
- 公式API仕様書PDF
- 公式SDK群

を元に、認証、エンドポイント群、パラメータ制約、ページング、代表的な落とし穴、MCP化する際の設計観点まで整理しています。

注意:

- この文書は公開リポジトリ向けです。実口座情報、実クレデンシャル、実証明書、実取引データは一切含めません。
- 一部の表現は複数の公式ソースを統合した要約です。
- 公式SDK間で一部表記ゆれがあるため、実装時は必ず最新ポータルでも再確認してください。

## 1. 全体像

GMOあおぞらネット銀行のAPIは、大きく次の面で構成されています。

1. 認可 API
2. トークン API
3. 法人 API
4. 個人 API
5. Webhook API

公式SDKの構成を見ると、`authorization`、`corporate`、`personal`、`webhook` が明確に分かれています。  
つまり、実装上は「OAuth系」と「業務API系」に分離して考えると理解しやすいです。

### 1.1 API群の役割

| API群 | 役割 |
| --- | --- |
| Auth | 認可コード取得、アクセストークン取得、リフレッシュ |
| Account | 口座一覧、残高、入出金明細、振込入金明細、Visaデビット明細 |
| Transfer | 振込手数料事前照会、振込依頼、振込状況照会 |
| Transfer Common | 振込取消、振込依頼結果照会 |
| Bulk Transfer | 総合振込の手数料照会、依頼、状況照会 |
| Virtual Account | 振込入金口座の発行、一覧、状態変更、解約申込、入金明細照会 |
| Webhook | 通知配信制御、未送信明細取得 |

## 2. 環境とベースURL

公式PHP SDKのREADMEに基づく環境URLは次のとおりです。

### 2.1 認可系

| 環境 | Base URL |
| --- | --- |
| STG | `https://stg-api.gmo-aozora.com/ganb/api/auth/v1` |
| PROD | `https://api.gmo-aozora.com/ganb/api/auth/v1` |

### 2.2 個人API

| 環境 | Base URL |
| --- | --- |
| STG | `https://stg-api.gmo-aozora.com/ganb/api/personal/v1` |
| PROD | `https://api.gmo-aozora.com/ganb/api/personal/v1` |

### 2.3 法人API

| 環境 | Base URL |
| --- | --- |
| STG | `https://stg-api.gmo-aozora.com/ganb/api/corporation/v1` |
| PROD | `https://api.gmo-aozora.com/ganb/api/corporation/v1` |

### 2.4 Webhook API

ここは公式ソース間で表記に揺れがあります。

- PHP SDK README: `https://stg-api.gmo-aozora.com/ganb/api/webhook/v1`
- Java SDK WebhooksApi: `https://stg-pi.gmo-aozora.com/ganb/api/webhooks/v1`

この差異は実装前に必ず最新ポータルで確認してください。  
Webhook系は特にホスト名とパスの整合性をハードコードせず、設定で切り替えられるようにしておくのが安全です。

## 3. 認証と認可

## 3.1 認可フローの基本

公開SDKと認可API文書から読み取れる基本フローは OAuth 2.0 / OpenID Connect 系の認可コードフローです。

1. クライアントが `/authorization` へ遷移URLを生成する
2. ユーザーが認証・認可を行う
3. 認可コードが `redirect_uri` に返る
4. `/token` に認可コードを渡してアクセストークンを取得する
5. 業務API呼び出し時は `x-access-token` にアクセストークンを入れる
6. `offline_access` を要求していた場合はリフレッシュトークンを使って再発行できる

## 3.2 認可エンドポイント

| メソッド | パス | 用途 |
| --- | --- | --- |
| `GET` | `/authorization` | ユーザー認証・認可を得る |

主要パラメータ:

| パラメータ | 必須 | 内容 |
| --- | --- | --- |
| `client_id` | 必須 | 事前発行されるクライアントID |
| `redirect_uri` | 必須 | 事前登録済みのリダイレクトURI |
| `response_type` | 必須 | `code` 固定 |
| `scope` | 必須 | 要求スコープ。複数はスペース区切り |
| `state` | 必須 | CSRF対策用ランダム値 |
| `nonce` | 任意 | IDトークンとクライアントセッション紐付け用 |

重要ポイント:

- `state` は必須級です。戻り時に必ず照合するべきです。
- `nonce` は OIDC 的なリプレイ防止用途です。IDトークンを使うなら毎回生成して検証すべきです。
- リフレッシュトークンが必要なら `scope` に `offline_access` が必要です。

### 3.3 公式SDKに出てくるスコープ例

公式Node.js SDKの認可サンプルには次のスコープ例が出ています。

```text
private:account private:virtual-account private:transfer private:bulk-transfer
```

ここから少なくとも以下の権限概念があることが分かります。

- `private:account`
- `private:virtual-account`
- `private:transfer`
- `private:bulk-transfer`
- `offline_access`

補足:

- Webhook用スコープ名称は確認できる一次情報をこの調査では特定できませんでした。
- 契約形態や利用申込内容によって実際に付与されるスコープは変わる可能性があります。

## 3.4 トークンエンドポイント

| メソッド | パス | 用途 |
| --- | --- | --- |
| `POST` | `/token` | アクセストークン新規発行、再発行 |

公式資料から確認できる仕様:

- 新規発行時の `grant_type` は `authorization_code`
- 再発行時の `grant_type` は `refresh_token`
- クライアント認証方式は少なくとも次の2種
  - `basic`
  - `post`

### 3.5 トークン要求の要点

| 項目 | 新規発行 | 再発行 |
| --- | --- | --- |
| `grant_type` | `authorization_code` | `refresh_token` |
| `code` | 必須 | 不要 |
| `refresh_token` | 不要 | 必須 |
| `redirect_uri` | 必須 | 不要 |
| `client_id` | `post` のとき必須 | `post` のとき必須 |
| `client_secret` | `post` のとき必須 | `post` のとき必須 |

### 3.6 トークン応答

公式SDKの `TokenResponse` から確認できる主要フィールド:

| フィールド | 内容 |
| --- | --- |
| `accessToken` | 業務API呼び出し時に `x-access-token` へ入れる値 |
| `refreshToken` | アクセストークン再発行用 |
| `scope` | 許可されたスコープ |
| `tokenType` | トークン種別 |
| `expiresIn` | 有効期限秒数 |
| `idToken` | JWT形式。任意 |

### 3.7 認証系エラー

Auth系 `ErrorResponse` では次のエラーコードが確認できます。

- `invalid_request`
- `invalid_client`
- `invalid_grant`
- `unauthorized_client`
- `unsupported_grant_type`
- `invalid_scope`
- `server_error`

実装上の要点:

- `invalid_grant` は認可コード不正や `redirect_uri` 不一致で出る
- `invalid_client` はクライアントID/シークレット不整合を疑う
- `invalid_scope` は契約・申込・利用許可の不足も疑う

## 4. 共通実装ルール

## 4.1 認証ヘッダー

業務APIの多くは `x-access-token` を要求します。  
SDK docs の `Authorization` 欄には "No authorization required" と書かれていても、これはSDK生成文書の表現であり、実際には `x-access-token` パラメータが必須です。

つまり実装上は:

- HTTPヘッダー `x-access-token: <access token>`

を必ず付与する前提で考えるべきです。

## 4.2 日付形式

日付系は `YYYY-MM-DD` が基本です。  
時刻付き項目は応答側に `HH:MM:SS+09:00` 形式が登場します。

## 4.3 ページング

多くの照会APIは 500 件上限で、`nextItemKey` によるページングです。

パターン:

1. 初回は `nextItemKey` なし
2. 応答に次キーがあれば次回リクエストへ同一条件で付与
3. これを繰り返して全件取得

注意:

- 初回と条件を変えると意図しない取りこぼしや重複の原因になります。
- Webhookの未送信一覧だけは「404になるまで取り続ける」という別ルールです。

## 4.4 空データ時の扱い

多くの照会APIは「該当なしでも `200 OK`」です。  
Webhook未送信一覧は「該当なしで `404 Not Found`」という別挙動です。

この差はクライアント実装で吸収が必要です。

## 4.5 文字種・コード値

複数の項目で以下が繰り返し出ます。

- 半角数字
- 半角文字
- 半角カナ英数記号
- 振込許容文字

特に振込名義、EDI情報、追加名義カナは文字種制約が厳しいため、MCPツール化するときは入力前バリデーションを持たせた方が安全です。

## 5. 口座系 API

法人・個人の両方で、SDK上は同じAPI群が定義されています。  
以下は法人APIのパスを基準に記載します。個人は `/corporation/` が `/personal/` に変わると考えると理解しやすいです。

## 5.1 口座API一覧

| メソッド | パス | 内容 |
| --- | --- | --- |
| `GET` | `/accounts` | 口座一覧照会 |
| `GET` | `/accounts/balances` | 残高照会 |
| `GET` | `/accounts/transactions` | 入出金明細照会 |
| `GET` | `/accounts/deposit-transactions` | 振込入金明細照会 |
| `GET` | `/accounts/visa-transactions` | Visaデビット取引明細照会 |

## 5.2 口座一覧照会

- 保有する全口座の一覧を取得
- `x-access-token` 必須

MCPで使うなら:

- `list_accounts`
- 結果を `accountId` ごとにキャッシュ

が基本になります。

## 5.3 残高照会

- `accountId` 指定あり: 単一口座
- `accountId` 省略: 全口座

MCP化では `get_balance(account_id?)` の形が自然です。

## 5.4 入出金明細照会

パス: `GET /accounts/transactions`

主な制約:

- 対象は円普通預金口座
- 上限 500件
- ソートは昇順
- `dateFrom` / `dateTo` の組み合わせに応じて対象範囲が変わる
- `nextItemKey` でページング

## 5.5 振込入金明細照会

パス: `GET /accounts/deposit-transactions`

主な制約:

- 対象は円普通預金口座
- 上限 500件
- ソートは昇順
- ページングあり
- 該当なしでも `200 OK`

## 5.6 Visaデビット取引明細照会

パス: `GET /accounts/visa-transactions`

特徴:

- 対象は Visa デビットカード保有口座
- 上限 500件
- ただし 1回の検索総件数が 99,999 件超は `400 Bad Request`
- ソート順は降順
- ページングあり

## 6. 振込 API

## 6.1 API一覧

| 分類 | メソッド | パス | 内容 |
| --- | --- | --- | --- |
| Transfer | `POST` | `/transfer/transferfee` | 振込手数料事前照会 |
| Transfer | `POST` | `/transfer/request` | 振込依頼 |
| Transfer | `GET` | `/transfer/status` | 振込状況照会 |
| Transfer Common | `POST` | `/transfer/cancel` | 振込取消依頼 |
| Transfer Common | `GET` | `/transfer/request-result` | 振込依頼結果照会 |

## 6.2 振込手数料事前照会

パス: `POST /transfer/transferfee`

用途:

- 振込前に入力妥当性と手数料を確認する

公式ドキュメント上の重要ポイント:

- 最大 99件まで登録可能
- 1件なら通常振込
- 2件以上なら一括振込扱い
- 手数料は実行時点で変動し得る
- 無料回数やポイントは計算対象から除外される

実装方針:

- `request` の前に必ず `transferfee` を呼ぶワークフローが安全
- UIやMCPでは「見積」と「実行」を分ける

## 6.3 振込依頼

パス: `POST /transfer/request`

用途:

- 振込または振込予約を申請する

主な制約:

- 最大 99件
- 1件なら通常振込
- 2件以上なら一括振込扱い

### 6.3.1 TransferRequest の重要フィールド

| フィールド | 内容 |
| --- | --- |
| `accountId` | 出金元口座ID |
| `remitterName` | 振込依頼人名。省略時は口座名義 |
| `transferDesignatedDate` | 振込指定日 |
| `transferDateHolidayCode` | 非営業日扱い。`1=翌営業日` `2=前営業日` `3=エラー` |
| `totalCount` | 合計件数。1〜99。1件なら省略可 |
| `totalAmount` | 合計金額。1件なら省略可 |
| `applyComment` | ビジネスID管理利用中の申請コメント |
| `transfers` | 振込明細配列 |

### 6.3.2 Transfer 明細の重要フィールド

| フィールド | 内容 |
| --- | --- |
| `itemId` | 明細番号。1件なら省略可 |
| `transferAmount` | 振込金額 |
| `ediInfo` | EDI情報 |
| `beneficiaryBankCode` | 仕向先金融機関コード |
| `beneficiaryBranchCode` | 仕向先支店コード |
| `accountTypeCode` | `1=普通` `2=当座` `4=貯蓄` `9=その他` |
| `accountNumber` | 7桁未満は左ゼロ埋め相当で扱う前提 |
| `beneficiaryName` | 受取人名 |

## 6.4 振込状況照会

パス: `GET /transfer/status`

用途:

- 仕向の振込状況、履歴の照会

主な仕様:

- 上限 500件
- ページングあり
- `queryKeyClass`
  - `1`: 振込申請番号指定
  - `2`: 条件で一括照会
- `requestTransferTerm`
  - `1`: 振込申請受付日基準
  - `2`: 振込指定日基準
- `requestTransferClass`
  - `1`: ALL
  - `2`: 振込申請のみ
  - `3`: 振込受付情報のみ

重要な運用ルール:

- `申請中` 以降が照会対象
- `依頼中`、`作成中` は照会対象外
- APIから作ったものだけでなく、定額自動振込で自動登録された振込も照会対象

## 6.5 振込取消依頼

パス: `POST /transfer/cancel`

用途:

- 振込、振込予約、総合振込の取消申請

重要ルール:

- `申請中` 以降は取消可能
- `依頼中`、`作成中` は取消不可
- 同じ `applyNo` に重複依頼は不可

### 6.5.1 cancelTargetKeyClass

| 値 | 意味 |
| --- | --- |
| `1` | 振込申請取消 |
| `2` | 振込受付取消 |
| `3` | 総合振込申請取消 |
| `4` | 総合振込受付取消 |

法人のビジネスID管理利用状況によって指定可能値が変わる点が重要です。

## 6.6 振込依頼結果照会

パス: `GET /transfer/request-result`

用途:

- 更新系APIの処理結果照会

対象:

- 振込依頼
- 総合振込依頼
- 振込取消依頼

実装上の使い方:

- 非同期っぽい処理結果確認APIとして扱う
- `request` 後に `applyNo` をキーにポーリングする

## 7. 総合振込 API

## 7.1 API一覧

| メソッド | パス | 内容 |
| --- | --- | --- |
| `POST` | `/bulktransfer/transferfee` | 総合振込手数料事前照会 |
| `POST` | `/bulktransfer/request` | 総合振込依頼 |
| `GET` | `/bulktransfer/status` | 総合振込状況照会 |

## 7.2 総合振込手数料事前照会

特徴:

- 実行時の制度変更や税変更で手数料が変わり得る
- ポイントは計算対象外

## 7.3 総合振込依頼

用途:

- 総合振込の申請

### 7.3.1 BulkTransferRequest の重要フィールド

| フィールド | 内容 |
| --- | --- |
| `accountId` | 出金元口座ID |
| `remitterName` | 振込依頼人名 |
| `transferDesignatedDate` | 振込指定日 |
| `transferDateHolidayCode` | 非営業日調整コード |
| `transferDataName` | 総合振込データ識別用メモ |
| `totalCount` | 1〜9,999 |
| `totalAmount` | 合計金額 |
| `applyComment` | 申請コメント |
| `bulkTransfers` | 明細配列 |

### 7.3.2 BulkTransfer 明細の重要フィールド

| フィールド | 内容 |
| --- | --- |
| `itemId` | 明細番号。1〜9999 |
| `beneficiaryBankCode` | 金融機関コード |
| `beneficiaryBranchCode` | 支店コード |
| `accountTypeCode` | 預金種別 |
| `accountNumber` | 口座番号 |
| `beneficiaryName` | 受取人名 |
| `transferAmount` | 金額 |
| `ediInfo` | EDI情報 |
| `identification` | `Y` ならEDI通知、空/NULL/スペースなら通知しない |

参考値で処理に使わないと明記されているフィールドもあります。

- `beneficiaryBankName`
- `beneficiaryBranchName`
- `clearingHouseName`
- `newCode`
- `transferDesignnatedType`

これらは入力しても銀行側で処理判断に使わない前提で設計した方がよいです。

## 7.4 総合振込状況照会

パス: `GET /bulktransfer/status`

重要ポイント:

- 1回で取得できるのは最大500件
- `detailInfoNecessity=true` のときは総合振込明細情報を取る
- `bulktransferItemKey` で明細の取得開始位置を指定
- 1500件を取るなら `1`, `501`, `1001` のように分割
- 対象は現在契約中の総合振込サービスのみ

### 7.4.1 総合振込状況照会の設計上の難所

- 一覧照会と明細照会が同じAPIで切り替わる
- `nextItemKey` と `bulktransferItemKey` の役割が異なる
- UIやMCPでは「申請一覧取得」と「特定申請の明細展開」を別ツールに分けた方が扱いやすい

## 8. 振込入金口座 API

## 8.1 API一覧

| メソッド | パス | 内容 |
| --- | --- | --- |
| `GET` | `/va/deposit-transactions` | 振込入金口座入金明細照会 |
| `POST` | `/va/issue` | 振込入金口座発行 |
| `POST` | `/va/status-change` | 振込入金口座状態変更 |
| `POST` | `/va/list` | 振込入金口座一覧照会 |
| `POST` | `/va/close-request` | 振込入金口座解約申込 |

## 8.2 振込入金口座発行

パス: `POST /va/issue`

重要ポイント:

- 1リクエストで最大 1000口座
- 銀行メンテ時は実行できないため、事前に余剰発行してプールしておくことが推奨されている

### 8.2.1 VaIssueRequest の重要フィールド

| フィールド | 内容 |
| --- | --- |
| `vaTypeCode` | `1=期限型`, `2=継続型` |
| `issueRequestCount` | 発行口座数 |
| `raId` | 入金先口座ID |
| `vaContractAuthKey` | 通常は `NULL` 前提 |
| `vaHolderNameKana` | 追加名義カナ |
| `vaHolderNamePos` | `1=後ろ`, `2=前` |

追加名義カナは特に制約が細かいです。

- 利用記号は限定
- 口座名義カナとの合計40文字以内
- 前置時のスペースルールあり
- 連続スペース不可

MCPツールにする場合はここを前処理で厳格検証すべきです。

## 8.3 振込入金口座状態変更

パス: `POST /va/status-change`

`vaStatusChangeCode`:

| 値 | 意味 |
| --- | --- |
| `1` | 停止 |
| `2` | 再開 |
| `3` | 削除 |

制約:

- 1リクエスト最大100口座

## 8.4 振込入金口座一覧照会

パス: `POST /va/list`

特徴:

- 取得上限 500件
- `nextItemKey` によるページング
- 発行日時昇順
- 多数のフィルタ条件を持つ

主なフィルタ:

- `vaTypeCode`
- `depositAmountExistCode`
- `vaHolderNameKana`
- `vaStatusCodeList`
- `latestDepositDateFrom/To`
- `vaIssueDateFrom/To`
- `expireDateFrom/To`
- `raId`
- `vaIdList`
- `sortItemCode`
- `sortOrderCode`

### 8.4.1 sortItemCode

| 値 | 意味 |
| --- | --- |
| `1` | 有効期限日時 |
| `2` | 最終入金日 |
| `3` | 発行日時 |
| `4` | 最終入金金額 |

### 8.4.2 sortOrderCode

| 値 | 意味 |
| --- | --- |
| `1` | 昇順 |
| `2` | 降順 |

## 8.5 振込入金口座入金明細照会

パス: `GET /va/deposit-transactions`

特徴:

- `raId` または `vaId` のどちらかが事実上必要
- 上限 500件
- `dateFrom/dateTo` は照会日から6ヶ月以内制約
- `sortOrderCode`: `1=昇順`, `2=降順`
- ページングあり

## 8.6 振込入金口座解約申込

パス: `POST /va/close-request`

重要ポイント:

- 解約は受付月の月末に行われる

つまり「即時無効化API」ではなく、「月末解約申込API」と理解した方が運用事故を防げます。

## 9. Webhook API

## 9.1 API一覧

| メソッド | パス | 内容 |
| --- | --- | --- |
| `POST` | `/subscribe` | 通知配信制御 |
| `GET` | `/unsentlist/va-deposit-transaction` | 未送信入金明細取得 |

## 9.2 通知配信制御

パス: `POST /subscribe`

用途:

- イベント通知の配信開始・停止

### 9.2.1 認証方式

Webhook APIの認証は `x-access-token` ではなく、公式文書では:

- 銀行システムが配信先システムに発行した `clientId:clientSecret` を Base64 エンコードした値

を `authorization` に設定すると説明されています。

このため、通常のOAuthアクセストークンベースとは別物として設計した方が安全です。

### 9.2.2 SubscribeRequestBody

| フィールド | 内容 |
| --- | --- |
| `subscribeStatus` | `0=配信停止要求`, `1=配信開始要求` |
| `eventTypes` | イベント種別配列 |

### 9.2.3 EventType

現時点で公開SDKから確認できるイベント種別:

- `va-deposit-transaction`

これは「振込入金口座への入金明細通知」です。

## 9.3 未送信入金明細取得

パス: `GET /unsentlist/va-deposit-transaction`

特徴:

- 配信停止中または送信エラー時の救済取得API
- 未送信/送信エラー明細を一括取得
- 本APIで取得した明細は配信済み扱いになる
- 法人口座および個人事業主口座のみ対象
- 個人口座は対象外
- 取得上限 500件
- 明細が残っている可能性がある限り繰り返し取得
- 明細が無くなったら `404 Not Found`

MCPや運用バッチの実装では:

- `404` をエラーではなく「枯渇シグナル」と扱う
- 再取得ループの停止条件にする

のがポイントです。

## 10. 法人APIと個人APIの違い

公式SDK上では、個人APIと法人APIは非常に近いAPI面を持ちます。  
ただし、各フィールド説明には法人特有の注記が頻繁に出ます。

代表例:

- ビジネスID管理利用中のみ有効な `applyComment`
- ビジネスID管理利用有無で変わる取消区分
- 総合振込機能
- Webhook未送信取得の対象外となる個人口座

このため設計上は次の切り分けが有効です。

- 共通クライアント
- 契約種別ごとの差分バリデータ
- 法人専用ツール
- 個人/法人共通ツール

## 11. 実装時の落とし穴

## 11.1 SDK文書の "No authorization required" を誤解しない

生成ドキュメント上はそう見えても、実際には `x-access-token` パラメータ必須です。  
ここを誤解すると「認証不要API」と勘違いして実装事故になります。

## 11.2 WebhookのURL表記差異

先述の通り、`webhook` / `webhooks`、`stg-api` / `stg-pi` の差異があります。  
固定実装ではなく設定化し、接続テストを最初の工程に入れるべきです。

## 11.3 200で空、404で空、の混在

同じ「該当なし」でもAPIごとに意味が違います。

- 多くの明細照会: `200 OK` + 明細なし
- Webhook未送信一覧: `404 Not Found`

## 11.4 日付条件の意味がAPIごとに違う

同じ `dateFrom` / `dateTo` でも:

- 当日扱い
- 最古からToまで
- 6ヶ月以内制約
- ソート基準の違い

があり、共通の検索UIを作るときに一律に扱うと危険です。

## 11.5 総合振込は単なる大量振込ではない

総合振込は:

- 独自の依頼API
- 独自の状況照会API
- 明細取得フラグ
- 明細開始キー

を持っており、通常振込をループして代替する思想では整理しづらいです。

## 11.6 仮想口座名義の文字種制約

`vaHolderNameKana` は実装事故が起きやすいです。  
入力時点で厳格正規化と長さ検証をしないと、生成失敗の原因になります。

## 12. MCPサーバーに落とし込むなら

## 12.1 最小ツールセット

公開MCPとして作るなら、まずは次のツール群が現実的です。

| MCPツール案 | 対応API |
| --- | --- |
| `list_accounts` | `/accounts` |
| `get_balances` | `/accounts/balances` |
| `get_transactions` | `/accounts/transactions` |
| `get_deposit_transactions` | `/accounts/deposit-transactions` |
| `quote_transfer_fee` | `/transfer/transferfee` |
| `create_transfer_request` | `/transfer/request` |
| `get_transfer_status` | `/transfer/status` |
| `get_transfer_request_result` | `/transfer/request-result` |
| `cancel_transfer` | `/transfer/cancel` |
| `quote_bulk_transfer_fee` | `/bulktransfer/transferfee` |
| `create_bulk_transfer_request` | `/bulktransfer/request` |
| `get_bulk_transfer_status` | `/bulktransfer/status` |
| `issue_virtual_accounts` | `/va/issue` |
| `list_virtual_accounts` | `/va/list` |
| `get_virtual_account_deposits` | `/va/deposit-transactions` |
| `change_virtual_account_status` | `/va/status-change` |
| `close_virtual_account_contract` | `/va/close-request` |
| `set_webhook_subscription` | `/subscribe` |
| `fetch_unsent_webhook_messages` | `/unsentlist/va-deposit-transaction` |

## 12.2 実装順序

おすすめ順:

1. 認可URL生成
2. トークン取得/更新
3. 口座一覧
4. 残高
5. 入出金明細
6. 振込見積
7. 振込依頼
8. 状況照会
9. 仮想口座
10. Webhook

理由:

- まず読み取り系で疎通と権限を固める
- 次に書き込み系を段階的に増やす
- WebhookはURL差異と運用設計が絡むため最後

## 12.3 安全設計

- トークンは永続化するなら暗号化
- `state` と `nonce` を必ず検証
- `x-access-token` をログ出力しない
- `applyNo`、`accountId`、`vaId` も公開リポジトリのログ例ではダミー化
- 本番とSTGの設定を完全分離
- 書き込み系APIは dry-run 相当の見積ツールと対にする

## 12.4 MCP向けのUX設計

LLMは誤操作しやすいため、書き込み系は最低でも2段階に分けると安全です。

例:

1. `quote_transfer_fee`
2. `create_transfer_request`

また、戻り値には次を必ず含めるべきです。

- 送信先口座要約
- 金額
- 指定日
- 受付番号 `applyNo`
- 追跡に使う照会キー

## 13. 公式ソースから見た未確定点

この調査時点で、次は公式ソース間で要確認です。

1. Webhookの正確な本番/STG URL
2. Webhook用スコープの正式名称
3. 個人APIと法人APIの機能差分の最新状態
4. 口座種別ごとの実利用可否

ここは「未確定のままコードに埋めない」が重要です。

## 14. 実装チェックリスト

- 認可URL生成で `state` を保存したか
- `nonce` を保存・検証したか
- `offline_access` の有無を要件化したか
- `x-access-token` を全業務APIで付与しているか
- `nextItemKey` を条件不変で引き継いでいるか
- `404` をWebhook未送信一覧の正常終了として扱っているか
- 振込依頼前に手数料事前照会を挟んでいるか
- 非営業日コードの扱いをUIで明示しているか
- 仮想口座名義の文字制約を前段バリデーションしているか
- STG/PROD/Webhook URLを設定化しているか

## 15. 参考ソース

- 公式API開発者ポータル: <https://api.gmo-aozora.com/ganb/developer/>
- 法人向けAPI仕様書PDF: <https://gmo-aozora.com/business/service/pdf/api-spec-corporate.pdf>
- 公式GitHub組織: <https://github.com/gmoaozora>
- Node.js SDK: <https://github.com/gmoaozora/gmo-aozora-api-nodejs>
- Java SDK: <https://github.com/gmoaozora/gmo-aozora-api-java>
- PHP SDK: <https://github.com/gmoaozora/gmo-aozora-api-php>

## 16. 一言でまとめると

GMOあおぞらAPIは、単純な残高照会APIではなく、

- OAuth認可
- 明細系の厳密なページング
- 振込の事前照会と申請分離
- 総合振込の独立面
- 仮想口座の大量発行/管理
- Webhookの運用面

まで含んだ、かなり実務寄りの銀行APIです。  
MCP化するなら「読み取り系から段階導入」「書き込み系は二段階確認」「環境差異を設定化」が成功パターンです。
