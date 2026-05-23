---
title: "Terraform で学ぶ Azure Functions & API Management：AWS からの移行で変わる書き方"
emoji: "🏗️"
type: "tech"
topics: ["azure", "terraform", "azurefunctions", "apimanagement", "aws"]
published: false
---

## はじめに

[第一弾の記事](https://zenn.dev/)では Azure Functions と Azure API Management（APIM）のコンセプトを AWS との対比で整理しました。この記事では **Terraform での実装** に焦点を当て、AWS 側の書き方と何が変わるかを最小 CRUD 構成のサンプルコードを交えて解説します。

対象読者：

- `aws_lambda_function` や `aws_api_gateway_rest_api` を Terraform で書いたことがある
- Azure Functions と APIM のコンセプトは把握している
- Terraform の基本文法（`resource` / `variable` / `output`）は理解している

## まず：AWS vs Azure リソース対応表

どのリソースがどれに対応するかを先に確認します。

| AWS | Azure（Terraform resource name） | 備考 |
|---|---|---|
| `aws_lambda_function` | `azurerm_linux_function_app` | Azure は Service Plan も別リソース |
| `aws_iam_role`（Lambda 用） | 基本不要 | Function Key 認証が一般的 |
| `aws_lambda_permission` | 基本不要 | APIM との接続に明示的な許可設定は不要 |
| `aws_api_gateway_rest_api` | `azurerm_api_management` | APIM はインスタンス単位 |
| `aws_api_gateway_resource` + `aws_api_gateway_method` | `azurerm_api_management_api_operation` | 1リソースで HTTP メソッド + パスを定義 |
| `aws_api_gateway_integration` | `azurerm_api_management_backend` + ポリシー | Backend + set-backend-service ポリシーの組み合わせ |
| `aws_api_gateway_deployment` | 不要 | APIM は常時有効（デプロイ概念なし） |

AWS では Lambda 単体で数リソース、API Gateway も `resource → method → integration → deployment` と縦に積み上げる構造でした。Azure でも APIM 側のリソース数は同程度以上ですが、**構造の見通しが良い**（API → Operation → Policy という階層）のが特徴です。

---

## Azure Functions を Terraform で構築する

### AWS との最大の違い：Storage Account と Service Plan が必須

Lambda は `aws_lambda_function` + IAM ロールがあれば最低限動きます。Azure Functions は**必ず 3 つのリソースがセットで必要**です。

```
azurerm_resource_group          ← Azure 全般で必須（AWS に相当物なし）
azurerm_storage_account         ← Functions の状態管理・デプロイに必須
azurerm_service_plan            ← ホスティングプランの定義（Lambda にはない）
azurerm_linux_function_app      ← 本体
```

### 最小構成サンプル（Flex Consumption プラン）

```hcl
# ────────────────────────────────────────────────
# provider
# ────────────────────────────────────────────────
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }
}

provider "azurerm" {
  features {}
}

# ────────────────────────────────────────────────
# リソースグループ（AWS に相当物なし）
# ────────────────────────────────────────────────
resource "azurerm_resource_group" "main" {
  name     = "rg-myapp-prod"
  location = "Japan East"
}

# ────────────────────────────────────────────────
# Storage Account（Lambda には不要だが Azure Functions には必須）
# ────────────────────────────────────────────────
resource "azurerm_storage_account" "main" {
  name                     = "stmyappprod001"   # 3〜24 文字、英小文字と数字のみ
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

# ────────────────────────────────────────────────
# Service Plan（ホスティングプラン：Lambda にはない概念）
# sku_name の代表例:
#   "FC1"      → Flex Consumption（新規推奨・サーバーレス）
#   "EP1"      → Elastic Premium（常時ウォーム）
#   "Y1"       → Consumption（旧型・非推奨）
# ────────────────────────────────────────────────
resource "azurerm_service_plan" "main" {
  name                = "asp-myapp-prod"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  os_type             = "Linux"
  sku_name            = "FC1"  # Flex Consumption
}

# ────────────────────────────────────────────────
# Function App 本体
# ────────────────────────────────────────────────
resource "azurerm_linux_function_app" "main" {
  name                = "func-myapp-prod"   # 最大 32 文字（重要！）
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location

  storage_account_name       = azurerm_storage_account.main.name
  storage_account_access_key = azurerm_storage_account.main.primary_access_key
  service_plan_id            = azurerm_service_plan.main.id

  # site_config は空でも必須ブロック（省略するとエラー）
  site_config {
    application_stack {
      python_version = "3.11"  # node_version / java_version 等も選択可
    }
  }

  # Lambda の環境変数に相当（app_settings で定義）
  app_settings = {
    "ENVIRONMENT"       = "production"
    "DATABASE_URL"      = var.database_url
    # Application Insights 連携（任意だが推奨）
    "APPINSIGHTS_INSTRUMENTATIONKEY" = azurerm_application_insights.main.instrumentation_key
  }
}

# Application Insights（Lambda の CloudWatch に相当、任意だが推奨）
resource "azurerm_log_analytics_workspace" "main" {
  name                = "log-myapp-prod"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "PerGB2018"
}

resource "azurerm_application_insights" "main" {
  name                = "appi-myapp-prod"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  workspace_id        = azurerm_log_analytics_workspace.main.id
  application_type    = "web"
}
```

**AWS との差分ポイント：**

| 項目 | AWS（Lambda） | Azure（Functions） |
|---|---|---|
| IAM ロール | 必須（`aws_iam_role` + `aws_iam_policy_document`） | 基本不要 |
| コンピューティング定義 | Lambda 自体に `memory_size` / `timeout` を直接指定 | `azurerm_service_plan` を別リソースで定義 |
| ストレージ | 不要（コードは ZIP/S3） | `azurerm_storage_account` が必須 |
| ランタイム指定 | `runtime = "python3.12"` | `site_config.application_stack.python_version = "3.11"` |
| 環境変数 | `environment.variables = {}` | `app_settings = {}` |

:::message alert
**Function App の名前は 32 文字以内**に収める必要があります。`func-mycompany-verylong-name-prod` のような長い名前は Host ID Collision を引き起こすことがあり、公式ドキュメントで注意喚起されています。
:::

---

## Azure API Management を Terraform で構築する

### APIM のリソース階層

APIM を Terraform で表現すると以下の階層になります。

```
azurerm_api_management           ← APIM インスタンス（API Gateway 本体）
└── azurerm_api_management_api   ← API（例: items-api）
    ├── azurerm_api_management_api_operation  ← 各エンドポイント（GET /items 等）
    │   └── azurerm_api_management_api_operation_policy  ← 操作単位のポリシー
    ├── azurerm_api_management_api_policy     ← API 全体のポリシー
    └── azurerm_api_management_backend        ← バックエンド（Functions の URL）
```

### ステップ 1：APIM インスタンスを作る

```hcl
resource "azurerm_api_management" "main" {
  name                = "apim-myapp-prod"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  publisher_name      = "MyCompany"          # 必須
  publisher_email     = "api@mycompany.com"  # 必須

  # sku_name は "Tier_capacity" の形式
  # Consumption_0 / Developer_1 / BasicV2_1 / StandardV2_1 / Premium_1
  sku_name = "Developer_1"  # 開発・検証向け（本番は BasicV2_1 以上を推奨）
}
```

:::message
**`azurerm_api_management` のプロビジョニングは非常に時間がかかります**（Developer で 15〜45 分）。`terraform apply` は `-parallelism=10` にしても変わりません。初回構築時は待ちましょう。
:::

`sku_name` の比較：

| sku_name | 月額コスト目安 | SLA | 本番用途 |
|---|---|---|---|
| `"Consumption_0"` | 使った分だけ | なし | 小規模・検証 |
| `"Developer_1"` | 低〜中 | なし | **本番非推奨**（開発専用） |
| `"BasicV2_1"` | 中 | あり | 小〜中規模本番（新規推奨） |
| `"StandardV2_1"` | 中〜高 | あり | 中規模本番（VNet 統合付き） |

### ステップ 2：API を定義する

```hcl
resource "azurerm_api_management_api" "items" {
  name                = "items-api"
  resource_group_name = azurerm_resource_group.main.name
  api_management_name = azurerm_api_management.main.name
  revision            = "1"                   # API のバージョン管理用
  display_name        = "Items API"
  path                = "items"               # ベースパス（/items/*）
  protocols           = ["https"]

  # OpenAPI 仕様を直接渡すことも可能（import ブロックを使う）
}
```

### ステップ 3：HTTP エンドポイント（Operation）を定義する

AWS では `aws_api_gateway_resource` + `aws_api_gateway_method` の2リソース組み合わせが必要でした。APIM では **1 つの `azurerm_api_management_api_operation`** にまとまります。

CRUD 5 エンドポイントの例：

```hcl
locals {
  operations = {
    list_items = {
      operation_id = "list-items"
      display_name = "List Items"
      method       = "GET"
      url_template = "/"
    }
    create_item = {
      operation_id = "create-item"
      display_name = "Create Item"
      method       = "POST"
      url_template = "/"
    }
    get_item = {
      operation_id = "get-item"
      display_name = "Get Item"
      method       = "GET"
      url_template = "/{id}"
    }
    update_item = {
      operation_id = "update-item"
      display_name = "Update Item"
      method       = "PUT"
      url_template = "/{id}"
    }
    delete_item = {
      operation_id = "delete-item"
      display_name = "Delete Item"
      method       = "DELETE"
      url_template = "/{id}"
    }
  }
}

resource "azurerm_api_management_api_operation" "items" {
  for_each = local.operations

  operation_id        = each.value.operation_id
  api_name            = azurerm_api_management_api.items.name
  api_management_name = azurerm_api_management.main.name
  resource_group_name = azurerm_resource_group.main.name
  display_name        = each.value.display_name
  method              = each.value.method
  url_template        = each.value.url_template

  # パスパラメータがある場合は template_parameter ブロックが必要
  dynamic "template_parameter" {
    for_each = can(regex("\\{id\\}", each.value.url_template)) ? [1] : []
    content {
      name     = "id"
      required = true
      type     = "string"
    }
  }
}
```

:::message
`url_template` に `{id}` のようなパスパラメータを含む場合、`template_parameter` ブロックを必ず追加しないとエラーになります。これは API Gateway の `request_parameters` に相当します。
:::

### ステップ 4：バックエンド（Functions）を登録する

```hcl
resource "azurerm_api_management_backend" "functions" {
  name                = "backend-functions"
  resource_group_name = azurerm_resource_group.main.name
  api_management_name = azurerm_api_management.main.name

  protocol    = "http"
  # Functions のホスト名。末尾スラッシュなしで書く
  url         = "https://${azurerm_linux_function_app.main.default_hostname}/api"

  # ARM リソース ID を渡すと APIM が Functions を認識しやすくなる
  resource_id = "https://management.azure.com${azurerm_linux_function_app.main.id}"

  credentials {
    header = {
      # Function Key で認証する場合（APIM から Functions を呼ぶ際の認証）
      "x-functions-key" = azurerm_linux_function_app.main.default_function_keys["default"]
    }
  }
}
```

**AWS との差分：**  
API Gateway + Lambda では `aws_lambda_permission` でゲートウェイからの呼び出しを明示的に許可する必要がありました。APIM + Functions では、**関数キーをヘッダーで渡すだけ**で認証が完結します（lambda_permission に相当する設定は不要）。

### ステップ 5：API 全体にポリシーを適用する

AWS では API Gateway でのレート制限・認証はコンソール設定や Lambda Authorizer が中心でした。APIM では **XML ポリシーを Terraform の `xml_content` に埋め込む**ことで宣言的に管理できます。

```hcl
# API 全体にポリシーを適用する（全 Operation に共通）
resource "azurerm_api_management_api_policy" "items" {
  api_name            = azurerm_api_management_api.items.name
  api_management_name = azurerm_api_management.main.name
  resource_group_name = azurerm_resource_group.main.name

  xml_content = <<XML
<policies>
  <inbound>
    <base />
    <!-- レート制限: 60秒で100回まで -->
    <rate-limit calls="100" renewal-period="60" />
    <!-- バックエンドに Functions を設定 -->
    <set-backend-service backend-id="${azurerm_api_management_backend.functions.name}" />
  </inbound>
  <backend>
    <base />
  </backend>
  <outbound>
    <base />
    <!-- レスポンスに CORS ヘッダーを追加 -->
    <set-header name="Access-Control-Allow-Origin" exists-action="override">
      <value>*</value>
    </set-header>
  </outbound>
  <on-error>
    <base />
  </on-error>
</policies>
XML
}
```

特定の Operation だけにポリシーを適用したい場合は `azurerm_api_management_api_operation_policy` を使います：

```hcl
# DELETE だけ追加で IP フィルタを適用する例
resource "azurerm_api_management_api_operation_policy" "delete_guard" {
  operation_id        = azurerm_api_management_api_operation.items["delete_item"].operation_id
  api_name            = azurerm_api_management_api.items.name
  api_management_name = azurerm_api_management.main.name
  resource_group_name = azurerm_resource_group.main.name

  xml_content = <<XML
<policies>
  <inbound>
    <base />
    <ip-filter action="allow">
      <address-range from="10.0.0.0" to="10.0.0.255" />
    </ip-filter>
  </inbound>
  <backend><base /></backend>
  <outbound><base /></outbound>
  <on-error><base /></on-error>
</policies>
XML
}
```

---

## 全体構成：リソース依存関係の図

```
azurerm_resource_group
├── azurerm_storage_account
├── azurerm_service_plan
├── azurerm_linux_function_app          ← depends_on: storage, service_plan
│   └── (default_hostname, function_key を output として参照)
├── azurerm_log_analytics_workspace
├── azurerm_application_insights        ← depends_on: log_analytics_workspace
└── azurerm_api_management
    ├── azurerm_api_management_backend  ← depends_on: linux_function_app
    ├── azurerm_api_management_api
    │   ├── azurerm_api_management_api_operation (×5: CRUD)
    │   │   └── azurerm_api_management_api_operation_policy (特定 operation のみ)
    │   └── azurerm_api_management_api_policy   ← depends_on: backend
    └── (必要に応じて azurerm_api_management_subscription, product 等)
```

---

## ハマりポイントまとめ

### 1. `site_config {}` は空でも省略不可

```hcl
# NG（エラーになる）
resource "azurerm_linux_function_app" "main" {
  ...
  # site_config がないと "An argument named site_config is required" エラー
}

# OK
resource "azurerm_linux_function_app" "main" {
  ...
  site_config {}  # 空ブロックでも必須
}
```

### 2. Function App 名は 32 文字以内

`func-mycompany-service-name-production` のような長い名前は Host ID Collision を引き起こします。命名規則を事前に決めておきましょう。

```hcl
# 推奨パターン（短く、一意性を確保）
name = "func-${var.app_name}-${var.env}"
# 例: func-items-prod（14文字）
```

### 3. APIM の `sku_name` は `"Tier_capacity"` 形式

```hcl
# NG
sku_name = "Developer"    # エラー：capacity が必要

# OK
sku_name = "Developer_1"  # Tier_capacity の形式で指定

# Consumption だけ capacity = 0 固定
sku_name = "Consumption_0"
```

### 4. Storage Account 名の制約

Lambda にはない制約ですが、Azure Storage Account 名は：
- **3〜24 文字**
- **英小文字と数字のみ**（ハイフン・アンダースコア不可）

```hcl
# NG
name = "my-storage-account"  # ハイフン不可

# OK
name = "stmyapprod001"
```

### 5. `azurerm_api_management` の適用は時間がかかる

Developer ティアでも初回 `apply` に **20〜40 分**かかることがあります。CI/CD パイプラインに組み込む場合はタイムアウト設定に注意が必要です。

### 6. AWS の `aws_api_gateway_deployment` に相当するものは不要

API Gateway では変更のたびに `aws_api_gateway_deployment` を作成して `aws_api_gateway_stage` に紐付ける必要がありました。APIM では Operation やポリシーを変更すると **即時反映**されます（デプロイ操作は不要）。

---

## outputs.tf の例

```hcl
output "function_app_hostname" {
  value = azurerm_linux_function_app.main.default_hostname
}

output "apim_gateway_url" {
  value = azurerm_api_management.main.gateway_url
}

output "items_api_url" {
  value = "${azurerm_api_management.main.gateway_url}/${azurerm_api_management_api.items.path}"
}
```

---

## まとめ

| 観点 | AWS（Lambda + API Gateway） | Azure（Functions + APIM） |
|---|---|---|
| 最低リソース数（Functions 側） | 2〜3（lambda + IAM role） | 4（resource group + storage + service plan + function app） |
| API 定義リソース数（CRUD 5エンドポイント） | 約 15 リソース | 約 8〜10 リソース |
| ポリシー・認証設定 | Lambda Authorizer（コード）/ コンソール設定 | XML ポリシー（Terraform に埋め込み） |
| エンドポイントと統合の定義 | resource + method + integration を分離 | api_operation 1つにまとまる |
| デプロイ操作 | `aws_api_gateway_deployment` が必要 | 不要（即時反映） |
| 初回 apply 時間 | Lambda は数十秒 | APIM は 20〜40 分 |

Azure Functions + APIM の Terraform は**リソース数こそ多めですが、階層構造が明確**で読みやすいコードになります。AWS の `gateway_integration` + `gateway_method` + `gateway_resource` の組み合わせに慣れていれば、APIM の `api → operation → policy` の流れは直感的に理解できるはずです。

---

## 参考リンク

- [azurerm_linux_function_app - Terraform Registry](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/linux_function_app)
- [azurerm_api_management - Terraform Registry](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/api_management)
- [azurerm_api_management_api_operation - Terraform Registry](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/api_management_api_operation)
- [azurerm_api_management_backend - Terraform Registry](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/api_management_backend)
- [azurerm_api_management_api_operation_policy - Terraform Registry](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/api_management_api_operation_policy)
- [Quickstart: Azure Functions with Terraform（公式）](https://learn.microsoft.com/en-us/azure/azure-functions/functions-create-first-function-terraform)
- [Quickstart: Azure API Management with Terraform（公式）](https://learn.microsoft.com/en-us/azure/api-management/quickstart-terraform)
