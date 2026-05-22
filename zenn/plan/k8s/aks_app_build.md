---
title: "概念は分かった人のための Azure AKS 実践 — ACR・マネージド PostgreSQL と組んで『K8s が主役』のアプリを最小構成で作る"
status: plan
---

## この記事の位置づけ

前提記事「[Docker は何となく使える人のための Kubernetes 入門](k8s_for_docker_users.md)」の **後続・実践編**。

前提記事のゴールは「概念の地図とマニフェストの読み方」だった。最後に置いた割り切りが本記事の出発点になる:

> ステートレスな部分は K8s に、DB はマネージドサービス（RDS / Cloud SQL など）に逃がす

本記事は **これを Azure 上で実際に手を動かして組む**。読者が `az` / `kubectl` を叩いて、概念として理解した K8s リソースを現実の構成に落とす。

## 設計方針（ユーザー確定事項）

- 題材アプリ: **自作の2層ステートレスアプリ（フロント + API）**。公式 AKS Store サンプルは使わない（RabbitMQ 等が混ざり K8s 学習が薄まるため）
- DB 接続: **Kubernetes Secret 中心**。Workload Identity によるパスワードレスは「発展」セクションで概観のみ
- **K8s が主役**。周辺 Azure サービスは ACR（必須）と マネージド PostgreSQL（ステートフルの外出し）だけ
- POC として自然な省略（むしろ望ましい）: APIM / Redis / Front Door / Key Vault 連携の作り込み / 監視・ログ・CI-CD / VNet 分離・Private Endpoint / DB の HA・バックアップ詳細

## 想定読者・問い

- 想定読者: 前提記事レベルで K8s の概念（Deployment/Service/Ingress/HPA/ConfigMap/Secret、宣言的モデル）は理解したが、クラウドで実際に動かしたことがない人。Azure と一般技術は分かる前提
- 問い: 概念として理解した K8s リソースを Azure AKS 上で実際に組み合わせ、外部マネージド DB に繋がる現実的な Web アプリを最小構成で作るには? そのとき **K8s は何を担い、何を Azure に任せるのか**?

## タグ案

`["kubernetes", "azure", "aks", "container", "devops"]`
（前提記事の `kubernetes` / `container` / `devops` を再利用し、`azure` / `aks` を追加）

## セクション構成

### 1. この記事について
- 前提記事の振り返り（宣言的モデル、用語マップ、最後の割り切り）
- ゴールと前提（Azure サブスクリプション・`az` CLI・`kubectl`・K8s 概念）
- 設計方針: K8s が主役、周辺は ACR とマネージド PostgreSQL だけ。何を省くか
- 主張: 「概念 → 実構成」の橋渡し。手を動かして線引きを体感する
- 根拠: 前提記事 [k8s_for_docker_users.md](../publish/k8s_for_docker_users.md)

### 2. 作るもの — 構成の全体像
- 2層ステートレスアプリ（フロント + API）+ マネージド PostgreSQL の図（前提記事の図と地続き: 青=ステートレス on AKS、橙=マネージド DB）
- どの部品が どの Azure リソース / K8s リソースに対応するか表
- 「K8s が肩代わりする部分」と「Azure マネージドに任せる部分」の線引き
- 主張: 前提記事の最終図を、そのまま Azure の実物に置き換える
- 根拠: 前提記事

### 3. 全体の流れ（俯瞰）
- リソースグループ → ACR → ビルド&プッシュ → AKS（ACR連携）→ マネージド PostgreSQL → `kubectl` デプロイ → 公開 → スケール
- 各ステップが「何のためか」を一言
- 根拠: tutorial シリーズ全体

### 4. ステップ1: イメージの置き場所 — ACR を作ってビルド&プッシュ
- なぜレジストリが要るか（AKS のノード上の containerd が pull できる場所が必要。手元の docker のイメージは知らない）= 前提記事「Docker でビルドし K8s で動かす」の実装
- `az group create` / `az acr create`
- `az acr build`（ACR Tasks: クラウド側でビルド、手元に docker 不要）
- 主張: 「Docker でビルド → K8s で動かす」の "置き場所" を最初に用意する
- 根拠: https://learn.microsoft.com/en-us/azure/aks/tutorial-kubernetes-prepare-acr （extract_build_deploy.json）

### 5. ステップ2: AKS クラスタを作る — そして ACR と繋ぐ
- `az aks create --attach-acr`（kubelet マネージド ID に AcrPull ロールを自動付与＝Pod が認証情報なしで pull できる）
- マネージド ID がデフォルト。サポート対象 K8s バージョン / ノードイメージ（Azure Linux 2.0 退役: 2025/11 更新終了・2026/3 イメージ削除）に注記
- `az aks get-credentials` で `kubectl` を接続
- 「control plane は Azure が運用し、自分は Node と上のワークロードに集中」= 前提記事の control plane / kubelet が誰の持ち物になるか
- 主張: マネージド K8s の意味は「望ましい状態を維持する頭脳を Azure に預ける」こと
- 根拠: https://learn.microsoft.com/en-us/azure/aks/tutorial-kubernetes-deploy-cluster , https://learn.microsoft.com/en-us/azure/aks/cluster-container-registry-integration （extract_build_deploy.json）/ Azure Linux 退役は search_aks_deploy_overview.json

### 6. ステップ3: マネージド DB を用意する — ステートフルを K8s の外へ
- なぜ DB を K8s に載せないか（前提記事の StatefulSet/PV の難所、バックアップ/HA/フェイルオーバー）→ ここで実装として「逃がす」
- `az postgres flexible-server create`（`--tier` / `--sku-name` / `--version` / `--public-access`）
- ネットワーク: public access + ファイアウォール（`0.0.0.0` = Azure サービス許可）。POC簡略。本番は VNet/Private Endpoint と注記
- 接続: `sslmode=require`
- 主張: ステートフルの難所は専用サービスに任せる、を実際に実行する
- 根拠: https://learn.microsoft.com/en-us/azure/postgresql/configure-maintain/quickstart-create-server , https://learn.microsoft.com/en-us/azure/postgresql/security/security-firewall-rules （extract_postgres.json）

### 7. ステップ4: マニフェストを書く — 概念を YAML に落とす（K8s 主役パート）
- ConfigMap / Secret: DB 接続情報を分離（Secret に接続情報、`envFrom` で注入）= 前提記事の用語をそのまま実装
- Deployment（API）: `image` は `<acr>.azurecr.io/...`、liveness/readiness probe、resources（requests/limits）
- Deployment（フロント）: 同じパターン
- Service（ClusterIP）: 内部入口
- 「同じ Deployment + Service パターンを2回使い回す」= 前提記事の「知識を使い回せる」を実演
- 主張: 前提記事の用語マップが、そのまま動く YAML になる
- 根拠: 前提記事 + https://learn.microsoft.com/en-us/azure/aks/load-balancer-standard （search_deploy_app_step.json）

### 8. ステップ5: 外部に公開する — Service(LoadBalancer) と Ingress
- まず素朴に Service `type: LoadBalancer` → Azure Load Balancer が EXTERNAL-IP を払い出す（`kubectl get service --watch`、pending→IP）。前提記事の LoadBalancer タイプの実体
- 次に Ingress: application routing addon を有効化（マネージド NGINX）。`/api` は API、それ以外はフロント、のルーティング = 前提記事の「クラスタ全体の玄関」
- 重要な注記（移行期）: OSS Ingress NGINX のメンテ終了 2026/3、application routing addon は 2026/11 まで本番サポート → Gateway API へ。POC では addon で十分だが将来は Gateway API
- 主張: 「個々の入口（Service）」と「玄関（Ingress）」の役割分担を実物で見る。ただし Ingress 周辺は移行期で、断定を避ける
- 根拠: https://learn.microsoft.com/en-us/azure/aks/tutorial-kubernetes-deploy-application , https://learn.microsoft.com/en-us/azure/aks/load-balancer-standard , https://learn.microsoft.com/en-us/azure/aks/app-routing , https://learn.microsoft.com/en-us/azure/aks/concepts-network-ingress （extract_ingress.json, search_deploy_app_step.json）

### 9. ステップ6: スケールと自己修復を体感する
- HPA: 負荷で API を自動増減（前提記事の HPA を実際に適用）。AKS は metrics-server がデフォルト有効
- Pod を手で消すと復活（自己修復の体感）= 前提記事の reconciliation ループ
- `kubectl scale` / `kubectl get hpa` / `kubectl delete pod`
- 主張: 前提記事で「魔法ではなくループ」と説明した収束を、実際に目で見る
- 根拠: 前提記事 + https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/

### 10. ［発展］パスワードレス接続 — Workload Identity（概観のみ）
- Secret に接続情報を置く方式の弱点（機密が K8s 内に居座る、ローテーション）
- Workload Identity: SA トークンを Entra トークンに交換、マネージド ID で DB に接続。パスワードレス
- `az aks update --enable-oidc-issuer --enable-workload-identity`、federated credential、SA annotation の流れを概観
- POC の次の一歩として概観のみ（深入りしない）。OIDC は K8s 1.34+ 新規クラスタでデフォルト等の注記
- 主張: 本番ではここへ向かう、という方向だけ示す
- 根拠: https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview , https://learn.microsoft.com/en-us/azure/aks/use-oidc-issuer , https://learn.microsoft.com/en-us/azure/postgresql/security/security-connect-with-managed-identity （extract_workload_identity.json, search_aks_connect_managed.json）

### 11. 後片付け
- `az group delete`（課金事故防止。前提記事の「ロードバランサ消し忘れ3週間課金」エピソードを受ける）
- 根拠: quickstart-create-server（クリーンアップ節）

### 12. まとめ — K8s が担ったもの、Azure に任せたもの
- 表で整理: K8s（Deployment/Service/Ingress/HPA/ConfigMap/Secret = 望ましい状態の維持）vs Azure マネージド（control plane / ACR / DB / LB）
- 前提記事の「線引き」を実構成で実証できた、と締める
- 省略したもの（APIM/Redis/Key Vault/監視/CI-CD）と、それぞれ何のためか一言 = 現実の次の一歩
- 根拠: 全体

## 根拠ファイル一覧（temp/aks_app_build/）

- search_aks_deploy_overview.json — AKS デプロイ概観、Azure Linux 退役
- search_aks_connect_managed.json — マネージド接続・Ingress・managed identity
- extract_build_deploy.json — ACR 作成/ビルド、`az aks create --attach-acr`
- extract_ingress.json — application routing addon、移行期注記
- extract_postgres.json — flexible-server create、ファイアウォール
- search_postgres_create.json — postgres CLI・接続文字列
- search_deploy_app_step.json — `kubectl apply`、LoadBalancer EXTERNAL-IP
- extract_workload_identity.json — Workload Identity / OIDC
