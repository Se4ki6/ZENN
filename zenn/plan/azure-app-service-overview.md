---
title: Azure App Service 入門 — 初学者のための概要・使い分け・関連サービス完全ガイド
status: plan
---

## 記事タイトル候補

1. 「Azure App Service とは何か — Web アプリを最速で動かす PaaS の全体像」
2. 「初学者のための Azure App Service 入門 — 概要・使い分け・組み合わせサービスを解説」
3. 「Azure App Service 入門: いつ使う？何と組み合わせる？初学者向け選択ガイド」
4. 「Azure の Web ホスティングを理解する — App Service を中心に使い分けを整理する」

## 想定読者

- Azure を触り始めたばかりの開発者
- Web アプリを Azure にデプロイしたいが、どのサービスを選べばいいか迷っている人
- AWS や GCP など他クラウドの経験はあるが、Azure の Web ホスティングを把握したい人
- 前提知識: Web アプリ（HTTP サーバー）の基本的な概念を理解している

## 記事で答える問い

「Azure App Service とは何で、いつ使うべきで、何と組み合わせれば良いのか？」

## 扱う範囲・扱わない範囲

**扱う**
- Azure App Service の概念・主な機能
- 料金プラン（Free〜Isolated）の概要
- 使うべきシナリオと使わないシナリオ（Container Apps / AKS / Functions との比較）
- よく一緒に使われる関連 Azure サービスの紹介

**扱わない**
- ポータル・CLI での詳細な設定手順
- コードサンプル（デプロイ手順の詳細）
- App Service Environment (ASE) の詳細設計
- IaC（Terraform / Bicep）での管理方法

---

## セクション構成

### 0. はじめに（導入）

**主張**: 「Azure で Web アプリを動かしたい」と思ったとき、最初に候補に上がる PaaS が App Service。でも選択肢が多くて迷う人が多い。この記事で全体像を整理する。

**根拠 URL**: なし（導入文）

---

### 1. Azure App Service とは何か

**主張**: Azure App Service は、Web アプリ・REST API・モバイルバックエンドを「インフラ管理不要」で動かせる PaaS（Platform as a Service）。対応スタックが広く、OS も Windows/Linux の両方に対応。

**主なポイント**:
- 定義: マネージドな Web アプリホスティング基盤
- 対応スタック: .NET / Java (SE, Tomcat, JBoss) / Node.js / Python / PHP / カスタムコンテナ
- 対応 OS: Windows / Linux
- App Service が「面倒を見てくれること」: OS パッチ / スケーリング / ロードバランサー / TLS 証明書管理
- App Service Plan（プラン）の概念: リソースの「入れ物」として複数アプリを相乗り可能

**根拠 URL**:
- https://learn.microsoft.com/en-us/azure/app-service/overview
- https://learn.microsoft.com/en-us/azure/app-service/overview-hosting-plans

---

### 2. 料金プランの概要（Free〜Isolated）

**主張**: App Service には Free から Isolated まで 6 カテゴリの料金ティアがあり、開発用途か本番用途かで選択肢が絞られる。まず「本番で使えるティア」の基準を知ることが重要。

**主なポイント**:

| カテゴリ | 代表 SKU | 用途の目安 |
|----------|----------|------------|
| Free / Shared | F1 / D1 | 学習・PoC のみ（SLA なし） |
| Basic | B1〜B3 | 開発・テスト環境 |
| Standard | S1〜S3 | 本番環境の標準 |
| Premium (v2/v3/v4) | P1V3〜P5mV4 | 高パフォーマンス・高密度 |
| Isolated v2 | I1V2〜I6V2 | ネットワーク完全隔離・コンプライアンス要件 |

- カスタムドメイン・SSL は Basic 以上
- 自動スケール（スケールアウト）は Standard 以上
- VNet 統合（Regional VNet Integration）は Standard 以上

**根拠 URL**:
- https://learn.microsoft.com/en-us/azure/app-service/overview-hosting-plans
- https://learn.microsoft.com/en-us/azure/app-service/app-service-configure-premium-v4-tier

---

### 3. Azure App Service を使うべきシナリオ

**主張**: 「コードを書いて、動かす」にフォーカスしたいとき、App Service は最も手軽な選択肢になる。下記のシナリオに当てはまるなら積極的に検討できる。

**主なシナリオ**:
1. **シンプルな Web アプリ・API**: Flask / FastAPI / Express / ASP.NET Core など、既存スタックで動く HTTP サービス
2. **モバイルバックエンド**: iOS/Android から呼ばれる REST API
3. **社内基幹アプリ（LoB アプリ）**: 認証（Azure AD 統合）や VNet 接続が必要な内部向け Web アプリ
4. **CMS・WordPressなどの OSS Web アプリ**: Azure Marketplace テンプレートで即座に起動
5. **CI/CD でデプロイを自動化したい**: GitHub Actions / Azure DevOps との連携が組み込み対応

**根拠 URL**:
- https://learn.microsoft.com/en-us/azure/app-service/overview
- https://learn.microsoft.com/en-us/azure/architecture/microservices/design/compute-options

---

### 4. Azure App Service を使わないほうが良いシナリオ（他サービスとの使い分け）

**主張**: App Service が万能というわけではない。要件によっては Container Apps / Azure Functions / AKS のほうが適しているケースがある。初学者が迷いやすい比較軸を整理する。

**比較マトリクス（概要）**:

| 要件 | 推奨サービス | 理由 |
|------|------------|------|
| HTTP ベースの普通の Web アプリ | **App Service** | 最も手軽、インフラ管理不要 |
| イベント駆動・散発的な処理 | **Azure Functions** | 実行時間分だけ課金、スケールゼロ |
| コンテナで Dapr・サービスディスカバリが必要 | **Container Apps** | マイクロサービス向けの組み込み機能 |
| Kubernetes クラスタへの直接アクセスが必要 | **AKS** | ノードプール・ネットワークポリシーの完全制御 |
| コンプライアンスで完全隔離ネットワークが必要 | **App Service（Isolated/ASE）** | 専用 VNet 内での App Service 運用 |

**App Service を選ばないべきケース**:
- 処理が散発的で「動いていない時間に課金したくない」→ Functions
- Kubernetes のエコシステム（Helm, Istio など）をそのまま使いたい → AKS
- 複数のマイクロサービスをコンテナ単位でスケールしたい → Container Apps

**根拠 URL**:
- https://learn.microsoft.com/en-us/azure/architecture/microservices/design/compute-options
- https://learn.microsoft.com/en-us/azure/architecture/guide/choose-azure-container-service
- https://learn.microsoft.com/en-us/answers/questions/1337789/azure-app-service-vs-azure-container-apps-which-to

---

### 5. 一緒に使われる主な Azure サービス

**主張**: App Service 単体で完結するアプリは少なく、実際の構成ではいくつかの Azure サービスと組み合わせる。初学者が知っておくべき定番の組み合わせを紹介する。

**カテゴリ別 関連サービス**:

| カテゴリ | サービス | 役割 |
|----------|----------|------|
| データベース | Azure SQL Database / Azure Cosmos DB | リレーショナル or NoSQL データ永続化 |
| シークレット管理 | Azure Key Vault | 接続文字列・API キーを安全に管理 |
| 監視・ログ | Application Insights / Azure Monitor | リクエスト追跡・エラー検知・パフォーマンス計測 |
| ストレージ | Azure Blob Storage | 画像・ファイルの保存 |
| グローバル配信 | Azure Front Door / Azure CDN | エッジからの高速配信・WAF |
| 認証 | Microsoft Entra ID (旧 Azure AD) / Azure AD B2C | ユーザー認証・SSO |
| キャッシュ | Azure Cache for Redis | セッション管理・高速キャッシュ |
| CI/CD | GitHub Actions / Azure DevOps | デプロイ自動化 |
| コンテナレジストリ | Azure Container Registry | カスタムコンテナイメージの管理 |
| プライベートネットワーク | Azure Virtual Network | VNet 統合でバックエンドを隔離 |

**初学者向けの典型的な最小構成例**:
```
[ユーザー] → [Azure App Service] → [Azure SQL Database]
                    ↓
             [Application Insights]（監視）
             [Azure Key Vault]（接続文字列）
```

**根拠 URL**: 追加調査が必要（各サービスの公式統合ドキュメント）

---

### 6. まとめ・次のステップ

**主張**: App Service は「インフラを気にせず Web アプリを動かす」ための強力な出発点。使い分けの判断軸を持った上で、次は実際に動かしてみることで理解が深まる。

**次のステップ候補（リンク案）**:
- App Service クイックスタート（各言語別）
- Azure Developer CLI (`azd`) での最初のデプロイ
- 関連記事リンク（本リポジトリ内の Azure 関連記事）

---

## 不足情報・追加調査ポイント

本格執筆前に以下を追加調査する:

1. **関連サービスの統合ドキュメント**: App Service と Key Vault / Application Insights / Azure SQL の接続方法の公式 URL
   - 候補: `learn.microsoft.com/azure/app-service/tutorial-connect-msi-sql-database` など
2. **料金の具体的数値**: 執筆時点の各ティア価格（変動するため公式価格ページへのリンクで対応）
3. **App Service の SLA 数値**: 現在の SLA（99.95% など）の公式確認

## 概要提案の根拠ファイル

- `temp/azure_app_service/search_app_service_overview.json`
- `temp/azure_app_service/search_app_service_vs_alternatives.json`

## 主要参照 URL

| URL | 用途 |
|-----|------|
| https://learn.microsoft.com/en-us/azure/app-service/overview | App Service 概要・機能一覧 |
| https://learn.microsoft.com/en-us/azure/app-service/overview-hosting-plans | プラン・料金ティア |
| https://learn.microsoft.com/en-us/azure/app-service/environment/overview | App Service Environment（ASE） |
| https://learn.microsoft.com/en-us/azure/architecture/microservices/design/compute-options | コンピュート選択ガイド |
| https://learn.microsoft.com/en-us/azure/architecture/guide/choose-azure-container-service | コンテナサービス選択ガイド |
