---
title: "Docker は何となく使える人のための Kubernetes 入門 — 概念・Docker との違い・使うべきかの判断"
status: plan
---

## 想定読者と方針

- 想定読者: 技術には詳しいが、Docker は `docker run` / `docker compose` を何となく使う程度。Kubernetes はほぼ無知。
- この記事が答える問い: 「Docker は分かる。で、Kubernetes は結局なにを解決する道具で、自分たちは使うべきなのか?」
- 方針: コードは雰囲気が伝わる最小スニペットに留め、**基礎概念・Docker との類似点/相違点・使用可否判断**に集中する。
- 扱わない: 本格的なマニフェスト記述、Helm / Operator / CNI / RBAC の詳細、クラスタ構築手順。

## セクション構成

### 1. この記事について（導入）
- 主張: Docker は「1 台のマシンでコンテナを動かす」道具。だが本番では「複数台に分散」「落ちたら復旧」「無停止で更新」「アクセス先を一定に」が必要になり、そこに穴が空く。その穴を埋めるのが Kubernetes（K8s）。
- ここでゴール（読後に何が分かるか）を宣言する。

### 2. なぜ Docker だけでは足りなくなるのか
- 主張: `docker run` は手元では完璧だが、(1) コンテナが落ちたら誰が再起動するか (2) サーバが 1 台死んだらどこで動かし直すか (3) 10 個に増やす/減らすのを誰がやるか (4) 更新中に止めないには、を人間が手作業で見る前提になっている。
- 「手順を実行する人」を仕組みに置き換えたい、という動機づけ。
- 根拠: Pod/controller の "automatic healing in case of Pod failure. If a Node fails, a controller notices..."（concepts/workloads/pods）

### 3. Kubernetes の本質は「命令」ではなく「望ましい状態への収束」
- 主張: K8s の核は宣言的モデル。「A をやって次に B」ではなく「最終的にこうあってほしい（desired state）」を書くと、独立した制御プロセス群が現状をそこへ continuously 収束させる（reconciliation）。これが自己修復の正体。
- 公式の強い一文を引用: "it eliminates the need for orchestration ... a set of independent, composable control processes that continuously drive the current state towards the provided desired state"（concepts/overview）
- Docker との対比: `docker run` は命令（今これを動かせ）、K8s マニフェストは状態の宣言（常に 3 つ動いていてほしい）。
- 根拠: https://kubernetes.io/docs/concepts/overview/

### 4. Docker との関係を正しく理解する（混乱しやすい点）
- 主張: 「Kubernetes は Docker の代わり」ではない。レイヤーが違う。Docker は「イメージをビルドする/1 台で動かす」開発ツール、K8s は「多数のコンテナを多数のマシンで運用する」層。
- 重要な誤解の解消: そもそも "Docker" は単一物ではなく技術スタックで、その中の `containerd` が実際のランタイム。K8s が必要とするのはそこだけで、Docker の UX 部分は不要。だから K8s は v1.24 で dockershim を外した（= ランタイムとしての Docker サポート除去）。
- ただし**`docker build` で作ったイメージはそのまま K8s で動く**。Docker は今も開発ツールとして有用。
- 根拠: https://kubernetes.io/blog/2020/12/02/dont-panic-kubernetes-and-docker/

### 5. 最小限の用語マップ（Pod / Deployment / Service / Node）
- 主張: 4 語だけ押さえれば全体像が見える。
  - **Node**: 仕事をする 1 台（VM か物理）。学習環境なら 1 台でもよい。
  - **Pod**: スケジューリングの最小単位。中に 1 個（時に複数）のコンテナ。**使い捨て(ephemeral)**で、各 Pod が固有 IP を持つ。
  - **Deployment**: 「Pod を N 個保て」を宣言し、レプリケーション・ローリング更新・自己修復を担う controller。
  - **Service**: 入れ替わり続ける Pod 群に**安定したアクセス先**を与える（サービスディスカバリ）。アプリ側を書き換えずに済む。
- さらに上位に control plane（API server / etcd / scheduler / controller-manager）と各 Node の kubelet があることだけ軽く触れる（深入りしない）。
- 根拠: workloads/pods, controllers/deployment, services-networking/service, architecture/nodes, overview/components

### 6. Docker Compose を知っているなら、ここで地続きになる
- 主張: Compose の「サービス定義 YAML」と K8s の「マニフェスト YAML」は発想が近い。実際、公式は Compose を K8s リソースへ変換する `Kompose` を案内している。
- ただし対応は 1:1 ではない（Compose にない概念＝Service の抽象、ローリング更新、複数ノード前提など）ことを注記。
- 小スニペット: compose 1 ブロック ↔ Deployment + Service の対応イメージ（最小・概念用）。
- 根拠: https://kubernetes.io/docs/tasks/configure-pod-container/translate-compose-kubernetes/

### 7. 使うべきか — 判断軸
- 主張: K8s は「複数台・無停止・自己修復・スケール」を仕組みで欲しいときに効く。それが要らないなら過剰。
- 向くケース: 多数のサービス/多数のノード、可用性要件、頻繁なデプロイ、宣言的運用への移行意思。
- 向かない/コスト過剰なケース: 1 台で足りる、トラフィックが小さい/予測可能、運用人員が薄い。K8s は "not psychic" で、教えないと正しく動かない＝学習・運用コストが現実に乗る。
- 実際の落とし穴を引用（Istio 入れて K8s 自体のデバッグに時間を溶かす / ロードバランサ消し忘れ課金など）。
- マネージド（EKS/GKE/AKS）で運用負荷の一部はクラウドに寄せられるが、複雑性ゼロにはならない。
- 代替: 単一マシンなら Compose、自己修復だけなら再起動ポリシーや小さな PaaS で足りることも。
- 根拠: https://kubernetes.io/blog/2025/10/20/seven-kubernetes-pitfalls-and-how-to-avoid/ , https://kubernetes.io/docs/concepts/security/multi-tenancy/ , https://kubernetes.io/docs/concepts/overview/（What K8s is not）

### 8. まとめと最初の一歩
- 主張: K8s = 宣言的に書いた「望ましい状態」を維持し続ける運用基盤。Docker の上位互換ではなく上位レイヤー。
- 最初の一歩: いきなり本番クラスタではなく、Docker Desktop の K8s か minikube でローカルから。
- 根拠: https://docs.docker.com/desktop/use-desktop/kubernetes/ , https://kubernetes.io/docs/tutorials/kubernetes-basics/

## frontmatter（publish 用の予定）
- title: 上記
- emoji: ☸️
- type: tech
- topics: ["kubernetes", "docker", "container", "devops", "infrastructure"]
  - 既存記事のタグ（docker, devops）を再利用。kubernetes/container/infrastructure を追加。
- published: false

## 根拠ファイル
- temp/k8s_intro/search_k8s_overview.json
- temp/k8s_intro/extract_k8s_core_concepts.json
- temp/k8s_intro/extract_k8s_arch_judgment.json
- temp/k8s_intro/search_extract_when_not_k8s.json
