---
title: Azure Functions 入門 ― サーバーレスの仕組みと使いどころを理解する
status: plan
---

## 記事タイトル候補

1. 「Azure Functions 入門 ― サーバーレスの仕組みと「使う・使わない」の判断軸を整理する」
2. 「はじめての Azure Functions ― コードだけに集中できるサーバーレスとは何か」
3. 「Azure Functions を使うべき場面、使わない場面 ― 初学者のための判断ガイド」
4. 「Azure Functions ことはじめ ― App Service・Durable Functions との比較で理解するサーバーレス設計」

**推奨:** 1 番（判断軸の整理を前面に出す点が初学者に刺さりやすい）

---

## 想定読者

- クラウド初学者・Azure を触り始めたエンジニア
- 「サーバーレス」という言葉は聞いたことがあるが、どう使うか迷っている人
- App Service との違い・使い分けがわからない人
- 前提知識: HTTP の基礎、簡単なプログラミング経験があれば OK

## 記事で答える問い

- Azure Functions とは何か、なぜ「サーバーレス」と呼ばれるのか
- Azure Functions を使うべき場面・使わない場面（App Service / Durable Functions との比較）
- 代表的なユースケースと、それぞれどう実現するか
- Azure Functions とよく組み合わせて使われるサービスは何か

## 扱う範囲・扱わない範囲

### 扱う
- Azure Functions の基本概念（トリガー・バインディング・ホスティングプラン）
- App Service、Durable Functions との使い分け判断
- 代表的なユースケース（HTTP API、バッチ処理、イベント駆動）
- 連携サービスの紹介（概念レベル）

### 扱わない
- コードの実装詳細・言語別チュートリアル
- CI/CD パイプラインの構築
- セキュリティ設計の詳細（RBAC、ネットワーク隔離など）
- Azure API Management との組み合わせ（別記事で扱い済み）

---

## セクション構成

### 0. はじめに

**主張**: クラウドでアプリを動かすとき「サーバーの管理をどこまで自分でやるか」は重要な設計判断。Azure Functions はその管理を極限まで Azure に委ねる選択肢。

---

### 1. Azure Functions とは

**主張**: Azure Functions は「イベントが起きたらコードを実行する」仕組みを提供するサーバーレスコンピューティングサービス。インフラの管理なしにコードだけを書けばよい。

**カバーする内容**:
- サーバーレスの意味（インフラを意識せず、実行した分だけ課金）
- Function App と Function の関係（Function App = 複数の Function をまとめる箱）
- **トリガー**（関数を起動するイベント）と**バインディング**（外部サービスへの接続をコードなしで宣言）の概念
- 対応言語：C#, Java, JavaScript/TypeScript, Python, PowerShell

**根拠 URL**:
- https://learn.microsoft.com/en-us/azure/azure-functions/functions-overview
- https://learn.microsoft.com/en-us/azure/azure-functions/functions-triggers-bindings

---

### 2. ホスティングプランを知る

**主張**: Azure Functions には 3 つの主要ホスティングプランがあり、プランによってコスト・スケール・制約が異なる。まず Consumption プランの特性を理解すれば入門として十分。

**カバーする内容**:

| プラン | 特徴 | コスト | 向いているケース |
|---|---|---|---|
| **Consumption（従量課金）** | 実行した分だけ課金、自動スケール | 実行回数・実行時間に応じた従量課金（月180万回まで無料） | 実行頻度が低い・予測困難な処理 |
| **Flex Consumption** | 関数ごとのスケール、Always Readyインスタンス対応 | 従量課金＋Always Readyインスタンス分の固定費 | コールドスタートを減らしたいケース |
| **Premium** | 常時起動インスタンスあり、VNet 統合対応 | 常時稼働インスタンス分の固定費（高め） | コールドスタート許容不可、セキュアな接続が必要 |
| **Dedicated（App Service plan）** | 既存 App Service プランで実行 | App Service プランの料金に含まれる（追加コストなし） | VM を他サービスと共有して使い回したいケース |

- コールドスタートの説明（Consumption は初回起動に数秒かかる場合がある）

**根拠 URL**:
- https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale

---

### 3. Azure Functions を使う場合・使わない場合

**主張**: Azure Functions は「短時間・イベント駆動」の処理に強い。長時間実行・常時起動・フルスタック Web アプリには App Service や Durable Functions が向く。

**カバーする内容**:

#### 3-1. App Service を選ぶべき場合

- Web アプリ全体（フロントエンド + バックエンド）を動かしたい
- 常時起動が必要でコールドスタートを一切許容できない
- 独自ミドルウェアや複雑なサーバー設定が必要
- 既存の App Service 環境に乗せたい

#### 3-2. Durable Functions を選ぶべき場合（Azure Functions の拡張）

- 複数の処理を順番に組み合わせる長時間ワークフローが必要
- 人の承認待ちなど「途中で止まる処理」がある
- Azure Functions のタイムアウト制限（Consumption プランのデフォルト 5 分）を超えるケース
- 複数の並列処理をまとめてオーケストレーションしたい

#### 3-3. Logic Apps を選ぶべき場合

- コードを書かず GUI で連携フローを組みたい
- 1,400+ のコネクタで SaaS サービスと手軽に繋ぎたい
- 非エンジニアがメンテナンスする想定

#### 3-4. Azure Functions が適しているケース

- HTTP API として短時間のレスポンスを返す
- ファイルアップロード・DB 更新・メッセージ到達などのイベントで処理を動かす
- 夜間バッチなどスケジュール実行（Timer トリガー）
- 軽量なマイクロサービスの 1 機能

**根拠 URL**:
- https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale
- https://learn.microsoft.com/en-us/azure/azure-functions/functions-compare-logic-apps-ms-flow-webjobs

---

### 4. 代表的なユースケースと構成例

**主張**: Azure Functions は幅広い場面で使えるが、特に「イベント駆動バックエンド」「スケジュールバッチ」「軽量 API」の 3 パターンを押さえれば大半のシナリオをカバーできる。

**カバーする内容**:

#### 4-1. HTTP API として使う（HTTP トリガー）
- シンプルな REST API エンドポイントを作るシナリオ
- 例: フォーム送信を受け取り DB に保存する

#### 4-2. ファイル処理・イベント連携（Blob / Event Grid トリガー）
- 例: Blob Storage に画像がアップロードされたら自動でリサイズ処理

#### 4-3. メッセージキューの処理（Service Bus / Queue Storage トリガー）
- 例: 注文メッセージをキューから受け取り、在庫 DB を更新する

#### 4-4. スケジュール実行（Timer トリガー）
- 例: 毎日深夜にレポートを生成して Storage に保存する

**根拠 URL**:
- https://learn.microsoft.com/en-us/azure/azure-functions/functions-overview（Scenarios セクション）
- https://learn.microsoft.com/en-us/azure/azure-functions/functions-triggers-bindings

---

### 5. よく一緒に使われる Azure サービス

**主張**: Azure Functions 単体より、周辺サービスと組み合わせることで価値が倍増する。代表サービスの役割を知っておくと設計判断が早くなる。

**カバーする内容**:

| サービス | 役割 | Functions との関係 |
|---|---|---|
| **Azure Blob Storage** | ファイル保存 | Blob トリガー・入出力バインディング |
| **Azure Queue Storage** | 軽量メッセージキュー | Queue トリガー |
| **Azure Service Bus** | エンタープライズメッセージング | Service Bus トリガー・出力バインディング |
| **Azure Event Grid** | イベントルーティング | Event Grid トリガー |
| **Azure Event Hubs** | 大量イベントのストリーミング | Event Hubs トリガー |
| **Azure Cosmos DB** | NoSQL DB | Cosmos DB トリガー・入出力バインディング |
| **Azure API Management** | API ゲートウェイ（認証・レート制限） | Functions の前段に配置 |
| **Application Insights** | ログ・監視・パフォーマンス計測 | Functions と標準統合 |

**根拠 URL**:
- https://learn.microsoft.com/en-us/azure/azure-functions/functions-triggers-bindings
- https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-event-grid
- https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-event-hubs

---

### 6. まとめ ― Azure Functions を選ぶための判断フロー

**主張**: 「短時間」「イベント駆動」「スケール自動化」の 3 条件が揃えば Azure Functions は強力な選択肢。迷ったら本記事の判断フローを参照。

**カバーする内容**:
- 簡易フローチャート形式のまとめ
  - 長時間 or 常時起動 → App Service / Durable Functions
  - コード不要でつなぎたい → Logic Apps
  - イベント駆動・短時間・スケール必要 → Azure Functions ✅
- 次のステップへの誘導（公式チュートリアル、関連記事）

---

## 調査ファイルと根拠 URL

| 調査ファイル | 内容 |
|---|---|
| `temp/azure-functions-intro/search_azure_functions_overview.json` | 概要・公式Docs URL一覧 |
| `temp/azure-functions-intro/search_azure_functions_comparison.json` | 比較・使い分けの情報 |
| `temp/azure-functions-intro/search_azure_functions_services.json` | 連携サービス・トリガー一覧 |

### 主要参照 URL

- https://learn.microsoft.com/en-us/azure/azure-functions/functions-overview
- https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale
- https://learn.microsoft.com/en-us/azure/azure-functions/functions-triggers-bindings
- https://learn.microsoft.com/en-us/azure/azure-functions/functions-compare-logic-apps-ms-flow-webjobs
- https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-event-grid
- https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-event-hubs

---

## 追加調査が必要な箇所（本文執筆前に確認推奨）

- Consumption プランのデフォルトタイムアウト（5分）と最大タイムアウトの最新値
- Flex Consumption プランの GA 状況（Preview か GA か）
- 各ホスティングプランの料金体系の最新情報
