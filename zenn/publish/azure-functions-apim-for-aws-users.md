---
title: "AWS Lambda/API Gateway ユーザーのための Azure Functions & API Management 入門"
emoji: "⚡"
type: "tech"
topics: ["azure", "azurefunctions", "apimanagement", "serverless", "aws"]
published: false
---

## はじめに

AWS Lambda や API Gateway を使ってサーバーレスアーキテクチャを組んだことがある人が Azure に移ってきたとき、「似たようなサービスがあるはずだけど何が違うの？」と感じることは多いです。

この記事では **Azure Functions** と **Azure API Management（APIM）** を中心に、AWS との対応関係と Azure 固有の設計思想を整理します。AWS 側の詳細な説明は最小限に留め、Azure 側の理解に集中します。

## まず全体像を掴む：サービス対応表

| AWS | Azure | 備考 |
|---|---|---|
| Lambda | **Azure Functions** | メインの対応サービス |
| API Gateway | **Azure API Management** | 機能範囲が大きく異なる（後述） |
| Step Functions | **Durable Functions** | Azure Functions の拡張機能 |
| S3 / DynamoDB / SQS（状態保持用途） | **Azure Storage**（Blob / Table / Queue） | Functions のバインディングで宣言的に接続 |
| Lambda レイヤー | **Function App の共有設定** | Function App 単位で共有 |

Azure の公式ドキュメント「[Azure for AWS professionals](https://learn.microsoft.com/en-us/azure/architecture/aws-professional/compute)」では、Lambda の主な対応先として Azure Functions を挙げています。ただし「同じもの」ではなく、設計上の差異がいくつかあります。

## Azure Functions の基本概念

### Function App という概念：Lambda との最大の違い

Lambda では、1 つの関数が 1 つの独立したエンティティです。関数ごとにデプロイ・スケール・設定を管理します。

Azure Functions では、複数の関数を **Function App** という単位にまとめます。Function App は複数の関数を収容するコンテナであり、以下が共有されます。

- デプロイ単位（Function App ごとにまとめてデプロイ）
- 実行コンテキスト（アプリ設定・接続文字列）
- スケール設定（プランに応じて Function App 単位または関数単位でスケール）

公式の比較表より：

> AWS Lambda では各 Lambda 関数が独立したエンティティだが、Azure Functions の Function App は複数の関数のコンテナとなり、共有の実行コンテキストと設定を提供する。複数の関数を 1 つのエンティティとして扱うことで、デプロイと管理が簡略化される。

実務上は、関連するビジネス機能をひとつの Function App にまとめるのが一般的です。

### トリガーとバインディング：宣言的な接続

Lambda でも EventBridge・SQS・S3 イベントなどをトリガーに設定できますが、Azure Functions はそれをさらに統一的な概念「**バインディング**」として扱います。

- **トリガー**：関数の起動条件（必ず1つ）。HTTP リクエスト、タイマー、キューメッセージ、Blob のアップロードなど
- **入力バインディング**：トリガー発火時に関数へ渡すデータ（読み取り）
- **出力バインディング**：関数の実行結果を書き出す先（書き込み）

バインディングは `function.json`（v3 以前）またはコード上のアノテーション（v4）で宣言するだけで、SDK を直接呼ばずにサービスへ接続できます。

```json
{
  "bindings": [
    { "type": "httpTrigger", "direction": "in", "authLevel": "function", "name": "input" },
    { "type": "http",        "direction": "out", "name": "$return" },
    {
      "type": "queue", "direction": "out",
      "name": "myQueueItem",
      "queueName": "outqueue",
      "connection": "MyStorageConnectionAppSetting"
    }
  ]
}
```

この例では HTTP リクエストをトリガーとして受け取り、レスポンスを返しながら Storage Queue にメッセージを出力します。SDK を直接使わず宣言だけで接続できる点は、Lambda + SQS を手動で実装するより簡潔です。

### ホスティングプランの選び方

Lambda には実行モデルが実質1種類（Provisioned Concurrency をオプションで追加）ですが、Azure Functions は **ホスティングプラン** によって課金モデル・スケール動作・機能が大きく変わります。

| プラン | Lambda との比較 | 特徴 | 推奨ケース |
|---|---|---|---|
| **Flex Consumption**（新・推奨） | Lambda 標準に最も近い | ペイパーユース、VNet 対応、メモリサイズ選択可能、高速スケールアウト | 新規サーバーレス開発全般 |
| **Premium**（Elastic Premium） | Provisioned Concurrency に近い | 常時ウォームインスタンス、VNet 対応、長時間タイムアウト、Linux コンテナ対応 | コールドスタートを排除したい場合 |
| **Dedicated**（App Service） | EC2 常時稼働に近い | 固定コスト、既存 App Service プランの流用 | 常時起動が必要 / コスト予測重視 |
| **Consumption**（旧型） | — | Linux は廃止済み、新規非推奨 | **新規開発では使わない** |

:::message
公式ドキュメントでは「新しいサーバーレス Function App には Flex Consumption プランを使うこと」と明記されており、旧 Consumption プランは legacy 扱いです。
:::

### コールドスタートと課金の違い

AWS Lambda は 2024 年以降、初期化フェーズ（Init）も課金対象になりました。

Azure Functions の Flex Consumption / Consumption プランでは、**初期化フェーズは課金対象外**です。課金されるのは実際の実行時間（`Executed` ログに記録された時間）のみです。ただしコールドスタートによる遅延自体は発生します。遅延が許容できないケースは Premium プランで解決します。

### Durable Functions：Step Functions の代替

複数の Lambda を Step Functions で繋ぐようなワークフローを実装したい場合、Azure の対応は **Durable Functions** です。これは Azure Functions の拡張機能として追加インストールして使います。

- ファンアウト/ファンイン（並列実行の集約）
- 長時間実行ワークフロー（数日単位も可）
- 人間の承認待ちフロー
- ステートフルなエンティティ管理

Lambda のコードと同じ言語・同じコードベースで書けるため、Step Functions のように別サービスに切り出す必要がありません。

## Azure API Management（APIM）の基本概念

ここが AWS ユーザーにとって最も「別物」として意識すべき部分です。AWS API Gateway がシンプルな HTTP プロキシ + ルーティングに近いのに対し、**APIM はエンタープライズ向けの API プラットフォーム**です。

### 3 つのコンポーネント

APIM は以下の 3 つのコンポーネントで構成されます。

```
┌──────────────────────────────────────────────────────┐
│              Azure API Management                    │
│                                                      │
│  ┌─────────────┐  ┌────────────────┐  ┌──────────┐  │
│  │  API gateway │  │ Management     │  │Developer │  │
│  │ (data plane) │  │ plane          │  │portal    │  │
│  └──────┬──────┘  └────────────────┘  └──────────┘  │
│         │                                            │
└─────────┼────────────────────────────────────────────┘
          ▼
    Backend APIs（Azure Functions 等）
```

| コンポーネント | 役割 | AWS での近いもの |
|---|---|---|
| **API ゲートウェイ** | リクエストのプロキシ・ポリシー適用・テレメトリ収集 | API Gateway |
| **管理プレーン** | API の定義・ポリシー設定・分析ダッシュボード | API Gateway コンソール |
| **開発者ポータル** | API ドキュメント・テスト・サブスクリプション管理の UI | **なし（AWS には相当機能がない）** |

開発者ポータルは APIM に内蔵されており、API コンシューマー向けのドキュメントサイトとテスト環境を自動生成します。AWS API Gateway にはこれに相当する機能がなく、別途 Swagger UI などを自前で用意する必要があります。

### ポリシーエンジン：APIM 最大の強み

APIM の最も特徴的な機能が **ポリシーエンジン**です。AWS API Gateway のマッピングテンプレートや Lambda Authorizer に相当しますが、はるかに豊富です。

ポリシーは XML で記述し、4 つのフェーズに適用されます。

```xml
<policies>
  <inbound>
    <!-- クライアント → ゲートウェイのリクエストに適用 -->
    <rate-limit calls="100" renewal-period="60" />
    <validate-jwt header-name="Authorization" ... />
  </inbound>
  <backend>
    <!-- ゲートウェイ → バックエンドのリクエストに適用 -->
    <set-backend-service ... />
  </backend>
  <outbound>
    <!-- バックエンド → クライアントのレスポンスに適用 -->
    <json-to-xml apply="always" />
    <cache-store duration="3600" />
  </outbound>
  <on-error>
    <!-- エラー発生時の処理 -->
    <return-response>...</return-response>
  </on-error>
</policies>
```

**主要なビルトインポリシー（75 種類以上）：**

| カテゴリ | ポリシー | 内容 |
|---|---|---|
| 認証 | `validate-jwt` | JWT トークンの検証 |
| レート制限 | `rate-limit` / `rate-limit-by-key` | 呼び出し回数制限 |
| 変換 | `json-to-xml` / `xml-to-json` | フォーマット変換 |
| キャッシュ | `cache-lookup` / `cache-store` | レスポンスキャッシュ |
| セキュリティ | `ip-filter` | IP アドレスフィルタ |
| バックエンド | `circuit-breaker`（backend 設定） | サーキットブレーカー |

:::message alert
APIM のレート制限は分散アーキテクチャの性質上、設定値と実際の制限数に誤差が生じる場合があります（公式ドキュメントでも明記）。厳密なレート制限が求められるケースは追加の考慮が必要です。
:::

### ティアの選び方

APIM のティアは歴史的経緯で種類が多いですが、**新規開発では v2 ティアを選ぶ**のが基本です。

| ティア | SLA | VNet 統合 | 推奨用途 |
|---|---|---|---|
| **Consumption** | なし | なし | プロトタイプ・小規模 |
| **Basic v2** | あり | なし | 小〜中規模本番 |
| **Standard v2** | あり | あり（単一 VNet） | 中規模本番 |
| **Premium v2** | あり | あり + マルチリージョン | エンタープライズ |
| Developer（classic） | なし | あり | **本番非推奨**（開発・テスト専用） |

v2 ティアは「より信頼性が高いプラットフォームに構築されており、幅広い顧客が利用できるよう設計されている」と公式ドキュメントに記載されています。

### ゲートウェイの種類

| ゲートウェイ | 概要 |
|---|---|
| **マネージドゲートウェイ** | Azure 上でフルマネージド（デフォルト） |
| **セルフホストゲートウェイ** | オンプレミスや他クラウドにコンテナとして配置可能 |
| **ワークスペースゲートウェイ** | 開発チームごとに分離した API 管理（Workspace 機能） |

セルフホストゲートウェイはハイブリッドクラウド構成で特に有用で、AWS API Gateway にはない機能です。

## Azure Functions と APIM を組み合わせる

Lambda + API Gateway の組み合わせに対応する Azure の典型構成は、**Azure Functions を APIM のバックエンドとして登録する**パターンです。

```
クライアント
    ↓
Azure API Management
  ├── ポリシー適用（認証・レート制限・変換）
  ├── 開発者ポータルで API ドキュメント公開
  └── バックエンドルーティング
         ↓
    Azure Functions（Function App）
         ↓
    Azure Storage / Cosmos DB 等
```

この構成のメリット：
- Functions の URL を直接公開せず、APIM を通して一元管理できる
- 認証・レート制限・ログを APIM 側で担うことで Functions はビジネスロジックに集中できる
- 開発者ポータルで API ドキュメントとテスト環境を自動生成できる

## まとめ：AWS からの移行で意識したいポイント

| 項目 | AWS | Azure | 注意点 |
|---|---|---|---|
| 関数のまとめ方 | 各関数が独立 | Function App に束ねる | デプロイ・設定の粒度が変わる |
| ホスティングプラン | Provisioned Concurrency でウォーム化 | Premium プランで常時ウォーム | Flex Consumption が新規の推奨 |
| 外部サービス接続 | SDK で手動 | バインディングで宣言的 | コードが簡潔になる |
| API プロキシ | API Gateway（シンプル） | APIM（ポリシーエンジン付き） | XML ポリシーの学習コストあり |
| API ドキュメント | 別途用意 | 開発者ポータルが内蔵 | すぐ使えるが UI カスタマイズが必要なことも |
| セルフホストゲートウェイ | なし | あり（APIM） | ハイブリッド構成で差が出る |

AWS のサービスとの1対1の対応に慣れてしまっていると、Function App という「コンテナ」概念や APIM のポリシーエンジンに最初戸惑うかもしれません。しかし、その仕組みを理解すると「複数の関数を1つのアプリとして管理する」「API の横断的な関心事をポリシーで一元化する」という Azure の設計思想が見えてきます。

---

## 参考リンク

- [AWS Lambda to Azure Functions migration guide](https://learn.microsoft.com/en-us/azure/azure-functions/migration/migrate-aws-lambda-to-azure-functions)
- [Azure for AWS professionals - Compute](https://learn.microsoft.com/en-us/azure/architecture/aws-professional/compute)
- [Azure Functions triggers and bindings](https://learn.microsoft.com/en-us/azure/azure-functions/functions-triggers-bindings)
- [Azure Functions hosting options](https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale)
- [Flex Consumption plan](https://learn.microsoft.com/en-us/azure/azure-functions/flex-consumption-plan)
- [Azure API Management overview](https://learn.microsoft.com/en-us/azure/api-management/api-management-key-concepts)
- [API Management gateways](https://learn.microsoft.com/en-us/azure/api-management/api-management-gateways-overview)
- [API Management v2 tiers](https://learn.microsoft.com/en-us/azure/api-management/v2-service-tiers-overview)
- [How to set or edit policies](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-policies)
