---
title: Azure Durable Functions 入門：サーバーレスで"長い処理"を扱う方法
status: plan
---

## 記事メタ情報

| 項目 | 内容 |
|---|---|
| 想定読者 | Azure 初学者。App Service / Azure Functions の名前は聞いたことがある程度 |
| 記事が答える問い | なぜ普通の Azure Functions では足りないのか？Durable Functions はどんなときに使えばよいのか？ |
| 扱う範囲 | Azure 計算サービスの整理・Durable Functions の概念・使いどころ・主要パターン・関連サービス |
| 扱わない範囲 | コード実装の詳細・料金の詳細計算・Durable Task SDK の詳細・Preview 機能 |

---

## セクション構成

### 1. はじめに

**主張:** Azure の「計算サービス」は複数あり、それぞれ役割が違う。まず全体像を把握しないと Durable Functions の位置づけが分からない。

**内容:**
- 「Webアプリを作るなら App Service？それとも Functions？」という初学者の疑問から入る
- Azure の主な計算サービスを一枚図で整理（App Service / Functions / Durable Functions の関係）
- この記事を読むと何が分かるかを明示する

**根拠 URL:**
- https://learn.microsoft.com/en-us/answers/questions/5606861/what-s-the-difference-between-azure-app-service-an

---

### 2. Azure App Service とは

**主張:** App Service は「常時起動の Web アプリ・API」をホストするためのプラットフォームであり、サーバーレスではない。

**内容:**
- Web アプリ・REST API・モバイルバックエンドをホストするマネージドサービス
- 特徴：常時起動・常時課金・スケールアップは手動または設定ベース
- サポート言語：.NET、Java、Python、Node.js、PHP
- 内蔵機能：CI/CD 連携・デプロイスロット・カスタムドメイン・SSL・高度なネットワーク設定
- 向いているケース：長時間稼働 Web アプリ、継続的な可用性が必要な API

**根拠 URL:**
- https://learn.microsoft.com/en-us/answers/questions/5606861/what-s-the-difference-between-azure-app-service-an
- https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale

---

### 3. Azure Functions とは（Durable Functions の土台）

**主張:** Azure Functions はイベント駆動・ステートレス・短命の処理に特化したサービスであり、Durable Functions の土台になる。

**内容:**
- イベント（HTTP・Queue・Timer など）をトリガーに関数が実行される
- ステートレス・短命が原則（HTTP トリガーは最大 230 秒でタイムアウト）
- 従量課金（実行回数・実行時間・メモリ使用量）
- 向いているケース：単発 API、通知送信、イベント処理、マイクロサービス
- 向いていないケース：長時間・複数ステップ・状態管理が必要な処理

**根拠 URL:**
- https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale
- https://learn.microsoft.com/en-us/azure/azure-functions/functions-best-practices
- https://learn.microsoft.com/en-us/azure/azure-functions/functions-compare-logic-apps-ms-flow-webjobs

---

### 4. Azure Durable Functions とは

**主張:** Durable Functions は Azure Functions の拡張機能であり、ステートフルなワークフローを関数で実現できる。

**内容:**
- Azure Functions の extension（追加機能）として動作する
- 「ステートフル」＝複数ステップをまたいで状態を保持できる
- 構成要素を表で整理：

  | コンポーネント | 役割 |
  |---|---|
  | Orchestrator（オーケストレーター） | ワークフロー全体を制御する指揮者 |
  | Activity（アクティビティ） | 実際の処理を行う最小単位（DB 書き込み、API 呼び出しなど） |
  | Entity（エンティティ） | 小さな状態を管理するオブジェクト（カウンター・フラグなど） |
  | Client（クライアント） | オーケストレーションを開始・管理する |

- 内部では Azure Storage（Queue・Blob・Table）を使って状態を自動管理する
- 開発者はビジネスロジックだけに集中できる

**根拠 URL:**
- https://learn.microsoft.com/en-us/azure/azure-functions/durable-functions/durable-functions-overview
- https://learn.microsoft.com/en-us/azure/durable-task/common/programming-model-overview

---

### 5. 使う場合・使わない場合

**主張:** 普通の Azure Functions で足りるかどうかは「処理が長いか」「状態管理が必要か」「複数ステップか」の3軸で判断できる。

**内容:**

**使う場合（Durable Functions が向いている）:**
- 処理時間が長い（分・時間・日単位）
- 複数の関数をまたいで状態を保持する必要がある
- 人の承認・外部イベントの待機が必要
- 並列処理の結果をまとめて次のステップに渡す

**使わない場合（普通の Functions で十分）:**
- 単発の短い処理（数秒以内で完了する）
- ステートレスで完結する（関数間で状態を引き継がない）
- シンプルな API エンドポイント・通知送信

**判断表（初学者向け）:**

| 条件 | 普通の Functions | Durable Functions |
|---|---|---|
| 処理時間 | 数秒〜数分 | 数分〜数日 |
| 状態管理 | 不要 | 必要 |
| ステップ数 | 1 ステップ | 複数ステップ |
| 人の承認待ち | なし | あり |
| 並列+集約 | 困難 | 得意 |

**根拠 URL:**
- https://learn.microsoft.com/en-us/archive/msdn-magazine/2019/august/azure-affairs-of-state-serverless-and-stateless-code-execution-with-azure-functions
- https://learn.microsoft.com/en-us/azure/azure-functions/performance-reliability
- https://learn.microsoft.com/en-us/answers/questions/5855574/best-practices-for-beginners-in-azure-functions
- https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale

---

### 6. 主要パターン（シナリオ）5種

**主張:** Durable Functions には代表的な5つのパターンがあり、どれも「普通の Functions では難しい処理」を解決する。

**内容（各パターンを概要＋具体例で紹介）:**

#### 6-1. Function Chaining（関数の連鎖実行）
- A → B → C の順に実行し、前の出力を次の入力にする
- 例：データ取得 → 変換 → DB 保存

#### 6-2. Fan-out / Fan-in（並列実行して集約）
- 複数の処理を並列実行し、全部終わったら結果をまとめる
- 例：複数 API を並列呼び出し → 結果を統合してレポート生成

#### 6-3. Human Interaction（人の承認待ち）
- 外部イベント（人の入力・承認）を待機してから処理を続ける
- 例：申請フロー（承認 / 却下を数日間待つ）、SMS 認証コード検証
- 待機中は compute コストが発生しない

#### 6-4. Monitor（ポーリング監視）
- 条件が満たされるまで定期的に状態をチェックし続ける
- 例：外部バッチジョブの完了待機、在庫変動の監視

#### 6-5. Stateful Entities（ステートフルエンティティ）
- 長期間にわたって小さな状態を保持・操作するオブジェクト
- 例：カウンター、ショッピングカート、セッション管理

**根拠 URL:**
- https://learn.microsoft.com/en-us/azure/durable-task （パターン一覧）
- https://learn.microsoft.com/en-us/azure/durable-task/common/durable-task-sequence （Function Chaining）
- https://learn.microsoft.com/en-us/azure/durable-task/common/durable-task-human-interaction （Human Interaction）

---

### 7. 一緒に使われるサービス

**主張:** Durable Functions は単体では動かず、必ず Azure Storage が必要。用途に応じて他のサービスと組み合わせる。

**内容（表で整理）:**

| サービス | 役割 | 必須/任意 |
|---|---|---|
| Azure Storage（Queue / Blob / Table） | Durable Functions の内部状態保存・メッセージ管理に必須のバックエンド | **必須** |
| Durable Task Scheduler（DTS） | マネージド型のスケジューラー。Storage よりも運用が楽になる新しい選択肢 | 任意 |
| Azure Service Bus | より高信頼・高スループットのメッセージング（Storage Queue の代替） | 任意 |
| Azure Event Grid | イベント駆動トリガー・オーケストレーション完了通知の送信 | 任意 |
| Azure Logic Apps | ノーコード・ローコードのワークフロー。Durable Functions と使い分け or 併用 | 任意 |
| Application Insights | 実行ログ・パフォーマンス監視・エラートレース | 強く推奨 |

**各サービスの補足:**
- **Azure Storage** は Durable Functions を使う限り切り離せない。別の Function App と共有しないことが推奨される（パフォーマンス上の理由）
- **Durable Task Scheduler** は 2025 年以降に登場した新しいマネージドバックエンド。MSSQL や Storage の代替として利用可能
- **Event Grid** は `host.json` の `eventGridTopicEndpoint` 設定でオーケストレーションの状態変化（開始・完了・失敗）を通知できる

**根拠 URL:**
- https://learn.microsoft.com/en-us/azure/azure-functions/performance-reliability
- https://learn.microsoft.com/en-us/azure/azure-functions/storage-considerations
- https://learn.microsoft.com/en-us/azure/azure-functions/functions-host-json-v1
- https://learn.microsoft.com/en-us/azure/container-apps/functions-overview

---

### 8. まとめ

**主張:** Durable Functions は「普通の Functions では難しい複雑・長時間・ステートフルな処理」を解決する。まずは使いどころを理解することが第一歩。

**内容:**
- 3サービスの使い分けを一言でまとめる
  - App Service → 常時稼働のWebアプリ・API
  - Azure Functions → 短命・ステートレスなイベント処理
  - Durable Functions → 長時間・複数ステップ・状態管理が必要なワークフロー
- 「まずは Function Chaining から試してみよう」という次のアクションを提示
- 公式クイックスタートへのリンクを案内

---

## 調査ファイル一覧

| ファイル | 内容 |
|---|---|
| `temp\azure_durable_functions_intro\search_durable_functions_overview.json` | Durable Functions 概要・GitHub |
| `temp\azure_durable_functions_intro\search_appservice_vs_functions.json` | App Service vs Functions 比較 |
| `temp\azure_durable_functions_intro\search_durable_functions_usecases.json` | ユースケース・ベストプラクティス |
| `temp\azure_durable_functions_intro\extract_durable_overview.json` | 公式概要ページ・プログラミングモデル抽出 |
| `temp\azure_durable_functions_intro\extract_appservice_functions_compare.json` | 比較ページ・スケール/ホスティング抽出 |
| `temp\azure_durable_functions_intro\search_durable_patterns.json` | パターン（Chaining・Fan-out・Human Interaction 等） |
| `temp\azure_dural_functions_intro\search_related_services.json` | 関連サービス（Storage・Event Grid 等） |

## 不足情報・追加調査ポイント

- Durable Task Scheduler（DTS）の詳細（2025 年以降の新機能のため情報が薄い）
- 各ホスティングプラン（Consumption / Flex Consumption / Premium）での Durable Functions の制約の違い
- コスト感の具体的な数値（待機中は課金されないことの根拠 URL）
