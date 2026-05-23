---
title: AWS Lambda/API Gateway ユーザーのための Azure Functions & APIM Terraform 移行ガイド（第二弾）
status: plan
---

## 記事タイトル候補

1. 「AWS Lambda/API Gateway を Terraform で書いてた人が Azure に来たら知っておくべき差分」
2. 「Terraform で学ぶ Azure Functions & API Management：AWS との書き方の違いと工夫」
3. 「Lambda → Azure Functions 移行の Terraform ガイド：リソース構造の違いと最小 CRUD 構成」

## 想定読者

- AWS Lambda + API Gateway を Terraform で構築・運用したことがある
- Azure Functions と Azure API Management は第一弾記事でコンセプトは理解した
- Terraform の基本文法（resource / data / variable / output）は理解している

## 記事で答える問い

- Azure Functions に必要な Terraform リソース群は何か？Lambda との構成の違いは？
- `azurerm_linux_function_app` の必須ブロックは何か？
- APIM を Terraform で定義すると何個のリソースが必要になるか？
- ポリシーは Terraform でどう書くか？
- APIM と Functions はどう繋ぐか？
- AWS との典型的なハマりポイントはどこか？

## 扱う範囲・扱わない範囲

### 扱う
- `azurerm_linux_function_app`（Flex Consumption / Premium プラン）
- `azurerm_api_management`・`azurerm_api_management_api`・`azurerm_api_management_api_operation`
- `azurerm_api_management_backend`・`azurerm_api_management_api_operation_policy`
- CRUD（GET list / POST / GET by ID / PUT / DELETE）の最小構成サンプル
- AWS リソースとの対応表
- 典型的なハマりポイント

### 扱わない
- Bicep / ARM テンプレートによる代替手段
- Terraform Cloud / CI/CD パイプライン構成
- Durable Functions の Terraform 定義
- ネットワーク設計（VNet 統合の詳細）
- コスト最適化の詳細

## セクション構成

### 1. はじめに
- 第一弾のコンセプト記事との関係を示す
- この記事でカバーするリソース群の全体像を予告

### 2. AWS vs Azure：Terraform リソース対応表
- `aws_lambda_function` → `azurerm_linux_function_app`（+ `azurerm_service_plan` + `azurerm_storage_account`）
- `aws_api_gateway_rest_api` → `azurerm_api_management`
- `aws_api_gateway_resource` + `aws_api_gateway_method` → `azurerm_api_management_api_operation`
- `aws_api_gateway_integration` → `azurerm_api_management_backend` + set-backend-service ポリシー
- `aws_lambda_permission` → 不要（または managed identity / function key で代替）
- `aws_iam_role` for Lambda → 不要（MSI は optional、Terraform で managed identity 設定可能）

### 3. Azure Functions を Terraform で構築する
**主張:** Lambda と違い、Functions には Storage Account + Service Plan が必須。これが最大の驚きポイント。

サンプル構成:
- `azurerm_resource_group`
- `azurerm_storage_account`
- `azurerm_service_plan`（sku_name = "FC1" で Flex Consumption）
- `azurerm_linux_function_app`（site_config, application_stack, app_settings）

**根拠:**
- registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/linux_function_app
- learn.microsoft.com/azure/azure-functions/functions-create-first-function-terraform

### 4. Azure API Management を Terraform で構築する
**主張:** APIM は1インスタンスを複数のサブリソースで構成する。API Gateway より記述量が多いが表現力が高い。

リソース構成:
- `azurerm_api_management`（インスタンス本体、sku_name = "Developer_1"）
- `azurerm_api_management_api`（API定義）
- `azurerm_api_management_api_operation` × N（HTTP エンドポイント）
- `azurerm_api_management_backend`（Functions への接続先）
- `azurerm_api_management_api_operation_policy`（rate-limit 等のポリシー）

**根拠:**
- registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/api_management
- registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/api_management_api_operation
- registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/api_management_backend
- registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/api_management_api_operation_policy

### 5. APIM と Functions を繋ぐ
- Backend リソースで Functions の URL を設定
- `set-backend-service` ポリシーで inbound に適用
- Function キーの扱い（`azurerm_linux_function_app.example.default_hostname`）

### 6. ハマりポイントまとめ
- `site_config {}` は空でも必須
- Function App 名は 32 文字以内制限
- APIM の `sku_name` は `"Tier_capacity"` 形式（例: `"Developer_1"`）
- Consumption tier は capacity が 0 固定（`"Consumption_0"`）
- `azurerm_api_management` はプロビジョニングが長い（15〜45 分）
- AWS の `lambda_permission` 相当は基本不要（function key 認証）

## 根拠ファイル一覧

| ファイル | 主な用途 |
|---|---|
| `temp/terraform-aws-to-azure/extract_tf_functions.json` | linux_function_app サンプル・必須パラメータ |
| `temp/terraform-aws-to-azure/extract_tf_apim.json` | api_management 必須パラメータ |
| `temp/terraform-aws-to-azure/extract_tf_apim_sub.json` | api, operation, backend |
| `temp/terraform-aws-to-azure/extract_tf_apim_policy.json` | api_operation_policy の xml_content |
| `temp/terraform-aws-to-azure/extract_tf_aws.json` | AWS Lambda / API Gateway 比較 |
