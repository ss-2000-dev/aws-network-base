# AWS WAF ログフィールド リファレンス

> **出典**: [AWS WAF ログフィールド公式ドキュメント](https://docs.aws.amazon.com/ja_jp/waf/latest/developerguide/logging-fields.html)

---
## 概要

AWS WAF はウェブ ACL（保護パック）を通過するトラフィックのログを記録します。  
以下に、ログに含まれるすべてのフィールドを分類して整理します。

---
## 1. リクエスト基本情報

| フィールド | 型 | 説明 |
|---|---|---|
| `timestamp` | number | タイムスタンプ（ミリ秒単位） |
| `requestId` | string | 基盤となるホストサービスが生成するリクエスト ID。ALB ではトレース ID、それ以外ではリクエスト ID |
| `clientIp` | string | リクエストを送信したクライアントの IP アドレス |
| `country` | string | リクエストの送信国。特定できない場合は `-` |
| `args` | string | クエリ文字列 |
| `uri` | string | リクエストの URI |
| `fragment` | string | URL の `#` 以降の部分（例: `#section2`） |
| `httpMethod` | string | リクエストの HTTP メソッド（GET / POST など） |
| `httpVersion` | string | HTTP のバージョン |
| `headers` | array | リクエストヘッダーの一覧 |
| `formatVersion` | number | ログの形式バージョン |

---
## 2. リクエスト元情報

| フィールド            | 説明                                                           |
| ---------------- | ------------------------------------------------------------ |
| `clientAsn`      | クライアント IP に紐づく AS 番号（ASN）。**ASN 一致ステートメントが使用されている場合のみ**記録される |
| `forwardedAsn`   | リクエストを転送したエンティティの IP に紐づく ASN                                |
| `ja3Fingerprint` | TLS Client Hello から生成される 32 文字のハッシュ。CloudFront と ALB のみ利用可能  |
| `ja4Fingerprint` | TLS Client Hello から生成される 36 文字のハッシュ。CloudFront と ALB のみ利用可能  |

---

## 3. リソース識別情報

| フィールド | 説明 |
|---|---|
| `webaclId` | ウェブ ACL（保護パック）の GUID |
| `httpSourceName` | リクエストの送信元サービス種別（下表参照） |
| `httpSourceId` | 関連付けられたリソースの ID（ARN 形式） |
| `cfDistributionTenantId` | CloudFront ディストリビューションテナントの識別子（オプション） |
### `httpSourceName` の値一覧

| 値                 | 対応サービス                    |
| ----------------- | ------------------------- |
| `CF`              | Amazon CloudFront         |
| `APIGW`           | Amazon API Gateway        |
| `ALB`             | Application Load Balancer |
| `APPSYNC`         | AWS AppSync               |
| `COGNITOIDP`      | Amazon Cognito            |
| `APPRUNNER`       | AWS App Runner            |
| `VERIFIED_ACCESS` | AWS Verified Access       |
### `httpSourceId` の ARN 形式

| サービス | ARN 形式 |
|---|---|
| CloudFront | `arn:partition:cloudfront::account-id:distribution/distribution-id` |
| ALB | `arn:partition:elasticloadbalancing:region:account-id:loadbalancer/app/name/id` |
| API Gateway | `arn:partition:apigateway:region::/restapis/api-id/stages/stage-name` |
| AppSync | `arn:partition:appsync:region:account-id:apis/GraphQLApiId` |
| Cognito | `arn:partition:cognito-idp:region:account-id:userpool/user-pool-id` |
| App Runner | `arn:partition:apprunner:region:account-id:service/name/id` |

---
## 4. WAF アクション情報

| フィールド                    | 説明                                                                      |
| ------------------------ | ----------------------------------------------------------------------- |
| `action`                 | WAF がリクエストに適用した**終了アクション**（`ALLOW` / `BLOCK` / `CAPTCHA` / `CHALLENGE`） |
| `responseCodeSent`       | カスタムレスポンスで送信されたレスポンスコード                                                 |
| `requestHeadersInserted` | カスタムリクエスト処理のために挿入されたヘッダーのリスト                                            |
|                          |                                                                         |

---
## 5. ルール評価情報

### 5-1. 終了ルール

| フィールド | 説明 |
|---|---|
| `terminatingRuleId` | リクエストを終了したルールの ID。終了したものがない場合は `Default_Action` |
| `terminatingRuleType` | 終了ルールの種別（`RATE_BASED` / `REGULAR` / `GROUP` / `MANAGED_RULE_GROUP`） |
| `terminatingRuleMatchDetails` | 終了ルールの一致詳細。**SQLi・XSS ルールのみ**設定される |

#### `terminatingRule` オブジェクトの構造

| サブフィールド | 説明 |
|---|---|
| `action` | 適用された終了アクション |
| `ruleId` | 一致したルールの ID |
| `ruleMatchDetails` | 一致詳細（SQLi・XSS のみ）。複数の検査基準を配列で提供 |

### 5-2. 非終了ルール

`nonTerminatingMatchingRules` — リクエストに一致した非終了ルールのリスト

| サブフィールド | 説明 |
|---|---|
| `action` | 適用されたアクション（`COUNT` / `CAPTCHA` / `CHALLENGE`） |
| `ruleId` | 非終了ルールの ID |
| `ruleMatchDetails` | 一致詳細（SQLi・XSS のみ）。`overriddenAction` も含まれる場合がある |

### 5-3. ルールグループ

| フィールド | 説明 |
|---|---|
| `ruleGroupList` | リクエストに対して動作したルールグループのリスト（一致情報含む） |
| `ruleGroupId` | ルールグループの ID。ブロックされた場合は `terminatingRuleId` と同じ |

### 5-4. 除外ルール

`excludedRules` — ルールグループで除外されたルールのリスト（アクションが Count に設定）

| サブフィールド | 説明 |
|---|---|
| `exclusionType` | 除外されたルールのアクションが Count であることを示すタイプ |
| `ruleId` | 除外されたルールの ID |

> **注意**: オーバーライドルールアクションでカウントに設定したルールは `excludedRules` には現れず、`action` / `overriddenAction` ペアとして記録される。

### 5-5. ラベル

| フィールド | 説明 |
|---|---|
| `labels` | ルールによって適用されたウェブリクエストのラベル（最初の 100 件） |

---

## 6. レートベースルール情報

`rateBasedRuleList` — リクエストに動作したレートベースルールのリスト

| サブフィールド | 説明 |
|---|---|
| `rateBasedRuleId` | レートベースルールの ID。終了ルールの場合は `terminatingRuleId` と一致 |
| `rateBasedRuleName` | レートベースルールの名前 |
| `limitKey` | 集約タイプ（下表参照） |
| `limitValue` | 単一 IP タイプのレート制限時のみ使用。無効 IP の場合は `INVALID` |
| `maxRateAllowed` | 指定時間枠内に許可されるリクエストの最大数 |
| `evaluationWindowSec` | WAF がカウントに含めた時間（秒） |
| `customValues` | ルールが識別した一意の値（文字列値は先頭 32 文字まで） |

### `limitKey` の値一覧

| 値 | 意味 |
|---|---|
| `IP` | ウェブリクエストの発信元 IP |
| `FORWARDED_IP` | リクエストヘッダーで転送された IP |
| `CUSTOMKEYS` | カスタム集約キー設定 |
| `CONSTANT` | 集約なし（すべてのリクエストをまとめてカウント） |

---

## 7. CAPTCHA / Challenge レスポンス

どちらも構造は共通。終了アクションか非終了アクションかで含まれる情報が異なる。

| フィールド | 説明 |
|---|---|
| `captchaResponse` | CAPTCHA アクションが適用されたときのステータス（最後に適用された時点の情報） |
| `challengeResponse` | Challenge アクションが適用されたときのステータス（最後に適用された時点の情報） |

| 状態 | 含まれる情報 |
|---|---|
| 終了アクション（トークンなし・無効・期限切れ） | レスポンスコード + 失敗理由（`failureReason`） |
| 非終了アクション（有効なトークンあり） | 解決タイムスタンプ |

> **TIP**: `failureReason` が空でないエントリをフィルタリングすることで、終了アクションと非終了アクションを区別できる。

---
## 8. その他

| フィールド            | 説明                              |
| ---------------- | ------------------------------- |
| `oversizeFields` | WAF が検査した結果、検査制限を超えていたフィールドのリスト |
### `oversizeFields` に含まれる可能性のある値

| 値 | 説明 |
|---|---|
| `REQUEST_BODY` | リクエストボディがオーバーサイズ |
| `REQUEST_JSON_BODY` | JSON ボディがオーバーサイズ |
| `REQUEST_HEADERS` | リクエストヘッダーがオーバーサイズ |
| `REQUEST_COOKIES` | クッキーがオーバーサイズ |

> **注意**: WAF が検査していないフィールドは、オーバーサイズでもここには記録されない。

---

## 付録：ログ JSON の構造イメージ

```json
{
  "timestamp": 1741734000000,
  "formatVersion": 1,
  "webaclId": "arn:aws:wafv2:...",
  "action": "BLOCK",
  "httpSourceName": "ALB",
  "httpSourceId": "arn:aws:elasticloadbalancing:...",
  "terminatingRuleId": "MyBlockRule",
  "terminatingRuleType": "REGULAR",
  "terminatingRule": {
    "action": "BLOCK",
    "ruleId": "MyBlockRule",
    "ruleMatchDetails": []
  },
  "httpRequest": {
    "clientIp": "192.0.2.1",
    "country": "JP",
    "uri": "/api/login",
    "args": "foo=bar",
    "httpMethod": "POST",
    "httpVersion": "HTTP/1.1",
    "headers": [...]
  },
  "ruleGroupList": [...],
  "rateBasedRuleList": [],
  "nonTerminatingMatchingRules": [],
  "labels": []
}
```



---

### ログフィールド
https://docs.aws.amazon.com/ja_jp/waf/latest/developerguide/logging-fields.html

### S3へ保存する方法
https://docs.aws.amazon.com/ja_jp/waf/latest/developerguide/logging-s3.html

AWS WAFのログをAmazon S3に直接保存（ロギング）することで、マネジメントコンソール上の1週間という制限を超えた長期保存や、Athenaを使ったデータ分析が低コストで可能になります。

設定手順の概要

AWS WAFからS3へのログ出力を有効化するためのステップです。

1. **専用S3バケットの準備**
    - 名前が `aws-waf-logs-` から始まるS3バケットを新規作成します。]
2. **S3バケットポリシーの付与**
    - WAFがログを書き込めるように、バケットのアクセス権限（ポリシー）で以下のサービスプリンシパル `://amazonaws.com` を許可します。]
3. **WAF側でロギングを有効化**
    - AWS WAFコンソールの「Logging」タブから「Enable」を選択し、送信先として作成したS3バケットを指定します。]

ログの例
https://docs.aws.amazon.com/ja_jp/waf/latest/developerguide/logging-examples.html