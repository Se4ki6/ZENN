---
title: AWS Lambda/API Gateway ユーザーのための Azure Functions & Azure API Management 入門
status: plan
---

## 記事タイトル候補

1. 「AWS Lambda/API Gateway ユーザーが Azure に来たら最初に知っておくべきこと」
2. 「Lambda → Azure Functions 移行前に押さえたいアーキテクチャの違い」
3. 「AWS 視点で理解する Azure Functions と Azure API Management の全体像」
4. 「Lambda/API Gateway ユーザーのための Azure サーバーレス入門」

## 想定読者

- AWS Lambda / AWS API Gateway を実務で触ったことがある
- Azure は初めてまたは触り始めたばかり
- サーバーレスアーキテクチャの基礎は理解している

## 記事で答える問い

- Azure Functions は Lambda と何が違うのか？ Function App という概念は何か？
- ホスティングプランはどう選べばよいか？
- Azure API Management（APIM）は API Gateway と何が違うのか？
- APIM のポリシーエンジンとはどういうものか？
- Azure Functions と APIM をどう組み合わせるか？

## 扱う範囲・扱わない範囲

### 扱う
- Azure Functions のコアコンセプト（Function App, トリガー, バインディング）
- ホスティングプランの比較（Flex Consumption / Premium / Dedicated）
- Durable Functions の位置付け
- APIM の3大コンポーネント（ゲートウェイ, 管理プレーン, 開発者ポータル）
- APIM のポリシーエンジン概要（inbound / backend / outbound / on-error）
- APIM のティア（v2 系を中心に）
- Azure Functions と APIM の統合方法の概要

### 扱わない
- 具体的なコード実装（Terraform / Bicep / Python 等）
- Azure Functions の言語別詳細
- APIM の全ポリシー詳解
- ネットワーク設計の詳細（VNet 統合の詳細）
- コスト試算

## セクション構成

### 1. はじめに（導入）
**主張:** AWS Lambdaと似ているようで設計思想が異なる部分がある。その差を理解することがスムーズな学習の近道。
**根拠:** learn.microsoft.com/azure/architecture/aws-professional/compute（Azure for AWS professionals）

---

### 2. まず全体像を掴む：サービス対応表
**主張:** AWS ↔ Azure のサービスマッピングを示し、どのサービスがどれに対応するかを一覧で確認する。
**内容:**
- Lambda → Azure Functions（または App Service WebJobs）
- API Gateway → Azure API Management（APIM）
- Step Functions → Durable Functions
- S3/DynamoDB/SQS（状態管理）→ Azure Storage（Blob/Table/Queue）

**根拠:** 
- learn.microsoft.com/en-us/azure/architecture/aws-professional/compute
- learn.microsoft.com/en-us/azure/azure-functions/migration/migrate-aws-lambda-to-azure-functions

---

### 3. Azure Functions の基本概念
**主張:** Lambda との最大の違いは「Function App」というコンテナ概念と「バインディング」による宣言的な接続。

#### 3-1. Function App という概念
**主張:** Lambda は1関数1エンティティだが、Azure Functions は複数の関数を1つの Function App に束ねる。これがデプロイ・設定・スケールの単位になる。
**根拠:** learn.microsoft.com/azure/azure-functions/migration/migrate-aws-lambda-to-azure-functions

#### 3-2. トリガーとバインディング
**主張:** Lambda はイベントソースマッピングで設定するが、Azure Functions はトリガー（起動条件）とバインディング（入出力の宣言的接続）を `function.json` または属性で定義する。
**根拠:** learn.microsoft.com/azure/azure-functions/functions-triggers-bindings

#### 3-3. ホスティングプランの選び方
**主張:** Lambda の課金モデルに近いのは Flex Consumption（新推奨）。Premium は常時暖機（cold start なし）が必要なケース向け。Consumption（旧型）は新規開発では避ける。

| プラン | 特徴 | AWS での近いもの |
|---|---|---|
| Flex Consumption（推奨） | ペイパーユース、VNet対応、メモリ選択可能 | Lambda（標準） |
| Premium（Elastic Premium） | 常時ウォーム、VNet対応、長時間タイムアウト | Lambda Provisioned Concurrency |
| Dedicated（App Service） | 常時起動、固定コスト | EC2 常時稼働 |

**根拠:** 
- learn.microsoft.com/azure/azure-functions/functions-scale
- learn.microsoft.com/azure/azure-functions/flex-consumption-plan
- learn.microsoft.com/azure/azure-functions/functions-premium-plan

#### 3-4. コールドスタートの扱い
**主張:** Azure Functions（Flex/Consumption）は初期化（Init）フェーズが課金対象外（Lambda は最近 Init も課金対象になった）。ただし遅延は発生するため、遅延が問題なら Premium プランで解決する。
**根拠:** learn.microsoft.com/answers/questions/5721106（コミュニティQ&A）

#### 3-5. Durable Functions（ステートフルオーケストレーション）
**主張:** AWS Step Functions に相当する機能が Azure Functions の拡張として提供されている。長時間ワークフロー、ファンアウト/ファンイン、人間承認フローなどを Lambda コードの延長で書ける。
**根拠:** learn.microsoft.com/azure/azure-functions/migration/migrate-aws-lambda-to-azure-functions（Stateful orchestration 節）

---

### 4. Azure API Management（APIM）の基本概念
**主張:** AWS API Gateway より遥かに多機能でエンタープライズ向け。「API の公開・保護・分析・開発者向けポータル」まで一体で提供する。

#### 4-1. 3つのコンポーネント
**主張:** APIM は ゲートウェイ・管理プレーン・開発者ポータル の3要素で構成される。開発者ポータルが内蔵されている点が API Gateway との大きな違い。
**根拠:** learn.microsoft.com/azure/api-management/api-management-key-concepts

#### 4-2. ポリシーエンジン（APIMの最大の強み）
**主張:** APIM のポリシーは XML で記述し、inbound / backend / outbound / on-error の4フェーズでリクエスト・レスポンスを柔軟に変換・制御できる。75以上のビルトインポリシーがある。

代表的なポリシー:
- `rate-limit` / `rate-limit-by-key`: レート制限
- `validate-jwt` / `oauth2-token-limit`: 認証
- `json-to-xml` / `xml-to-json`: フォーマット変換
- `cache-lookup` / `cache-store`: キャッシング
- `ip-filter`: IPフィルタリング
- `circuit-breaker`（backend設定）: バックエンド保護

**根拠:**
- learn.microsoft.com/azure/api-management/api-management-howto-policies
- learn.microsoft.com/azure/api-management/rate-limit-policy
- learn.microsoft.com/azure/api-management/backends

#### 4-3. ティアの選び方
**主張:** 新規開発では v2 ティア（Basic v2, Standard v2, Premium v2）を選ぶ。SLA があり、より信頼性が高いプラットフォーム上に構築されている。Consumption ティアはペイパーユース。

| ティア | 特徴 | 用途 |
|---|---|---|
| Consumption | ペイパーユース、SLA なし | 開発・小規模 |
| Basic v2 | SLA あり、低コスト | 小〜中規模本番 |
| Standard v2 | VNet統合、SLA | 中規模本番 |
| Premium v2 | マルチリージョン、ゾーン冗長 | エンタープライズ |

**根拠:** learn.microsoft.com/azure/api-management/v2-service-tiers-overview

#### 4-4. ゲートウェイの種類
**主張:** マネージドゲートウェイ（Azure 管理）だけでなく、セルフホストゲートウェイ（オンプレ/他クラウド）も使える。ワークスペースゲートウェイは開発チームごとの分離に使う。
**根拠:** learn.microsoft.com/azure/api-management/api-management-gateways-overview

---

### 5. Azure Functions と APIM を組み合わせる
**主張:** APIM のバックエンドに Azure Functions を登録することで、認証・レート制限・ログ・開発者ポータルを一元管理しながら Functions を公開できる。
**補足:** この連携が Lambda + API Gateway の組み合わせに対応する Azure の典型構成。
**根拠:** 上記の各公式ドキュメントから総合的に判断

---

### 6. まとめ：AWS からの移行で意識したいポイント
**主張:** 概念の対応を掴めば移行はスムーズ。一方で Function App 単位の管理、ポリシーエンジン、開発者ポータルは Azure 固有の価値なので積極的に活用する。

---

## 根拠ファイル一覧

| ファイル | 主な用途 |
|---|---|
| `temp/azure-functions-apim-vs-aws/extract_azure_functions.json` | Azure Functions / AWS 比較テーブル |
| `temp/azure-functions-apim-vs-aws/extract_apim.json` | APIM キーコンセプト・ゲートウェイ・ティア |
| `temp/azure-functions-apim-vs-aws/search_extract_functions_plans.json` | ホスティングプラン詳細 |
| `temp/azure-functions-apim-vs-aws/search_extract_apim_policies.json` | ポリシーエンジン詳細 |

## 主要参照 URL

- https://learn.microsoft.com/en-us/azure/azure-functions/migration/migrate-aws-lambda-to-azure-functions
- https://learn.microsoft.com/en-us/azure/architecture/aws-professional/compute
- https://learn.microsoft.com/en-us/azure/azure-functions/functions-triggers-bindings
- https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale
- https://learn.microsoft.com/en-us/azure/azure-functions/flex-consumption-plan
- https://learn.microsoft.com/en-us/azure/azure-functions/functions-premium-plan
- https://learn.microsoft.com/en-us/azure/api-management/api-management-key-concepts
- https://learn.microsoft.com/en-us/azure/api-management/api-management-gateways-overview
- https://learn.microsoft.com/en-us/azure/api-management/v2-service-tiers-overview
- https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-policies
- https://learn.microsoft.com/en-us/azure/api-management/rate-limit-policy
- https://learn.microsoft.com/en-us/azure/api-management/backends
