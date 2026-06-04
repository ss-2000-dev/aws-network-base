> 最終更新: 2026年6月  
> 参考: AWS 公式ドキュメント / AWS What's New

---
## TL;DR

|            | AWS WAF                              | AWS Network Firewall    |
| ---------- | ------------------------------------ | ----------------------- |
| **一言で言うと** | Webアプリへの攻撃を防ぐ L7 ファイアウォール            | VPC全体を守るネットワーク層 IDS/IPS |
| **守る対象**   | HTTP/HTTPS リクエスト                     | すべての TCP/UDP トラフィック     |
| **配置場所**   | CloudFront / ALB / API Gateway などの前段 | VPC 内のサブネット間            |
| **最小コスト**  | 〜$6/月〜                               | 〜$285/月〜（per AZ）        |

---
## 1. 基本仕様

| 項目 | AWS WAF | AWS Network Firewall |
|---|---|---|
| **OSI レイヤー** | Layer 7（アプリケーション層） | Layer 3〜7 |
| **対応プロトコル** | HTTP / HTTPS のみ | TCP / UDP / ICMP / HTTP / HTTPS など全般 |
| **ステートフル検査** | N/A（リクエスト単位の評価） | ✅ ステートフル検査に対応 |
| **ステートレス検査** | ✅（ルール評価） | ✅ ステートレス検査にも対応 |
| **IDS/IPS 機能** | ❌ | ✅ Suricata エンジンによる IDS/IPS |
| **マネージドサービス** | ✅ | ✅ |

- ネットワークファイアウォールのドメインリスト（ドメインフィルタリング）は、主にアウトバウンド（外向き）通信の制御に使用される


---
## 2. 配置・統合

### AWS WAF の統合先

| 統合先 | 備考 |
|---|---|
| Amazon CloudFront | エッジで検査。グローバルに効果 |
| Application Load Balancer (ALB) | リージョン単位で保護 |
| Amazon API Gateway | REST API / HTTP API |
| AWS AppSync | GraphQL API |
| Amazon Cognito | ユーザープール |
| AWS App Runner | コンテナアプリ |
| AWS Verified Access | ゼロトラストアクセス |
### AWS Network Firewall の統合先

| 統合先 | 備考 |
|---|---|
| VPC サブネット | ファイアウォールサブネットを専用作成して配置 |
| AWS Transit Gateway | ハブ&スポーク構成のハブ VPC に配置 |
| AWS Firewall Manager | マルチアカウント一元管理 |
| Amazon VPC（複数） | 1つのファイアウォールで最大 50 VPC を保護（per AZ） |

---

## 3. 機能比較

### 3-1. フィルタリング機能

| 機能 | AWS WAF | AWS Network Firewall |
|---|---|---|
| IP アドレスフィルタリング | ✅ IP セット | ✅ ステートレス/ステートフルルール |
| 地理情報（Geo）フィルタリング | ✅ 国単位 | ✅ GeoIP フィルタリング（Suricata） |
| ポート・プロトコルフィルタリング | ❌ | ✅ |
| ドメイン名フィルタリング（FQDN） | ❌ | ✅ |
| URL カテゴリフィルタリング | ❌ | ✅（TLS Inspection 必要） |
| HTTP ヘッダー検査 | ✅ | ✅（TLS Inspection で復号後） |
| リクエストボディ検査 | ✅ | ⚠️ Suricata で可能だが WAF より限定的 |
| SQL インジェクション検出 | ✅ | ⚠️ Suricata ルールで記述が必要 |
| XSS 検出 | ✅ | ⚠️ Suricata ルールで記述が必要 |
| レートリミット（Rate Limiting） | ✅ | ❌ |
| Bot 制御 | ✅（Bot Control） | ❌ |
| CAPTCHA / チャレンジ | ✅ | ❌ |

### 3-2. ルールエンジン

| 項目 | AWS WAF | AWS Network Firewall |
|---|---|---|
| ルール記述方式 | GUI / JSON（WAF ルール言語） | GUI / Suricata 互換ルール文字列 |
| カスタムルール | ✅ | ✅ |
| マネージドルール（AWS 提供） | ✅ Core Rule Set / Known Bad IPs など | ✅ Active Threat Defense など |
| サードパーティマネージドルール | ✅ AWS Marketplace | ❌（限定的） |
| ルールグループ | ✅ | ✅ |
| ルール優先度 | ✅ | ✅ |
| 正規表現マッチ | ✅ | ✅（Suricata の PCRE） |

### 3-3. TLS/暗号化トラフィックの扱い

| 項目 | AWS WAF | AWS Network Firewall |
|---|---|---|
| HTTPS 検査 | ✅（統合先で復号済みの平文を受け取る） | ✅ TLS Inspection（Advanced Inspection）で復号 |
| TLS インスペクション（復号） | N/A（CloudFront / ALB 側が処理） | ✅ オプション機能（Advanced Inspection） |
| SNI によるドメイン識別 | N/A | ✅ 復号なしでも SNI で判定可能 |
| ECH（Encrypted Client Hello）対応 | N/A | ⚠️ ECH 使用時は SNI 読み取り不可→接続リセット |

### 3-4. ログ・モニタリング

| 項目 | AWS WAF | AWS Network Firewall |
|---|---|---|
| ログ出力先 | S3 / CloudWatch Logs / Kinesis Data Firehose | S3 / CloudWatch Logs / Kinesis Data Firehose |
| フローログ | ❌（リクエスト単位のログ） | ✅ フローログ |
| アラートログ | ✅ | ✅ |
| CloudWatch メトリクス | ✅ | ✅ |
| モニタリングダッシュボード | ✅ | ✅（2025年9月更新で強化） |
| AWS Security Hub 連携 | ✅ | ✅ |

---

## 4. 料金

> ⚠️ 以下は **us-east-1（バージニア北部）** の参考価格です。最新情報は公式をご確認ください。

### AWS WAF

| 課金項目 | 単価 |
|---|---|
| Web ACL | $5.00 / 月 |
| ルール | $1.00 / ルール / 月 |
| リクエスト検査 | $0.60 / 100万リクエスト |
| Bot Control（Common） | $1.00 / 100万リクエスト |
| Bot Control（Targeted） | $10.00 / 100万リクエスト |
| マネージドルールグループ | $1.00 / ルールグループ / 月 + 使用料 |

**概算例（1 Web ACL + 10ルール + 月5000万req）**

```
$5（Web ACL）+ $10（ルール）+ $30（リクエスト）= $45 /月
```

### AWS Network Firewall

| 課金項目 | 単価 |
|---|---|
| ファイアウォールエンドポイント（プライマリ） | $0.395 / 時間 / AZ |
| ファイアウォールエンドポイント（セカンダリ） | $0.158 / 時間 / AZ |
| データ処理 | $0.065 / GB |
| Advanced Inspection（TLS検査） | 追加時間単価あり（データ処理追加料金は廃止） |
| Advanced Threat Protection | 追加データ処理料金あり |

**概算例（2AZ × プライマリエンドポイント + 5TB/月）**

```
$568.80（エンドポイント）+ $325（データ処理）= $893.80 /月
※ NAT Gateway と組み合わせ時は割引あり
```

### コスト比較まとめ

| | AWS WAF | AWS Network Firewall |
|---|---|---|
| **最低コスト** | 〜$6/月〜 | 〜$285/月〜（1AZ） |
| **3AZ 構成** | Web ACL 数に依存 | 〜$855/月〜（エンドポイントのみ） |
| **スケール方式** | リクエスト数に応じて従量課金 | エンドポイント数 × AZ 数が支配的 |
| **コスト予測のしやすさ** | ✅ しやすい | ⚠️ トラフィック量により変動 |

---

## 5. ユースケース別選択ガイド

| シナリオ | 推奨サービス | 理由 |
|---|---|---|
| SQLi / XSS から Web アプリを守りたい | ✅ **WAF** | Layer 7 検査でHTTPの意味を理解した判定が可能 |
| ボットや不正スクレイピングを防ぎたい | ✅ **WAF**（Bot Control） | ボット識別・CAPTCHA が統合済み |
| API Gateway / CloudFront のリクエストを制御したい | ✅ **WAF** | 直接統合できる |
| レートリミット（DDoS緩和）をかけたい | ✅ **WAF** | レートベースルールが利用可能 |
| VPC 内の East-West トラフィックを検査したい | ✅ **Network Firewall** | VPC 内部の横断トラフィックに対応 |
| アウトバウンドの通信先ドメインを制限したい | ✅ **Network Firewall** | FQDN / ドメインカテゴリフィルタリング |
| マルウェアシグネチャによる脅威検出（IDS/IPS）が必要 | ✅ **Network Firewall** | Suricata エンジンで対応 |
| Transit Gateway ハブで全 VPC のトラフィックを集中検査 | ✅ **Network Firewall** | ハブ&スポーク構成に最適 |
| 暗号化されたアウトバウンド通信の中身を検査したい | ✅ **Network Firewall**（TLS Inspection） | 復号→検査→再暗号化が可能 |
| 特定ポート・プロトコルをブロックしたい | ✅ **Network Firewall** | 5タプル（IP/Port/Protocol）ルールに対応 |
| コストを抑えて最低限の保護をしたい | ✅ **WAF** | 低コストから始められる |

---

## 6. アーキテクチャパターン

### パターン A：WAF のみ（小規模 Web アプリ）

```
Internet
   │
CloudFront または ALB
   │
[AWS WAF]  ← SQLi/XSS/Bot をここで遮断
   │
EC2 / Lambda / ECS
```

### パターン B：Network Firewall のみ（VPC 内部制御）

```
Internet
   │
Internet Gateway
   │
[AWS Network Firewall]  ← IDS/IPS・ドメインフィルタ
   │
サブネット（EC2 / RDS など）
```

### パターン C：両方を組み合わせた多層防御（推奨）

```
Internet
   │
CloudFront / ALB
   │
[AWS WAF]  ← Layer 7: SQLi・XSS・Bot・レートリミット
   │
[AWS Network Firewall]  ← Layer 3〜7: IDS/IPS・ドメイン制限・East-West
   │
プライベートサブネット（EC2 / RDS など）
```

> **ポイント**: WAF と Network Firewall は競合ではなく補完関係。  
> WAF はアプリ層の攻撃を止め、Network Firewall はネットワーク層の脅威を止める。

---

## 7. 管理・運用

| 項目 | AWS WAF | AWS Network Firewall |
|---|---|---|
| **設定の難易度** | ⭐⭐（中程度） | ⭐⭐⭐（高め・Suricata の知識が必要） |
| **マルチアカウント管理** | AWS Firewall Manager で可能 | AWS Firewall Manager で可能 |
| **Infrastructure as Code** | ✅ CloudFormation / Terraform | ✅ CloudFormation / Terraform |
| **ルールの自動更新** | ✅ マネージドルールグループ | ✅ Active Threat Defense マネージドルール |
| **テスト用モード** | ✅ Count モード | ✅ Alert（アラートのみ、ブロックなし） |
| **デプロイの複雑さ** | 低（統合先に関連付けるだけ） | 高（サブネット・ルートテーブルの変更が必要） |

---

## 8. まとめ：どちらを選ぶか

```
Web アプリ（HTTP/HTTPS）への攻撃対策が主目的？
  → YES → AWS WAF

VPC レベルのネットワーク制御・IDS/IPS が必要？
  → YES → AWS Network Firewall

両方必要（公開 Web アプリ + 内部ネットワーク保護）？
  → YES → 両方を組み合わせる（多層防御）
```

### 選定チェックリスト

| チェック項目 | WAF | Network Firewall |
|---|---|---|
| SQLi / XSS / Bot 対策が必要 | ✅ | |
| ポート・プロトコルレベルの制御が必要 | | ✅ |
| アウトバウンドのドメイン制限が必要 | | ✅ |
| VPC 内の East-West トラフィック検査が必要 | | ✅ |
| コストを月$100以内に抑えたい | ✅ | |
| Suricata ルールを書ける/書けるチームがいる | | ✅ |
| CloudFront / ALB / API Gateway と統合したい | ✅ | |
| Transit Gateway のハブ VPC で集中検査したい | | ✅ |
| レートリミット・DDoS 緩和をしたい | ✅ | |
| 暗号化通信（TLS）の中身を検査したい | | ✅ |
