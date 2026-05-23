---
title: Azure Durable Functions 入門：サーバーレスで"長い処理"を扱う方法
emoji: 🔄
type: tech
topics: [azure, azurefunctions, serverless, durable, azure初心者]
published: false
---

## はじめに

Azure を使い始めると、すぐに「計算サービス」の多さに戸惑います。

- **App Service** でいいの？
- **Azure Functions** を使うべき？
- **Durable Functions** って何が違うの？

この記事では、この3つの使い分けを整理したうえで、特に「普通の Functions では難しい複雑な処理」を解決する **Azure Durable Functions** に焦点を当てて解説します。

コードの実装詳細よりも「概念と使いどころ」の理解を優先するので、Azure 初学者の方でも読み進められる内容にしています。

---

## Azure の「計算サービス」を整理する

まず全体像を把握しておきましょう。

```
┌─────────────────────────────────────────────────┐
│              Azure App Service                  │
│  ・Web アプリ / API を常時稼働でホスト            │
│  ・ステートフル・常時課金                          │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│              Azure Functions                    │
│  ・イベント駆動・ステートレス・短命               │
│  ・従量課金（実行した分だけ）                     │
│                                                 │
│  ┌───────────────────────────────────────────┐  │
│  │        Azure Durable Functions            │  │
│  │  ・Functions の拡張機能                   │  │
│  │  ・ステートフルなワークフローを実現         │  │
│  └───────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

Durable Functions は、Azure Functions の上に乗る拡張機能です。まず App Service と Azure Functions の違いを押さえてから、Durable Functions に進みましょう。

---

## Azure App Service とは

> 「インフラを気にせず、Web アプリ・API を常時稼働でホストするためのプラットフォーム」

App Service は、Web アプリケーション・REST API・モバイルバックエンドを動かすためのマネージドサービスです。IIS や nginx のセットアップ、OS のパッチ適用、スケールアップの設定といった「インフラ管理」を Azure が肩代わりしてくれます。

### 主な特徴

| 項目 | 内容 |
|---|---|
| 稼働モデル | **常時稼働**（リクエストがなくても動き続ける） |
| 課金モデル | プランに応じた**固定課金**（使っていても使っていなくても発生） |
| サポート言語 | .NET、Java、Node.js、Python、PHP、カスタムコンテナ |
| OS | Windows・Linux 両対応 |
| スケール | 手動または自動スケールアウト |
| CI/CD | GitHub Actions・Azure Pipelines との統合 |
| その他 | カスタムドメイン・SSL・ステージングスロット |

### 向いているケース

- 一般的な Web サイト・EC サイト・管理画面
- 常時稼働が必要な REST API
- フレームワーク（Django、Spring Boot、ASP.NET など）で構築したアプリ

:::message
App Service は「サーバーレス」ではありません。VM を抽象化しているだけで、裏では計算リソースが常に確保されています。
:::

---

## Azure Functions とは（Durable Functions の土台）

> 「イベントが来たときだけ起動し、処理が終わったら終了するステートレスな関数実行環境」

Azure Functions は、**イベント駆動**の計算サービスです。HTTP リクエスト・キューへのメッセージ追加・タイマーなど、さまざまなトリガーに反応して関数が実行され、処理が終わると終了します。

### 主な特徴

| 項目 | 内容 |
|---|---|
| 稼働モデル | **イベント駆動**（トリガーが来たときだけ起動） |
| 課金モデル | **従量課金**（実行回数・実行時間・メモリ使用量） |
| 状態 | **ステートレス**（関数実行間で状態を保持しない） |
| タイムアウト | HTTP トリガーは最大 230 秒（Consumption プラン） |
| スケール | トリガーの量に応じて自動スケールアウト |

### 向いているケース

- 単発の API エンドポイント
- キュー・Blob・イベントに反応する処理
- 定期バッチ（毎時・毎日など）
- Webhook 受信・通知送信

### 向いていないケース

- 複数ステップをまたぐ処理（A が終わってから B を実行、など）
- 処理の途中で外部の応答を長時間待つ（承認待ち・外部 API のポーリングなど）
- 関数間で状態を引き継ぐ必要がある処理

この「向いていないケース」を解決するのが **Durable Functions** です。

---

## Azure Durable Functions とは

> 「Azure Functions の拡張機能。複数ステップ・長時間・状態管理が必要なワークフローを、関数で書けるようにする」

Durable Functions は Azure Functions に追加できる **extension（拡張機能）** として提供されています。通常の Functions ではできない「ステートフルなワークフロー」を、コードで記述できるようになります。

開発者がコードを書くのは変わりませんが、**状態の保存・チェックポイント・再開**を Durable Functions が自動で管理します。

### 構成要素

Durable Functions アプリは、以下の4種類の関数で構成されます。

| コンポーネント | 役割 | 例 |
|---|---|---|
| **Orchestrator（オーケストレーター）** | ワークフロー全体を制御する指揮者 | 「A → B → C の順で呼び出す」を定義 |
| **Activity（アクティビティ）** | 実際の処理を行う最小単位 | DB 書き込み・外部 API 呼び出し・メール送信 |
| **Entity（エンティティ）** | 小さな状態を長期間保持するオブジェクト | カウンター・フラグ・ショッピングカート |
| **Client（クライアント）** | オーケストレーションを開始・管理する | HTTP トリガーでオーケストレーションを起動 |

### 内部の仕組み（イメージ）

```
[Client] ──起動──> [Orchestrator]
                        │
              ┌─────────┼─────────┐
              ▼         ▼         ▼
          [Activity A] [Activity B] [Activity C]
              │
              └── 結果を Orchestrator に返す
                        │
                    次のステップへ...
```

オーケストレーターが状態をチェックポイントとして Azure Storage に自動保存するため、VM が再起動しても、プロセスがクラッシュしても、処理を途中から再開できます。

:::message
**重要な制約**：オーケストレーター関数は**決定論的**に書く必要があります。つまり、同じ入力に対して必ず同じ動作をしなければなりません。`DateTime.Now` や乱数・外部 API 呼び出しをオーケストレーター内に直接書くのは NG で、必ず Activity 関数に切り出します。
:::

---

## 使う場合・使わない場合

「普通の Azure Functions で足りるか？」を判断する3つの軸があります。

| 判断軸 | 普通の Functions | Durable Functions |
|---|---|---|
| **処理時間** | 数秒〜数分（最大 230 秒） | 数分〜数日・数ヶ月 |
| **状態管理** | 不要（ステートレス） | 必要（状態をまたぐ） |
| **ステップ数** | 1 ステップで完結 | 複数ステップを順番に |
| **人の承認待ち** | 困難（タイムアウトする） | 得意（無期限で待機可能） |
| **並列＋集約** | 自前で実装が大変 | パターンとして標準サポート |

### Durable Functions を使うべきケース

- **長時間かかる処理**：外部バッチジョブの完了待ち、数時間・数日単位の処理
- **複数ステップのワークフロー**：データ取得 → 変換 → 保存、のような順番が大事な処理
- **人の承認・外部イベント待ち**：承認フロー、SMS 認証コード検証
- **並列処理の調整**：複数の処理を並列実行して、全部終わったら次に進む

### 普通の Functions で十分なケース

- **単発の短い処理**：API レスポンス、通知送信、単純なデータ変換
- **ステートレスで完結する処理**：関数実行間で状態を引き継がない
- **シンプルなイベント処理**：キューのメッセージを読んで DB に書くだけ

---

## 主要パターン（シナリオ）5種

Durable Functions には代表的な5つのアプリケーションパターンがあります。それぞれ「普通の Functions では難しい処理」を解決します。

### 1. Function Chaining（関数の連鎖実行）

**概要**：A → B → C の順に実行し、前の出力を次の入力にする

```
[Activity A] → [Activity B] → [Activity C] → 完了
```

**具体例**：
- 注文データを取得（A）→ 在庫チェック（B）→ 決済処理（C）→ 確認メール送信（D）

普通の Functions でも実装できますが、どこで失敗したか追跡・再実行するのが大変です。Durable Functions なら各ステップのチェックポイントが自動で保存されます。

参考：https://learn.microsoft.com/en-us/azure/durable-task/common/durable-task-sequence

---

### 2. Fan-out / Fan-in（並列実行して集約）

**概要**：複数の処理を並列に実行し、全部終わったら結果をまとめて次に進む

```
                 ┌→ [Activity A1] ──┐
[Orchestrator] ──┼→ [Activity A2] ──┼→ 集約 → 次の処理
                 └→ [Activity A3] ──┘
```

**具体例**：
- 数百件の商品データを並列処理 → すべて完了したらレポート生成
- 複数の外部 API を同時呼び出し → 結果をまとめてレスポンス

Durable Functions の Orchestrator が全タスクの完了を追跡します。1つが失敗しても他の処理に影響しません。

参考：https://learn.microsoft.com/en-us/azure/durable-task

---

### 3. Human Interaction（人の承認待ち）

**概要**：処理を一時停止して、人の入力・承認を待ってから続きを実行する

```
[承認申請] → [待機...数時間〜数日] → [承認/却下を受信] → [次の処理]
```

**具体例**：
- 経費申請フロー（上長が承認するまで無期限で待機）
- SMS 認証コードの検証（ユーザーが入力するまで待機）

:::message
**待機中は compute コストが発生しません。** オーケストレーターはストレージに状態を保存して「休眠」するため、承認待ちで数日待っても課金されない点が重要です。
:::

普通の Functions だとタイムアウトしてしまうため、DB に状態を別途保存・管理する複雑な設計が必要になります。Durable Functions はその複雑さをフレームワーク側で吸収します。

参考：https://learn.microsoft.com/en-us/azure/durable-task/common/durable-task-human-interaction

---

### 4. Monitor（ポーリング監視）

**概要**：条件が満たされるまで定期的に状態をチェックし続ける長期実行プロセス

```
[チェック] → 未完了 → [待機] → [チェック] → 未完了 → ... → 完了
```

**具体例**：
- 外部バッチジョブの完了を 5 分おきに確認
- 在庫が一定数を下回ったら通知

Timer トリガーを使えば定期実行できますが、「完了したら止まる」「動的に間隔を変える」といった柔軟な制御は Durable Functions の方が容易です。

参考：https://learn.microsoft.com/en-us/azure/durable-task

---

### 5. Stateful Entities（ステートフルエンティティ）

**概要**：長期間にわたって小さな状態を保持・操作するオブジェクト（Actor パターン）

```
[Client] → カウントアップ → [Entity: Counter=5]
[Client] → カウントアップ → [Entity: Counter=6]
[Client] → 現在値を取得  → 6
```

**具体例**：
- アクセスカウンター
- ユーザーセッション状態の管理
- ショッピングカートの中身

Redis のようなキャッシュや DB を使わず、Durable Functions 内で状態を保持できます。

参考：https://learn.microsoft.com/en-us/azure/durable-task/common/programming-model-overview

---

## 一緒に使われるサービス

### 必須：Azure Storage

Durable Functions は内部的に **Azure Storage（Queue・Blob・Table）** を使って状態・チェックポイント・メッセージを管理します。これは切り離せません。

:::message alert
**パフォーマンス上の注意**：Durable Functions と Event Hubs トリガーの関数は、ストレージトランザクションを大量に発生させます。Function App ごとに専用の Storage Account を使うことが推奨されています。
:::

### 選択肢：Durable Task Scheduler（DTS）

Azure Storage の代わりに使えるマネージド型のバックエンドサービスです。Storage を自分で管理するより運用が楽になります（2025 年以降に登場した比較的新しい選択肢）。

### 任意：組み合わせると便利なサービス

| サービス | 役割・使い方 |
|---|---|
| **Azure Service Bus** | より高信頼・高スループットのメッセージキュー。Storage Queue より大きいメッセージ（最大 100 MB）を扱える |
| **Azure Event Grid** | オーケストレーションの開始・完了・失敗などのイベントを通知先（Webhook・Logic Apps など）に配信できる |
| **Azure Logic Apps** | ノーコード・ローコードのワークフロー。Durable Functions（コードで書く）と Logic Apps（GUI で書く）は目的が重なる部分もあり、用途に応じて使い分けまたは併用する |
| **Application Insights** | 実行ログ・パフォーマンス監視・エラートレース。本番環境では必須レベルで導入を推奨 |

---

## まとめ

3つのサービスの使い分けを一言でまとめると：

| サービス | 一言まとめ |
|---|---|
| **Azure App Service** | 常時稼働の Web アプリ・API をホストしたい |
| **Azure Functions** | イベントが来たときだけ動く短命・ステートレスな処理を書きたい |
| **Azure Durable Functions** | 長時間・複数ステップ・状態管理が必要なワークフローを Functions で書きたい |

「普通の Functions だと、タイムアウトする・状態を保てない・複数ステップの追跡が難しい」と感じたとき、Durable Functions の出番です。

### 次のステップ

まずは **Function Chaining** パターンから試してみるのがおすすめです。小さな順番処理を書いてみると、Durable Functions の「チェックポイント・再開」の仕組みを体感できます。

- [クイックスタート（C#）](https://learn.microsoft.com/en-us/azure/azure-functions/durable-functions/durable-functions-isolated-create-first-csharp)
- [クイックスタート（Python）](https://learn.microsoft.com/en-us/azure/azure-functions/durable-functions/quickstart-python-vscode)
- [クイックスタート（JavaScript）](https://learn.microsoft.com/en-us/azure/azure-functions/durable-functions/quickstart-js-vscode)
- [Durable Functions の公式概要](https://learn.microsoft.com/en-us/azure/azure-functions/durable-functions/durable-functions-overview)
