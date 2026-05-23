---
title: "Azure App Service 入門 — 初学者のための概要・使い分け・関連サービス完全ガイド"
emoji: "🌐"
type: "tech"
topics: ["azure", "appservice", "paas", "architecture", "webapp"]
published: false
---

# はじめに

「Azure で Web アプリを動かしたい」と思ったとき、候補として真っ先に名前が上がるサービスの一つが **Azure App Service** です。

しかし Azure には似た選択肢がいくつかあります。Container Apps、Azure Functions、AKS（Azure Kubernetes Service）……それぞれ何が違うのか、App Service はいつ使えばいいのか、迷う方も多いのではないでしょうか。

この記事では、Azure App Service を **初学者向けに** 以下の観点で整理します。

- App Service とは何か（概念と主な機能）
- 料金プランの種類
- どんなシナリオで使うべきか／使わないべきか
- 一緒によく使われる Azure サービスの紹介

:::message
この記事は **概念理解と使い分けの判断** に焦点を当てています。ポータルでの詳細設定手順やコードサンプルは扱いません。
:::

---

# 1. Azure App Service とは何か

## 一言で言うと

**インフラの管理なしに、Web アプリ・API・モバイルバックエンドを動かせる PaaS（Platform as a Service）です。**

従来の Web アプリの運用では「サーバーを立てる → OS をセットアップする → Nginx や Apache を設定する → スケールするために台数を増やす」といった作業が必要でした。App Service はこれらの **「インフラ管理の手間」を丸ごと Azure 側が引き受けてくれます。**

開発者は「コードを書いてデプロイする」ことに集中できます。

## 対応スタックと OS

App Service は幅広い言語・フレームワークに対応しています。

| 言語 | 対応フレーバー |
|------|-------------|
| .NET | ASP.NET Core など |
| Java | Java SE / Tomcat / JBoss |
| Node.js | Express, NestJS など |
| Python | Flask, FastAPI, Django など |
| PHP | Laravel など |
| カスタムコンテナ | Docker イメージ（Windows / Linux） |

OS は **Windows と Linux** の両方に対応しており、既存のスタックをそのまま持ち込みやすい設計になっています。

## App Service が「やってくれること」

App Service を使うと、以下のことを自分で管理しなくて済みます。

- **OS のパッチ適用・セキュリティ更新**: 自動で行われる
- **ロードバランサー**: 複数インスタンスへの分散が組み込み済み
- **TLS 証明書の自動更新**: 無料のマネージド証明書を使える
- **スケーリング**: 負荷に応じてインスタンスを増減（自動スケール）
- **CI/CD 連携**: GitHub Actions や Azure Pipelines との統合が標準

## App Service Plan（プラン）とは

App Service を使う際、必ず **App Service Plan（プラン）** を作成します。プランとアプリの関係は次のようなイメージです。

```
App Service Plan（プラン）
  = 「VM リソースの入れ物」

  ├── Web アプリ A（App Service）
  ├── Web アプリ B（App Service）
  └── Web アプリ C（App Service）
```

プランは「どのリージョンに」「どのサイズの VM を」「何台用意するか」を定義します。**同じプラン内の複数アプリはリソースを共有します。** 

コストを抑えたい場合は、1 つのプランに複数のアプリを相乗りさせることができます。一方、アプリごとに独立したリソースが必要なら、プランを分けます。

:::message
プランの料金はアプリの数に関係なく、プランのサイズ（SKU）と稼働時間で決まります。アプリが 1 つでも 10 個でも同じプランなら料金は同じです。
:::

---

# 2. 料金プランの概要

App Service の料金は **App Service Plan の「価格レベル（ティア）」** によって決まります。大きく 5 つのカテゴリに分かれています。

## 価格ティア一覧

| カテゴリ | 代表 SKU | 主な用途 | スケールアウト | SLA |
|----------|----------|----------|--------------|-----|
| **Free / Shared** | F1 / D1 | 学習・PoC のみ | ✗ | なし |
| **Basic** | B1〜B3 | 開発・テスト環境 | 手動（最大 3 台） | あり |
| **Standard** | S1〜S3 | 本番環境の標準 | 自動（最大 10 台） | あり |
| **Premium (v2/v3/v4)** | P1V3〜P5mV4 | 高パフォーマンス・高密度 | 自動（最大 30 台） | あり |
| **Isolated v2** | I1V2〜I6V2 | 完全ネットワーク隔離・コンプライアンス対応 | 自動 | あり |

## 選ぶポイント

**学習や PoC（概念実証）**: Free（F1）で十分。ただし SLA なし・CPU に制限あり。

**本番環境**: Standard（S1）以上を推奨。理由は以下の機能が Standard から利用可能になるため。
- カスタムドメイン + SSL（Basic から利用可）
- **自動スケール**（Standard から）
- **デプロイスロット**（ステージング環境）（Standard から）
- VNet 統合（Standard から）

**高い処理性能が必要・コスト最適化したい**: Premium v3 / v4 ティアへ。メモリ最適化オプション（`Pmv3` / `Pmv4`）や Azure 予約インスタンスを組み合わせると、Standard より総コストが下がるケースもあります。

**コンプライアンスや厳格なネットワーク分離が必要**: Isolated v2（App Service Environment v3）。他の Azure 顧客とインフラを共有しない完全専用環境が手に入りますが、その分コストが高くなります。

:::message
「とりあえず本番で App Service を試したい」なら **Standard S1** からスタートして、負荷を見てスケールアップ・スケールアウトを検討するのが定番の進め方です。
:::

---

# 3. Azure App Service を使うべきシナリオ

App Service が特に力を発揮するのは、次のようなシナリオです。

## シナリオ 1: シンプルな Web アプリ・REST API

Python（FastAPI / Flask）や Node.js（Express）、.NET（ASP.NET Core）など、**対応スタックで書いた HTTP サーバー** を動かすなら App Service が最も手軽です。コードを Git リポジトリにプッシュするだけでデプロイが完了するワークフローも実現できます。

## シナリオ 2: モバイルアプリのバックエンド

iOS / Android アプリが呼び出す REST API のバックエンドとして App Service はよく使われます。自動スケールにより、ユーザーが増えても対応できます。

## シナリオ 3: 社内向け基幹アプリ（LoB アプリ）

Microsoft Entra ID（旧 Azure AD）との認証統合や VNet 統合による内部ネットワークへのアクセスなど、**企業内アプリに必要な機能が標準で組み込まれています。** 認証の実装コストを大きく削減できます。

## シナリオ 4: WordPress などの OSS Web アプリ

Azure Marketplace のテンプレートを使って WordPress や Joomla、Drupal などを即座にデプロイできます。CMS の運用をインフラ管理なしで行いたい場合に向いています。

## シナリオ 5: CI/CD を組み込んだチーム開発

GitHub Actions や Azure DevOps との連携が標準でサポートされています。プルリクエストのマージをトリガーに自動デプロイし、**デプロイスロット**（ステージング環境）を使ってブルーグリーンデプロイを実現する構成も組みやすいです。

---

# 4. Azure App Service を使わないほうが良いシナリオ

App Service が万能というわけではありません。要件によっては別のサービスが適しているケースがあります。

## 比較マトリクス

| 要件 | 推奨サービス | 理由 |
|------|------------|------|
| シンプルな HTTP サービス | **App Service** | 手軽、インフラ不要 |
| 処理が散発的、動いていない時間に課金したくない | **Azure Functions** | 実行時間分だけ課金、スケールゼロ対応 |
| マイクロサービスで Dapr やサービスディスカバリが必要 | **Azure Container Apps** | 組み込みのマイクロサービス基盤 |
| Kubernetes を直接操作したい / 細かいクラスタ制御が必要 | **AKS** | Kubernetes API への直接アクセス |
| コンプライアンスで完全なネットワーク隔離が必要 | **App Service（Isolated/ASE）** | 専用 VNet 内での App Service 運用 |

## 具体的に「App Service ではなく別サービスを選ぶ」判断軸

**Azure Functions を選ぶとき**
- 処理がイベント駆動（メッセージキューのトリガー、Timer など）
- トラフィックが予測不能で、閑散時に課金をゼロにしたい
- 処理が短時間（数秒〜数分）で完結する

**Azure Container Apps を選ぶとき**
- 複数のマイクロサービスをコンテナ単位で独立してスケールさせたい
- Dapr を使ったサービス間通信やステート管理が必要
- スケールゼロが必要だが、Functions のようなトリガーベースではなくコンテナで動かしたい

**AKS を選ぶとき**
- Helm チャートや Istio など Kubernetes エコシステムをそのまま使いたい
- ノードプールの設定、ネットワークポリシーなど細粒度のクラスタ制御が必要
- すでに Kubernetes の運用ノウハウがあるチーム

:::message
**初学者向けの判断ショートカット**:
「コードを書いて HTTP で公開したい、難しいことはしたくない」→ **App Service**
「処理が散発的でコストを抑えたい」→ **Azure Functions**
「コンテナでマイクロサービスを動かしたい」→ **Container Apps**
「Kubernetes を使いたい」→ **AKS**
:::

---

# 5. App Service と一緒によく使われる Azure サービス

App Service 単体で完結するアプリは少なく、実際の構成では複数の Azure サービスと組み合わせます。初学者が知っておくべき定番の組み合わせを紹介します。

## カテゴリ別 関連サービス一覧

### データ・ストレージ

| サービス | 役割 |
|----------|------|
| **Azure SQL Database** | マネージドな SQL Server。App Service からの接続に最もよく使われるリレーショナル DB |
| **Azure Cosmos DB** | グローバル分散に対応した NoSQL DB。JSON ドキュメント形式 |
| **Azure Blob Storage** | 画像・PDF・動画などのファイル保存。App Service からアップロードされたファイルの格納先としてよく使われる |
| **Azure Cache for Redis** | セッション管理・クエリ結果のキャッシュに使うインメモリデータストア |

### セキュリティ・シークレット管理

| サービス | 役割 |
|----------|------|
| **Azure Key Vault** | DB 接続文字列・API キーなどの機密情報を安全に管理。App Service のアプリ設定から Key Vault 参照として読み込める（コードに機密情報を書かずに済む） |
| **Microsoft Entra ID（旧 Azure AD）** | 社内ユーザーの認証・SSO。App Service の「認証」機能から数クリックで設定可能 |
| **Azure AD B2C** | 外部ユーザー向けの認証（メール登録・Google/Facebook ログインなど）に使うカスタマー向け IAM |

### 監視・ログ

| サービス | 役割 |
|----------|------|
| **Application Insights** | リクエストのトレース・エラー検知・パフォーマンス計測。App Service のアプリ設定に接続文字列を追加するだけで有効化できる |
| **Azure Monitor** | メトリクス・ログの集約・アラート設定。App Service のオートスケールトリガーにも使われる |

### グローバル配信・ネットワーク

| サービス | 役割 |
|----------|------|
| **Azure Front Door** | グローバルな HTTP ロードバランサー＋WAF（Web Application Firewall）。複数リージョンの App Service をまとめて前段に置く構成でよく使われる |
| **Azure CDN** | 静的コンテンツのエッジキャッシュ配信 |
| **Azure Virtual Network** | VNet 統合で App Service からプライベートネットワーク内のリソース（DB など）に安全にアクセス |

### CI/CD・コンテナ

| サービス | 役割 |
|----------|------|
| **GitHub Actions** | App Service へのデプロイを自動化。App Service ポータルから Workflow ファイルを自動生成できる |
| **Azure DevOps** | CI/CD パイプラインの管理。Azure Pipelines で App Service へのデプロイが標準サポート |
| **Azure Container Registry (ACR)** | カスタムコンテナイメージの管理・プライベートレジストリ。コンテナで App Service を動かす場合の定番 |

## 初学者向けの典型的な最小構成例

```
                            ┌─────────────────────────┐
[ユーザー]  ──HTTPS──>  │  Azure App Service       │
                            │  （Web アプリ / API）    │
                            └────────┬────────────────┘
                                     │
              ┌──────────────────────┼────────────────────┐
              ▼                      ▼                    ▼
   ┌─────────────────┐  ┌─────────────────────┐  ┌───────────────────┐
   │ Azure SQL DB     │  │ Application Insights │  │  Azure Key Vault  │
   │ （データ永続化）  │  │ （監視・トレース）    │  │  （接続文字列管理） │
   └─────────────────┘  └─────────────────────┘  └───────────────────┘
```

この最小構成から始めて、必要に応じて Blob Storage（ファイル保存）、Azure Cache for Redis（キャッシュ）、Front Door（グローバル配信）を足していくのが一般的な拡張パスです。

## Key Vault 参照 — 接続文字列をコードに書かない方法

App Service では、アプリ設定の値として Key Vault の秘密情報を参照する **「Key Vault 参照」** 機能があります。

アプリ設定に以下のような形式で設定するだけで、コード上は通常の環境変数として読めます。

```
@Microsoft.KeyVault(SecretUri=https://my-vault.vault.azure.net/secrets/db-connection)
```

DB の接続文字列や API キーをコードやアプリ設定に直書きしなくて済むため、セキュリティのベストプラクティスとして標準的に使われます。

---

# 6. まとめ

Azure App Service は **「Web アプリを最速で、インフラを気にせず動かす」** ための強力な PaaS です。

本記事で解説した選択の判断軸を整理すると、次のようになります。

| 判断軸 | App Service を選ぶとき |
|--------|----------------------|
| アプリの形態 | HTTP ベースの Web アプリ・REST API |
| インフラ管理 | 自分で管理したくない |
| スタック | .NET / Java / Python / Node.js / PHP |
| スケール | リクエスト数に応じた自動スケールが必要 |
| コスト | 常時稼働前提のアプリ（サーバーレスより安くなるケースが多い） |

## 次のステップ

App Service の概念をつかんだら、次は実際に手を動かしてみましょう。

- [クイックスタート: ASP.NET Core Web アプリを作成する](https://learn.microsoft.com/ja-jp/azure/app-service/quickstart-dotnetcore)
- [クイックスタート: Python アプリをデプロイする](https://learn.microsoft.com/ja-jp/azure/app-service/quickstart-python)
- [クイックスタート: Node.js Web アプリを作成する](https://learn.microsoft.com/ja-jp/azure/app-service/quickstart-nodejs)
- [チュートリアル: ASP.NET Core + Azure SQL + Key Vault 構成](https://learn.microsoft.com/en-us/azure/app-service/tutorial-dotnetcore-sqldb-app)

---

## 参考リンク

- [Azure App Service の概要（公式）](https://learn.microsoft.com/ja-jp/azure/app-service/overview)
- [App Service Plan の概要（公式）](https://learn.microsoft.com/en-us/azure/app-service/overview-hosting-plans)
- [Azure コンピュートオプションの選択ガイド](https://learn.microsoft.com/en-us/azure/architecture/microservices/design/compute-options)
- [Azure コンテナサービスの選択ガイド](https://learn.microsoft.com/en-us/azure/architecture/guide/choose-azure-container-service)
- [Key Vault 参照の使い方](https://learn.microsoft.com/en-us/azure/app-service/app-service-key-vault-references)
- [App Service の監視（Application Insights 統合）](https://learn.microsoft.com/en-us/azure/app-service/monitor-app-service)
