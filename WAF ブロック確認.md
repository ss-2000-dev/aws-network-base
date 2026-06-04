https://dev.classmethod.jp/articles/tsnote-confirm-blocked-request-by-aws-waf/

WAF（主にAWS WAF）でブロックされたリクエストの詳細や、誤検知の原因を確認する手順です。ブロックされたログから「どのルールにマッチしたか」や「対象のURL・IP」を特定できます。 [[1](https://aws.taf-jp.com/blog/111208)]

1. マネジメントコンソールでの簡単な確認（サンプリングログ）

リアルタイムでサンプリングされたブロックログを素早く確認する方法です。

1. **AWS WAF & Shield コンソール** にログインします。
2. 対象の **Web ACL** を選択し、**[Overview]** タブを開きます。
3. **[Sampled requests]** セクションで、リクエストがブロックされた（Blocked）リクエストを選択します。
4. 詳細を展開すると、どのルール（Rule）にブロックされたか、また該当するURIなどを確認できます。

5. 詳細ログの検索（CloudWatch Logs Insights）

長期間のログや特定のIP・パスを詳細に検索するには、事前にログ出力を有効化する必要があります。

1. Web ACLの **[Logging and metrics]** タブから、ログの出力先（Amazon Kinesis Data Firehose または Amazon CloudWatch Logs）を設定します。
2. 出力先に CloudWatch Logs を指定した場合は、**[CloudWatch] > [Logs Insights]** に移動します。
3. 対象のロググループを選択し、以下のようなクエリを実行してブロックされたリクエストを抽出します。

```log
fields @timestamp, @message

| filter terminatingAction = 'BLOCK'
| sort @timestamp desc
| limit 100

```
3. ブロックの原因と対策

ブロックログを確認すると、主に以下の原因がわかります。

- **正規アクセスの誤検知（False Positive）：** お問い合わせフォームや特定の文字を含むURLが、SQLインジェクションなどのマネージドルールに誤って反応しているケース。
    - **対処法：** [Rules] タブから該当のルールを「Count（カウント）」モードに変更するか、特定のパス（URL）を除外する「Scope-down statement」を設定します。
- **悪意あるアクセスのブロック：** 脆弱性を狙うスキャンや攻撃ボットによるアクセス。
    - **対処法：** 特にアクションを起こす必要はありませんが、特定のIPアドレスからの通信が多すぎる場合は、IPレートベースのルールなどで自動遮断します。 [[1](https://equnetshop.com/blog/433/)]


CloudFront にアタッチされた AWS WAF のマネージドルールでブロックされたリクエスト（BLOCK アクションが発生したログ）を確認するには、「確認の即時性」や「保存期間」に応じて主に **3 つの方法**があります。

目的や現在の設定状況に合わせて最適な方法を選択してください。

## 1. 簡易・リアルタイムに確認する：「サンプリングされたリクエスト」

WAF が直近で処理したリクエストの中から**過去3時間・最大5,000件**のサンプルデータをコンソール上で手軽に確認する方法です。追加のロギング設定をしていなくてもデフォルトで閲覧可能です。

### 確認手順

1. AWS WAF コンソールを開き、リージョンで **Global (CloudFront)** を選択します。

2. 対象の Web ACL を選択し、**「Traffic overview」** タブ（または「Sampled requests」）を開きます。

3. 画面下部の「Sampled requests」テーブルに、検知されたリクエストの一覧が表示されます。

4. **Metric name**（ルールグループ名）や **Action** が `BLOCK` になっているものを探します。

5. 行を展開すると、クライアント IP、URI、ヘッダー情報、およびブロックを引き起こした具体的なマネージドルール内のルール名（Rule ID）を確認できます。


> **注意点:** サンプルデータであるため、高トラフィックな環境ではすべてのブロックリクエストが網羅されるわけではありません。また、過去3時間しか遡れないため、過去のトラブルシューティングには不向きです。

## 2. 詳細に分析・検索する：「CloudWatch Logs Insights」（推奨）

Web ACL のフルロギング（Logging configuration）を有効にして、ログの出力先を **CloudWatch Logs**（`aws-waf-logs-` から始まるロググループ）に設定している場合に利用できる、最も確実で強力な方法です。

### 確認手順

1. AWS WAF コンソールの対象 Web ACL 画面から、**「CloudWatch Logs Insights」**（またはログエクスプローラー）を開きます。
    
2. 以下のクエリを実行して、ブロックされたリクエストを抽出します。
    

SQL

```
fields @timestamp, httpRequest.clientIp, httpRequest.uri, terminatingRuleId, action
| filter action = "BLOCK"
| sort @timestamp desc
| limit 100
```

### マネージドルール特有のチェックポイント

マネージドルール（AWSManagedRulesCommonRuleSet など）内のどのルール（子ルール）でブロックされたかを特定したい場合は、`terminatingRuleMatchDetails` を確認します。

SQL

```
fields @timestamp, httpRequest.clientIp, httpRequest.uri, terminatingRuleId, terminatingRuleMatchDetails
| filter action = "BLOCK"
| sort @timestamp desc
```

- **`terminatingRuleId`**: ブロックを決定づけたマネージドルールグループの名前が表示されます。
    
- **`terminatingRuleMatchDetails`**: ルールグループ内の具体的なサブルール名や、リクエストのどのパーツ（引数やヘッダーなど）がシグネチャにヒットしたかの詳細が JSON 形式で出力されます。
    

## 3. CloudFront 側のアクセスログから追跡する

もし WAF のロギング（上記2）を有効化しておらず、サンプルの保管期間（上記1）も過ぎてしまっている場合、CloudFront 自体のアクセスログ（S3 やリアルタイムログ）から間接的に WAF のブロックを特定できる場合があります。

- **ステータスコード**: WAF でブロックされたリクエストは、CloudFront から通常 `403 (Forbidden)` として返されます。
    
- **結果**: CloudFront の標準アクセスログの `x-edge-result-type` フィールドが `Error` になり、WAF が拒否した形跡がステータスコードから推測できます。
    

> **注意点:** ただし、CloudFront のアクセスログ単体では「403 エラーになった」ことまでは分かりますが、**「マネージドルールのどの条件にヒットしてブロックされたか」という詳細な理由は記録されません。** 確実な原因特定には、WAF 側のフルロギング（CloudWatch Logs / S3 / Firehose）の有効化が不可欠です。

### 💡 運用上のTips（誤検知の調査など）

もし特定の正規リクエストがマネージドルールに誤ブロック（False Positive）されていてその原因を調べている場合は、**CloudWatch Logs Insights で `terminatingRuleMatchDetails` を特定し、該当のサブルールのみを「Count（カウント）」アクションにカウントダウン（Override）する**ことで、システムを停止させずに通信を通しながら検証することが可能です。

## CloudFront + WAF でブロックされたリクエストを確認する方法

主に以下の2つの方法があります。

---

### 方法1: AWS WAF ログを有効化して確認（推奨）

#### 1. WAF ログの有効化

1. **AWS コンソール** → **WAF & Shield** を開く
2. 対象の **Web ACL** を選択
3. **「Logging and metrics」タブ** → **「Enable logging」**
4. ログの送信先を選択：
    - **Amazon CloudWatch Logs**（リアルタイム確認に便利）
    - **Amazon S3**（長期保存・大量データ向け）
    - **Amazon Kinesis Data Firehose**（リアルタイム転送向け）

#### 2. CloudWatch Logs でのクエリ例（Logs Insights）

```sql
fields @timestamp, httpRequest.uri, httpRequest.clientIp, terminatingRuleId, action
| filter action = "BLOCK"
| sort @timestamp desc
| limit 100
```

ブロックしたマネージドルールを特定する場合：

```sql
fields @timestamp, httpRequest.clientIp, httpRequest.uri, terminatingRuleId, terminatingRuleMatchDetails
| filter action = "BLOCK"
| filter terminatingRuleId like /AWS-AWSManagedRules/
| sort @timestamp desc
```

---

### 方法2: AWS WAF コンソールのサンプルリクエストで確認

ログを有効にしていなくても、**直近のサンプル**は確認できます。

1. **WAF & Shield** → **Web ACLs** → 対象ACLを選択
2. **「Rules」タブ** → 各マネージドルールグループを展開
3. **「Sampled requests」** ボタンをクリック
4. ブロックされたリクエストのサンプル（最大100件・直近3時間）が表示される

> ⚠️ サンプルリクエストは全件ではなく抜粋です。網羅的な調査にはログが必要です。

---

### 方法3: CloudWatch Metrics で傾向を把握

- **WAF & Shield** → **Web ACLs** → **「Metrics」タブ**
- または CloudWatch コンソールで `AWS/WAFV2` 名前空間を確認
- メトリクス例：`BlockedRequests`（ルールグループ別にフィルタ可能）

---

### 確認できる主な情報

|項目|内容|
|---|---|
|`terminatingRuleId`|ブロックしたルール名（例: `AWS-AWSManagedRulesCommonRuleSet`）|
|`terminatingRuleMatchDetails`|マッチした詳細（SQLi, XSSなど）|
|`httpRequest.clientIp`|送信元IPアドレス|
|`httpRequest.uri`|リクエストされたURI|
|`httpRequest.headers`|リクエストヘッダー|

---

### おすすめの進め方

```
今すぐ確認したい → サンプルリクエスト（方法2）
継続的に監視したい → WAFログ有効化（方法1）+ CloudWatch Logs Insights
```

ログがまだ有効になっていない場合は、まず有効化することを強くお勧めします。有効化後すぐにログが流れ始めます。


WAF のログを CloudWatch Logs に送信する際、ロググループ名には**命名規則の制約**があります。

---

### 命名規則の制約

WAF の CloudWatch Logs のロググループ名は、必ず以下のプレフィックスで始まる必要があります：

```
aws-waf-logs-
```

これは AWS 側の**必須要件**であり、このプレフィックスがないとログの送信先として設定できません。

---

### 「特に設定しない場合」の挙動

WAF ログの有効化時に CloudWatch Logs を選択する場合、**ロググループは事前に自分で作成・指定する必要があります**。AWS が自動でロググループ名を付与する仕組みにはなっていません。

そのため、コンソールで設定する際は：

1. 事前に CloudWatch Logs でロググループを作成する
2. ロググループ名は `aws-waf-logs-` で始める必要がある

**例：**

```
aws-waf-logs-cloudfront-production
aws-waf-logs-my-webacl
```

---

### 補足

Kinesis Data Firehose を経由する場合も、Firehose のストリーム名が `aws-waf-logs-` で始まる必要があります。S3 の場合はこの制約はありません。

ロググループをまだ作成していない場合は、CloudWatch コンソールから先に作成してから WAF の Logging 設定を行う流れになります。