---
title: "Azure Functions 入門 ― サーバーレスの仕組みと「使う・使わない」の判断軸を整理する"
emoji: "⚡"
type: "tech"
topics: ["azure", "azurefunctions", "serverless", "cloud"]
published: false
---

## はじめに

「Azure Functions って何？」「App Service と何が違うの？」

Azure でサーバーレスを触り始めると、似たような名前のサービスが並んでいて判断に迷います。この記事では **Azure Functions の基本的な仕組み**と、**どんな場合に使うべきか・使わない方がよいか**の判断軸を整理します。コードの書き方よりも「そもそも何ができて、どう選ぶか」を重点的に説明するので、初学者の方でもついてこられる内容です。

---

## 1. Azure Functions とは

### サーバーレスとは何か

クラウドでアプリを動かすとき、「サーバー（仮想マシン）を自分で管理する」か「クラウドに管理を任せる」かという選択肢があります。

- **IaaS（仮想マシン）**: OS・ミドルウェア・スケーリングをすべて自分で管理
- **PaaS（App Service）**: OS やランタイムはクラウドが管理してくれる。ただし、常時起動のサーバー代はかかる
- **サーバーレス（Azure Functions）**: インフラをほぼ意識せずに済み、**コードが実行された分だけ課金**

Azure Functions は「**イベントが起きたときだけコードを実行する**」という仕組みのサーバーレスサービスです。実行時間が短ければ短いほど低コストで運用でき、スケールも Azure が自動で行います。

公式の定義は次のとおりです。

> Azure Functions is a serverless solution that allows you to build robust apps while using less code, and with less infrastructure and lower costs. Instead of worrying about deploying and maintaining servers, you can use the cloud infrastructure to provide all the up-to-date resources needed to keep your applications running.
>
> （和訳）Azure Functionsは、より少ないコードでより少ないインフラ、そして低コストで堅牢なアプリを構築できるサーバーレスソリューションです。サーバーのデプロイや維持管理を気にする代わりに、クラウドインフラを活用してアプリケーションの稼働に必要な最新リソースをすべて提供できます。
>
> — [What is Azure Functions? | Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-functions/functions-overview)

### Function App と Function の関係

Azure Functions には「Function App」と「Function」という 2 つの単位があります。

```
Function App（コンテナ）
├── Function A（HTTPトリガー）
├── Function B（Timerトリガー）
└── Function C（Blobトリガー）
```

- **Function**：実際に実行される 1 つの処理単位（コード）
- **Function App**：複数の Function をまとめて管理する箱。デプロイ・スケール・設定はこの単位で行う

### 対応言語

以下の言語に公式対応しています。

- C# / .NET
- Java
- JavaScript / TypeScript (Node.js)
- Python
- PowerShell

また、カスタムハンドラーを使えば Go や Rust でも動かせます。

---

## 2. トリガーとバインディング

Azure Functions を理解する上でもっとも重要な概念が **トリガー** と **バインディング** です。

### トリガー：関数を起動するイベント

トリガーは「何をきっかけに関数を実行するか」を定義します。1 つの Function につき 1 つのトリガーが必ず必要です。

| トリガー | 説明 | 典型的なユースケース |
|---|---|---|
| **HTTP** | HTTP リクエストを受信したら実行 | REST API、Webhook |
| **Timer** | cron 式で定期実行 | 夜間バッチ、定期レポート生成 |
| **Blob Storage** | Blob（ファイル）がアップロード/更新されたら実行 | 画像のリサイズ、ファイル変換 |
| **Queue Storage** | キューにメッセージが追加されたら実行 | 非同期処理のワーカー |
| **Service Bus** | Service Bus のキュー/トピックにメッセージが届いたら実行 | 信頼性の高いメッセージング |
| **Event Grid** | イベントが発生したら実行 | Azure サービス横断のイベント連携 |
| **Event Hubs** | イベントストリームにデータが届いたら実行 | IoT・大量ログのリアルタイム処理 |
| **Cosmos DB** | ドキュメントが作成/更新されたら実行 | DB 変更に連動した処理 |

### バインディング：外部サービスへの接続をコードレスで宣言する

バインディングは「関数が外部サービスからデータを受け取ったり、書き込んだりする方法」を宣言的に定義する仕組みです。

- **Input バインディング**：外部サービスからデータを読み込む（接続コードを自分で書かなくてよい）
- **Output バインディング**：外部サービスへデータを書き込む（接続コードを自分で書かなくてよい）

例えば「Blob にファイルがアップロードされたら処理して、結果を Cosmos DB に保存する」という関数は、こう書けます。

```
[BlobTrigger]       ← Blob Storage を監視（トリガー）
[CosmosDB Output]   ← 処理結果を Cosmos DB へ書き込む（Output バインディング）
↓
function(file, outputDoc) {
    // ファイルを処理して outputDoc に結果を書くだけ
    // 接続文字列の管理・SDK の呼び出しは Azure が肩代わりしてくれる
}
```

バインディングにより、ボイラープレートコード（接続・認証・エラーハンドリング）を大幅に省略できるのが Azure Functions の大きな特徴です。

---

## 3. ホスティングプランを知る

Azure Functions は複数の「ホスティングプラン」から選べます。プランによってコスト・スケール方式・制限が変わります。

### プラン比較

| プラン | 課金モデル | コスト概算 | コールドスタート | タイムアウト（デフォルト/最大） | VNet 統合 |
|---|---|---|---|---|---|
| **Flex Consumption**（推奨） | 実行量課金 | 実行時間 $0.000016/GB秒 + $0.20/100万回（Always Ready追加時は常時起動分を加算） | 少ない（Always Ready対応） | 30分 / 無制限 | ✅ |
| **Premium** | 常時起動インスタンスの時間課金 | EP1（1コア・3.5GB）で約 $143/月〜 | なし（常時起動） | 30分 / 無制限 | ✅ |
| **Dedicated（App Service プラン）** | App Service プランの時間課金 | B1（1コア・1.75GB）で約 $13/月〜（他サービスと共有なら追加コストなし） | なし（常時起動） | 30分 / 無制限 | ✅ |
| **Consumption**（レガシー） | 実行量課金 | 実行時間 $0.000016/GB秒 + $0.20/100万回（月180万回・40万GB秒まで無料） | あり | 5分 / 10分 | ❌ |

> 公式ドキュメントでは **Flex Consumption が推奨**（新規アプリのデフォルト）と明記されています。旧来の Consumption プランは Windows のみ対応のレガシー扱いとなっており、新規作成には Flex Consumption を選ぶのが基本です。

参考: [Azure Functions のスケールとホスティング | Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale)

### コールドスタートとは

Consumption 系のプランでは、一定時間アクセスがないとインスタンスがゼロに縮小されます。次にリクエストが来たとき、新しいインスタンスを起動するまでに数秒の遅延（コールドスタート）が発生することがあります。

- **許容できる場合**：バッチ処理・非同期ワーカーなど、多少のレイテンシが問題にならない用途
- **許容できない場合**：ユーザー向け同期 API などリアルタイムなレスポンスが求められる場合 → Premium または Dedicated プランを検討

---

## 4. Azure Functions を使う場合・使わない場合

「なんでも Functions でよいでは？」と思うかもしれませんが、Azure には目的別のサービスがあり、使い分けが大切です。

### Azure Functions が適している場合

以下の条件が重なるほど、Azure Functions はよいフィットです。

- ✅ **イベント駆動**で動かしたい（ファイルが来たら、メッセージが届いたら、etc.）
- ✅ **短時間で完了**する処理である（数秒〜数分）
- ✅ **実行頻度が不規則**で、常時起動はもったいない
- ✅ コードだけに集中して、インフラ管理を省きたい
- ✅ 自動スケールが必要で、トラフィック予測が難しい

### App Service を選ぶべき場合

| 状況 | 理由 |
|---|---|
| フロントエンド + バックエンドを一体で動かしたい | App Service は Web アプリ全体のホスティングに向く |
| 常時起動が必要でコールドスタートを一切許容できない | Functions の Consumption プランはコールドスタートがある |
| 複雑なサーバー設定・カスタムミドルウェアが必要 | Functions はランタイムの柔軟性に制限がある |
| 既存の App Service 環境を流用したい | Dedicated プランで Functions を動かす選択肢もあるが、素直に App Service を使う方がシンプル |

### Durable Functions を選ぶべき場合

**Durable Functions** は Azure Functions の拡張機能で、ステートフルなワークフロー（複数の関数を順番に実行するオーケストレーション）を記述できます。

| 状況 | 理由 |
|---|---|
| 処理が長時間に及ぶ（タイムアウトを超える） | Durable Functions は実行時間が無制限 |
| 人の承認待ちなど「途中で止まる処理」がある | 外部イベントを待機する仕組みがある |
| 複数の処理を並列実行し、全部完了を待ちたい | Fan-out/Fan-in パターンに対応 |
| 処理の途中経過を保存してリトライしたい | ステートが永続化される |

通常の Azure Functions は「1 回起動 → 処理 → 終了」ですが、Durable Functions は「起動 → 別の関数を呼ぶ → 待機 → 再開 → 終了」というワークフローを表現できます。

### Azure Logic Apps を選ぶべき場合

| 状況 | 理由 |
|---|---|
| コードを書かずに GUI でフローを組みたい | Logic Apps はノーコード/ローコードのワークフロービルダー |
| SaaS サービスと手軽に接続したい | 1,400 以上のコネクタ（Salesforce、Teams、Outlook 等）を標準搭載 |
| 非エンジニアがメンテナンスする想定がある | ビジュアルデザイナーで誰でも編集できる |

Logic Apps は「コードで書きたい複雑なロジック」より「既存サービスを繋ぐ自動化」に向いています。

### まとめ：判断フロー

```
処理をどう動かしたいか？
│
├─ GUIでコードなしに繋ぎたい
│   └─→ Logic Apps
│
├─ 長時間ワークフロー・承認待ちが必要
│   └─→ Durable Functions
│
├─ Web アプリ全体・常時起動・複雑なサーバー設定
│   └─→ App Service
│
└─ イベント駆動・短時間・自動スケール
    └─→ Azure Functions ✅
```

---

## 5. 代表的なユースケース

Azure Functions が実際にどんな場面で使われるか、4 つのパターンを紹介します。

### 5-1. HTTP API として使う（HTTP トリガー）

もっともシンプルな使い方。HTTP リクエストを受けてレスポンスを返す REST API を、サーバーの管理なしに作れます。

**シナリオ例**:
- お問い合わせフォームの送信を受け取り、DB に保存して確認メールを送る
- モバイルアプリのバックエンド API を軽量に作る

```
クライアント ──HTTP POST──▶ Azure Functions ──▶ Cosmos DB に保存
                                             └──▶ メール送信（Output バインディング）
```

注意点：HTTP トリガーは Azure Load Balancer の制約上、**230秒でタイムアウト**します（プラン設定に関わらず）。長時間の処理は Durable Functions の Async パターンに委ねましょう。

### 5-2. ファイルアップロード時の処理（Blob トリガー）

Blob Storage にファイルが届いたら自動で関数を実行するパターン。ストレージとビジネスロジックをポーリングなしで繋げます。

**シナリオ例**:
- ユーザーがアップロードした画像を自動でリサイズ・サムネイル生成
- CSV ファイルがアップロードされたらパースして DB に取り込む

```
Blob Storage（ファイルアップロード） ──▶ Azure Functions ──▶ リサイズ後の画像を別 Blob に保存
```

### 5-3. メッセージキューの処理（Queue / Service Bus トリガー）

キューにメッセージが溜まったら関数を起動するパターン。処理を非同期化でき、バースト時のスケールも自動で行われます。

**シナリオ例**:
- EC サイトの注文が入ったらキューに積み、在庫確認・メール送信を非同期処理
- 外部サービスへの API 呼び出しをキューで直列化して過負荷を防ぐ

```
注文 API ──▶ Service Bus キュー ──▶ Azure Functions ──▶ 在庫 DB 更新
                                                     └──▶ 発送メール送信
```

### 5-4. スケジュール実行（Timer トリガー）

cron 式でスケジュールを設定し、定期的にコードを実行するパターン。常時起動サーバーが不要なので低コストです。

**シナリオ例**:
- 毎日深夜に前日の売上レポートを集計して Blob Storage に保存
- 1 時間おきに外部 API からデータを取得してキャッシュを更新

```
タイマー（毎日 02:00） ──▶ Azure Functions ──▶ DB からデータ集計 ──▶ レポートを Blob に保存
```

---

## 6. よく一緒に使われる Azure サービス

Azure Functions は他の Azure サービスと組み合わせることで真価を発揮します。代表的な連携先を紹介します。

### ストレージ・データ系

| サービス | 役割 | Functions との接続 |
|---|---|---|
| **Azure Blob Storage** | ファイル・バイナリデータの保存 | Blob トリガー / Input・Output バインディング |
| **Azure Queue Storage** | シンプルなメッセージキュー | Queue トリガー / Output バインディング |
| **Azure Cosmos DB** | グローバル分散 NoSQL DB | Cosmos DB トリガー / Input・Output バインディング |
| **Azure SQL Database** | リレーショナル DB | Output バインディング（Azure SQL extension） |
| **Azure Table Storage** | シンプルな Key-Value ストア | Input・Output バインディング |

### メッセージング・イベント系

| サービス | 役割 | Functions との接続 |
|---|---|---|
| **Azure Service Bus** | エンタープライズグレードのメッセージキュー/トピック | Service Bus トリガー / Output バインディング |
| **Azure Event Grid** | Azure サービス横断のイベントルーティング | Event Grid トリガー / Output バインディング |
| **Azure Event Hubs** | 大量イベント・IoT データのストリーミング | Event Hubs トリガー / Output バインディング |

### API・監視系

| サービス | 役割 | Functions との接続 |
|---|---|---|
| **Azure API Management** | API ゲートウェイ（認証・レート制限・ドキュメント） | Functions の前段に配置して API を保護・管理 |
| **Application Insights** | ログ収集・パフォーマンス監視・アラート | Functions と標準統合。実行ログが自動で収集される |

### サービス選択の参考：「どのメッセージングサービスを使うか」

初学者が特に迷いやすいのがメッセージング系のサービス選択です。簡単な目安を示します。

| 状況 | 向いているサービス |
|---|---|
| シンプルで安価なキューがほしい | Queue Storage |
| デッドレターキュー・トランザクション・高信頼性が必要 | Service Bus |
| Azure サービス間のイベント（Blob 作成通知など）を扱いたい | Event Grid |
| 秒間数百万件のイベントをリアルタイムに処理したい | Event Hubs |

---

## 7. まとめ

Azure Functions は「イベントが起きたときだけコードを実行する」シンプルな仕組みに見えますが、トリガー・バインディングの豊富さと周辺サービスとの連携により、幅広い用途に対応できます。

**押さえておくポイント**:

1. **Function App** が管理単位、**Function** が実行単位
2. **トリガー**（起動イベント）と**バインディング**（外部サービスへの宣言的接続）が核心
3. **Flex Consumption プランが現在の推奨**。Consumption プランはレガシー扱い
4. 長時間・ステートフルなワークフローは **Durable Functions** へ
5. コードを書かずに繋ぎたいなら **Logic Apps** が向く
6. Application Insights と統合しておくと、トラブル時のデバッグが格段に楽になる

**次のステップ**:

- [Azure Functions のクイックスタート | Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-functions/functions-get-started)
- [Durable Functions の概要 | Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-functions/durable-functions/durable-functions-overview)

---

## 参考資料

- [What is Azure Functions? | Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-functions/functions-overview)
- [Azure Functions のスケールとホスティング | Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale)
- [Azure Functions のトリガーとバインド | Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-functions/functions-triggers-bindings)
- [Integration and Automation Platform Options in Azure | Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-functions/functions-compare-logic-apps-ms-flow-webjobs)
