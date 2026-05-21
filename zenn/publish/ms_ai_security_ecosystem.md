---
title: "Azure ガードレールの先 — PyRIT・Red Teaming Agent・Risk & Safety Evaluators・Defender for AI で“攻めて測って見張る”"
emoji: "🛡️"
type: "tech"
topics: ["azure", "security", "openai", "llm", "foundry"]
published: false
---

## この記事について

前回 [Azure中心で設計するLLMガードレール実践：入力・出力をどこで判定し、どこで止めるか](https://zenn.dev/headwaters/articles/4ba708199f684e) では、Azure AI Content Safety / Azure OpenAI / Foundry / APIM `llm-content-safety` の **runtime ガードレール設計** を整理しました。

ただ、運用に入ると次の問いがすぐ来ます。

- Prompt Shields を入れたが、**本当に効いているか試したことがあるか?**
- 本番稼働後の jailbreak / 認証情報漏えいを、**誰がどう拾うのか?**
- そもそも **PyRIT と AI Red Teaming Agent は何が違う**のか?

前回の話は、Microsoft が公開している AI セキュリティ全体像の中の **runtime gatekeeping** に絞った話でした。本稿ではその外側、つまり **build → CI → runtime monitoring → SOC** までを、Microsoft エコシステムのツール群で繋ぎ直します。

具体的には次のツールを軸にします。

- [**PyRIT**](https://microsoft.github.io/PyRIT/) — 開発手元での adversarial probing
- [**AI Red Teaming Agent (preview)**](https://learn.microsoft.com/en-us/azure/foundry/concepts/ai-red-teaming-agent) — PyRIT のクラウド自動化
- [**Risk and Safety Evaluators**](https://learn.microsoft.com/en-us/azure/foundry/concepts/evaluation-evaluators/risk-safety-evaluators) — CI ゲートになる採点器
- [**Agent Governance Toolkit (OSS)**](https://github.com/microsoft/agent-governance-toolkit) — エージェントの "行動レベル" runtime ガバナンス
- [**Defender for Cloud AI threat protection**](https://learn.microsoft.com/en-us/azure/defender-for-cloud/ai-threat-protection) — runtime での脅威検知と SOC 連携

## なぜ runtime ガードレールだけでは足りないのか

Microsoft の [Cloud Adoption Framework: Governance and security for AI agents](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ai-agents/governance-security-across-organization) は、AI セキュリティを 4 層で整理しています。

- **Data governance and compliance** — Purview など
- **Agent observability** — Defender for Cloud / Agent 365 / Log Analytics
- **Agent security** — Defender for Cloud AI threat protection / Content Safety / **AI Red Teaming Agent** / RBAC / Sentinel
- **Agent development** — Foundry SDK / Agent Framework / MCP

そして同ページは、運用ルールとして **「Mandate adversarial testing」** を明言しています。production リリース前と重要更新後の両方で red teaming を回し、prompt injection / data leakage / jailbreak を**事前**に潰せ、という主張です。

つまり Microsoft 公式の整理においても、

:::message
**「フィルタを置く」ことと「フィルタが効くと確認する」ことと「漏れたものを SOC で拾う」ことは別の責務**
:::

なのです。前回記事は最初の責務しか扱っていませんでした。本稿はその後ろ 2 つを埋めます。

## 先に結論 — Microsoft エコシステムの責務分解

ライフサイクル位相と主役ツールを先に貼ります。

| 位相 | 責務 | 主役ツール |
| --- | --- | --- |
| 開発 (local / lab) | 手動 + 自動の adversarial probing | **PyRIT** (OSS framework) |
| Pre-deployment / CI | 攻撃自動化 + safety スコア化 + CI ゲート | **AI Red Teaming Agent** + **Risk and Safety Evaluators** |
| Deployment (text レベル) | 入出力テキストのブロック | (前回記事) Content Safety / Foundry filters / APIM |
| Deployment (action レベル) | ツール呼び出し / 行動の policy enforcement | **Agent Governance Toolkit (OSS)** |
| Runtime monitoring | 漏れた攻撃の検知・可視化 | **Defender for Cloud AI threat protection** |
| SOC | XDR / SIEM 相関、incident 化 | **Defender XDR** / **Microsoft Sentinel** |

[Foundry の Observability](https://learn.microsoft.com/en-us/azure/foundry/concepts/observability) ページも明示しているとおり、Foundry は **pre-production evaluation** と **post-production monitoring** を分けて整理しています。

ここからは、各ツールが「どの位相を、どこまで」担うのかを順に見ていきます。

## PyRIT — 攻める側の OSS フレームワーク

[**PyRIT (Python Risk Identification Tool)**](https://microsoft.github.io/PyRIT/) は Microsoft が公開する OSS の AI red teaming フレームワークです。位置づけとしては「自動化可能な adversarial testing の基盤」で、3 つの使い方を公式トップで併記しています。

- **Scanner** — `pyrit_scan` / `pyrit_shell` で組み込みシナリオを CLI 実行
- **CoPyRIT (GUI)** — 人手 red teaming 向けの Web UI
- **Framework** — Python から `PromptSendingAttack` などを直接組む

公式トップ記載のキー機能を、設計判断に必要な観点で見直すと次のとおりです。

- **マルチターン攻撃戦略**: `Crescendo` / `TAP` / `Skeleton Key` などが標準同梱
- **ターゲット非依存**: OpenAI / Azure / Anthropic / Google / HuggingFace / 任意 HTTP / WebSocket / Playwright ベース Web アプリ / 自前ターゲット
- **メモリ**: SQLite または Azure SQL に会話・スコア・結果を保存。エクスポート前提
- **スコアラー**: true/false / Likert / classification / 任意ロジック、`Azure AI Content Safety` を採点器として流せる

ここで重要な誤解防止ポイントが 2 つあります。

:::message
**1. PyRIT は Azure 専用ツールではない**

OpenAI、Anthropic、HuggingFace、任意 HTTP / WebSocket、Playwright ベースの Web アプリまで叩けます。Azure 外モデルや自前推論サーバを同じスクリプトで red team するなら、PyRIT が現時点で最も自然な選択になります。

**2. PyRIT 単体は CI 用のレポート・スケジューラを持たない**

ASR ダッシュボードや継続実行・組織レポートを欲しいなら、その上に **AI Red Teaming Agent** が必要になります。
:::

## AI Red Teaming Agent (preview) — PyRIT のクラウド自動化

[**AI Red Teaming Agent**](https://learn.microsoft.com/en-us/azure/foundry/concepts/ai-red-teaming-agent) は preview の段階です。本番ワークロードへの利用は SLA がない前提で考えてください。

要するに、Foundry が PyRIT を「自分のプロジェクトの中で、自分のエージェントに対して」回せる形にしたものです。PyRIT との違いを 4 点に絞ると次になります。

- **curated seed prompts** がリスクカテゴリ別に同梱される (自前で攻撃文を書き起こさなくてよい)
- **fine-tuned adversarial LLM** が、攻撃シミュレーションと Risk and Safety Evaluators による採点を肩代わりする
- 指標として **Attack Success Rate (ASR)** を出す
- **pre-deployment 用の大規模実行**と **post-deployment 用のスケジュール実行**が分かれている

### 対応攻撃戦略 (PyRIT 由来)

公式 docs に列挙されている主なものだけ抜粋しても、攻撃面の広さは伝わります。

- `AnsiAttack` — ANSI エスケープによる表示・挙動操作
- `AsciiArt` / `AsciiSmuggler` — ASCII による視覚化・データ密輸
- `Atbash` / `Caesar` — 古典換字暗号でフィルタ回避
- `Base64` / `Binary` — エンコード変換
- `Flip` — 文字列反転(冒頭の "loot a bank" 例のように、direct refusal を回避する典型)
- `IndirectJailbreak` — 文書・ソース埋め込み型 (XPIA)
- `Crescendo` / `Multi turn` — 多ターンで段階的に追い込む

### SDK 実行 (cURL / Python)

Foundry SDK 経由でクラウド実行する場合の最小手順は次のようになります ([公式ドキュメント](https://learn.microsoft.com/en-us/azure/foundry/how-to/develop/run-ai-red-teaming-cloud) より)。

```bash
pip install "azure-ai-projects>=2.0.0"
```

```python
# 攻撃戦略を指定して red team run を作成する
eval_run = client.evals.runs.create(
    eval_id=red_team.id,
    name="Red Team Agent Safety Eval Run",
    data_source={
        "type": "azure_ai_red_team",
        "item_generation_params": {
            "type": "red_team_taxonomy",
            "attack_strategies": ["Flip", "Base64", "IndirectJailbreak"],
            "num_turns": 5,
            "source": {"type": "file_id", "id": taxonomy_file_id},
        },
        "target": target.as_dict(),
    },
)
```

`attack_strategies` 配列で戦略を、`num_turns` でマルチターン深さを、`source.id` で taxonomy (リスクカテゴリ + 対象エージェント設定) をそれぞれ指定します。

### preview の制約は必ず読む

設計判断に直結するので、公式の supportability matrix を表で残します。

| 対象 | 状態 |
| --- | --- |
| Foundry hosted prompt agents | Supported |
| Foundry hosted container agents | Supported |
| Foundry workflow agents | **Not Supported** |
| Non-Foundry agents | **Not Supported** |
| Non-Azure tools | **Not Supported** |
| Azure tool calls | Supported |
| Function tool calls | **Not Supported** |
| Browser automation tool calls | **Not Supported** |
| Connected Agent tool calls | **Not Supported** |
| Computer Use tool calls | **Not Supported** |

さらに次の制限もあります。

- **テキスト専用** (text-based scenarios only) — 画像 / 音声は対象外
- 実行には Foundry プロジェクトに対して `Azure AI User` ロールが必要
- Python 3.9 以上
- エージェント評価には対象エージェントが Foundry にデプロイ済みであること

:::message alert
言い換えると、**Foundry hosted prompt / container agent 以外を red team したいなら、自動的に PyRIT 直接利用に落ちます。**特に「関数ツールを多用する agent」「Computer Use を使う agent」「Foundry workflow agent」は preview 時点では Red Teaming Agent の対象外です。
:::

## Risk and Safety Evaluators — CI ゲートになる採点器

Red Teaming Agent が「攻める側」だとすると、[**Risk and Safety Evaluators**](https://learn.microsoft.com/en-us/azure/foundry/concepts/evaluation-evaluators/risk-safety-evaluators) は「採点する側」です。dataset / model / agent それぞれに対して safety スコアを返します。

公式 docs のカテゴリ表を、実務で使う観点で再整理します。

| 評価カテゴリ | 対応対象 | 何を見るか |
| --- | --- | --- |
| Hate / Sexual / Violence / Self-harm | Model + agents | 有害コンテンツ生成 |
| Indirect Attack (XPIA) | **Model only** | 文書・ソース埋め込み型 jailbreak への耐性 |
| Protected materials | Model + agents | 著作物テキスト・コードとの近さ |
| Code vulnerability | Model + agents | `path-injection` / `sql-injection` / `code-injection` / `stack-trace-exposure` 等 (Python・Java・C++・C#・Go・JS・SQL) |
| Ungrounded attributes | Model + agents | 人物属性 (demographics / emotional state) への根拠なき推論 |
| **Prohibited actions** (preview) | **Agents only** | 禁止アクション・禁止ツール呼び出しに乗るか |
| **Sensitive data leakage** (preview) | **Agents only** | 金融 / PII / health の漏えい |

### XPIA の中身

XPIA (cross-domain prompt injected attacks) は単一スコアではなく、内訳として次の 3 カテゴリで判定されます。

- **Manipulated content** — 情報の改竄・捏造、言語や書式の改変、特定情報の強調・隠蔽
- **Intrusion** — 不正アクセス・特権昇格、バックドア生成、伝統的 jailbreak での回避
- **Information gathering** — 不正なデータアクセス・改ざん・削除

前回記事で扱った Prompt Shields の「documents 経由攻撃」と直接対応する評価軸です。

### なぜ `prohibited_actions` と `sensitive_data_leakage` はエージェント専用なのか

両カテゴリは入力として `query` / `response` / **`tool_calls`** を取ります。tool_calls が要件である以上、bare model に対しては評価できません。これは Microsoft 公式の制約として明示されています。

設計上の意味は次のとおりです。

:::message
**「モデルが何を喋るか」と「エージェントが何を実行するか」は別の安全境界**

モデル単体評価は harm filter までで止まります。tool 実行を含む agent では、`prohibited_actions` / `sensitive_data_leakage` を追加しないと、authorization と data exfil を見落とします。
:::

### CI ゲートとしての置き場所

実運用では、Red Teaming Agent run の結果に対して次のような閾値ゲートを CI に組み込む構成が現実的です。

- **ASR が閾値超え** → マージ拒否、再学習・ガードレール再調整
- **`protected_materials` がヒット** → 法務 / 編集レビューに自動エスカレーション
- **`code_vulnerability` の重大度 high** → セキュリティレビュー必須
- **(agent) `prohibited_actions` ヒット** → ツール権限の見直し
- **(agent) `sensitive_data_leakage` ヒット** → DLP / 出力フィルタ見直し

ここまでが **shift-left** 側、つまり「デプロイ前にできるだけ壊しておく」側の話です。

## Agent Governance Toolkit (OSS) — エージェントの "行動レベル" ガバナンス

ランタイム側に戻る前に、もう一つ押さえておきたいツールがあります。

2026-04 に Microsoft Open Source Blog で公開された [**Agent Governance Toolkit (AGT)**](https://opensource.microsoft.com/blog/2026/04/02/introducing-the-agent-governance-toolkit-open-source-runtime-security-for-ai-agents) です。MIT ライセンスの OSS で、リポジトリは [github.com/microsoft/agent-governance-toolkit](https://github.com/microsoft/agent-governance-toolkit) にあります。

立ち位置は次の通りです。

:::message
- 前回記事の Content Safety / Prompt Shields は **「LLM が何を言ったか」(テキストレベル)** に効く
- AGT は **「エージェントが何をしようとしたか — tool 呼び出し / 決定」(action レベル)** に効く
- 同じ runtime 層だが、見ているものが違う
:::

Microsoft 公式が AGT を **「OWASP Agentic AI Top 10 の 10 項目すべてに対応する初のツールキット」** と位置付けている点が重要です。先に書いた Red Teaming Agent の Q&A は「Red Team SDK は OWASP を全カバーはしない」と明言していました。その**ランタイム側の穴**を、framework-agnostic な OSS で埋めるのが AGT の狙いです。

### 何ができるのか

[アーキテクチャ深掘り記事](https://techcommunity.microsoft.com/blog/linuxandopensourceblog/agent-governance-toolkit-architecture-deep-dive-policy-engines-trust-and-sre-for/4510105) によれば、AGT は 7 パッケージ構成で、概ね以下の責務を持ちます。

- **Policy engine** — deterministic に sub-millisecond で policy 判定 (LLM 判定でない)
- **Zero-trust identity** — エージェント単位の暗号学的 ID
- **Execution sandboxing** — ツール実行の隔離
- **Audit logging / SRE** — 自律エージェントに対する SRE 慣行 (alert / runbook / postmortem)
- **Agent Mesh** — 複数エージェント間の信頼スコアリング
- **Compile-time / CI enforcement** — .NET SDK は MSBuild ベースで**コンパイル時にポリシー違反を弾く**、language-agnostic な CI チェックも提供 ([shift-left 解説記事](https://techcommunity.microsoft.com/blog/linuxandopensourceblog/shift-left-governance-for-ai-agents-how-the-agent-governance-toolkit-helps-you-c/4516481))

つまり AGT は「runtime だけ」ではなく、**runtime + shift-left** の両側を持つツールキットです。

### Framework-agnostic に hook

AGT のもう一つの特徴は、エージェント実装の前提を縛らないことです。各フレームワークのネイティブ拡張点に hook します。

- **LangChain** — callback handler
- **CrewAI** — task decorator
- **Google ADK** — plugin system
- **Microsoft Agent Framework** — middleware pipeline

[AGT と Microsoft Agent Framework の "Better Together" 記事](https://devblogs.microsoft.com/agent-framework/governance-at-the-speed-of-agents-microsoft-agent-framework-and-agent-governance-toolkit-better-together) は両者を **「build / orchestrate」と「govern / audit」の二層**として位置付けています。Microsoft Agent Framework 1.0 が前者、AGT が後者という対です。

最小インストールは次の通り、Python 1 行で始められます。

```bash
pip install agent-governance-toolkit[full]
```

### 注意点

公式 blog 自身が、

> The policy files, workflow configurations, and code samples in this post are illustrative examples designed to demonstrate governance patterns. They are not intended as production-ready security configurations. Agent Governance Toolkit is designed to help implement governance controls but does not guarantee compliance with any specific regulatory framework, including but not limited to GDPR, HIPAA, EU AI Act, or Colorado AI Act.

と明示している点は外せません。**サンプル YAML は出発点であって、HIPAA / SOC2 / EU AI Act などのコンプライアンス遵守を保証するものではない**、と公式が宣言しています。導入時は法務・コンプラ部門とのレビューが前提です。

### Defender for AI / Content Safety との重なり方

3 者の役割は重なるようで重ならない、というのが今の整理です。

| ツール | 何を見るか | 何ができるか |
| --- | --- | --- |
| Azure AI Content Safety / Prompt Shields | 入出力 **テキスト** | text レベルで block / detect |
| **AGT** | エージェントの **tool 呼び出し / 行動 / ID** | action レベルで policy enforcement、コンパイル時シフトレフトも |
| Defender for Cloud AI threat protection | **runtime の挙動** | アラート化して **Defender XDR / Sentinel** に流す (止めない) |

「何を言ったか」「何をしようとしたか」「漏れたあと SOC で拾うか」を別の層に分解する、という前回記事の発想を agentic 領域に拡張したのが AGT という整理です。

## Defender for Cloud AI threat protection — 漏れたものを SOC で拾う

ここから **shift-right** 側、本番稼働後の話です。

[**Defender for Cloud AI threat protection**](https://learn.microsoft.com/en-us/azure/defender-for-cloud/ai-threat-protection) は **GA** で、Defender for AI Services プランの一部です。役割を一言で言うと次になります。

:::message
**Prompt Shields はリクエストを止める / 検出する。**
**Defender for AI は、その検出を含めたインシデントを SOC (Defender XDR / Sentinel) に届ける。**

ブロック装置の代替ではなく、ブロックを観測可能にする装置です。
:::

実際、後述するアラート群を見ると、`AI.Azure_Jailbreak.ContentFiltering.BlockedAttempt` のような **Prompt Shields のブロック結果に依存したアラート**が並びます。Prompt Shields の代わりではなく、Prompt Shields が無いと一部のアラートが鳴らない関係です。

### 主要アラート

[Alerts for AI services](https://learn.microsoft.com/en-us/azure/defender-for-cloud/alerts-ai-workloads) に掲載されているアラートのうち、設計に効くものを抜粋します。すべて MITRE ATT&CK タクティクスと severity が付与されています。

| アラート ID | 何を検知するか | MITRE | Severity |
| --- | --- | --- | --- |
| `AI.Azure_CredentialTheftAttempt` | モデル応答に認証情報が含まれた | Credential Access / Lateral Movement / Exfiltration | Medium |
| `AI.Azure_Jailbreak.ContentFiltering.BlockedAttempt` | Prompt Shields によって jailbreak がブロックされた | Privilege Escalation / Defense Evasion | Medium |
| `AI.Azure_Jailbreak.ContentFiltering.DetectedAttempt` | Prompt Shields で検出されたが**ブロックされなかった** | Privilege Escalation / Defense Evasion | Medium |
| `AI.Azure_Agentic_Jailbreak` | エージェントへの jailbreak 検出 | Privilege Escalation / Defense Evasion | Medium |
| (preview) `AI.Azure_Agentic_ASCIISmuggling` | ASCII smuggling 経由の indirect prompt injection | Impact | High |
| (preview) `AI.Azure_Agentic_DOWVolumeAnomaly` | "wallet attack" — 過剰利用による経済的攻撃 | Impact | Medium |
| (preview) `AI.Azure_Agentic_InstructionLeakage` | system prompt の漏えい | Impact | Low |
| (preview) `AI.Azure_LLMReconnaissance` / `AI.Azure_Agentic_LLMReconaissance` | capability / safety guardrail への偵察 | (記載) | (記載) |
| (preview) `AI.Azure_AnomalousToolInvocation` | エージェントの異常なツール呼び出し | Execution | Low |
| (preview) `AI.Azure_Agentic_AccessFromSuspiciousIP` | 脅威インテリ判定済み IP からのアクセス | Execution | High |
| (preview) `AI.Azure_Agentic_AccessFromAnonymizedIP` | Tor 経由アクセス | Execution | — |
| (preview) `AI.Azure_Agentic_MaliciousUrl.UserPrompt` | エージェントへのフィッシング試行 | — | — |

設計面で押さえるべきポイントは 3 つです。

- **`BlockedAttempt` と `DetectedAttempt` を区別している**
  - 「ブロックできた」と「検出はしたが通った」は別事象として運用する必要があります。
- **エージェント系アラートが急速に増えている**
  - `Agentic_*` 系の preview アラートは、tool 呼び出し・偵察・経済攻撃まで広がっています。エージェント運用にはこの層が事実上必須です。
- **MITRE タクティクスが付いているので SIEM 相関に乗る**
  - Sentinel 側で AI 由来事象を従来の検知ルールと同じ枠で扱えます。

### 制約 (必ず把握する)

- **テキスト トークン専用** — 画像・音声トークンはスキャン対象外
- 対応 AI サービス: Azure OpenAI / Azure AI Model Inference の対応モデル
- サブスクリプションレベルで有効化、必要ロールは Owner 相当
- **30 日無料トライアル、最大 75 Billion トークン** スキャンまで無料、超過時に課金開始
- 商用 Azure、Azure Government、21Vianet、**接続済み AWS アカウント** にも対応
- Defender XDR ポータルに統合されるため、AI ワークロード以外のアラートと同じインシデント上で相関できる

Sentinel など SIEM へのストリーミングは Defender for Cloud アラート全体の枠組みに乗ります ([Security alerts and incidents](https://learn.microsoft.com/en-us/azure/defender-for-cloud/alerts-overview))。

## 全体像 — build → CI → runtime → SOC

ここまでの整理を、1 つの流れに直すと次のようになります。

```
[開発 / lab]
   |- PyRIT (framework / scanner / CoPyRIT)
   |    |- Crescendo / TAP / Skeleton Key などで手元検証
   |    |- 任意ターゲット (Azure外モデル / 自前HTTP / Playwright) 含む
   v
[CI / pre-deployment]
   |- AI Red Teaming Agent (preview)
   |    |- curated seed prompts × 攻撃戦略 (Flip / Base64 / IndirectJailbreak / ...)
   |    |- Risk and Safety Evaluators で採点
   |    |- ASR / カテゴリスコアを CI ゲートに使う
   v
[Deployment / runtime — text レベル] ← 前回記事の責務
   |- Azure AI Content Safety (Prompt Shields / blocklist / Groundedness / Protected material)
   |- Azure OpenAI / Foundry content filtering
   |- APIM `llm-content-safety` (共通強制)
   v
[Deployment / runtime — action レベル]
   |- Agent Governance Toolkit (OSS / MIT)
   |    |- policy engine / zero-trust ID / execution sandbox / Agent Mesh
   |    |- LangChain / CrewAI / Google ADK / Microsoft Agent Framework に hook
   |    |- OWASP Agentic Top 10 をカバー
   v
[Runtime monitoring]
   |- Defender for Cloud AI threat protection (GA)
   |    |- BlockedAttempt / DetectedAttempt / CredentialTheft / Agentic_Jailbreak ...
   |    |- MITRE タクティクス付きアラート
   v
[SOC / XDR / SIEM]
   |- Defender XDR (相関 + incident 化)
   |- Microsoft Sentinel (SIEM / SOAR ストリーム)
   v
[Continuous loop]
   |- post-deployment 用 Red Teaming Agent スケジュール実行
   |- 検知された新手攻撃を seed prompt に追加 → CI に反映
```

「runtime ガードレール」はこの 5 層スタックの**中の 1 層**でしかありません。残り 4 層を埋めないと「フィルタは置いた、効いてるかは知らない、漏れたら気づけない」という状態になります。

### Foundry にデプロイしていないモデルや自前 HTTP backend の場合

エージェントが Foundry hosted prompt / container agent 以外、または対象モデルが Azure OpenAI / AI Model Inference でない場合の代替パターンも併せて押さえておきます。

| 位相 | 通常パターン | 外側パターン |
| --- | --- | --- |
| 開発 / lab | PyRIT | PyRIT (変わらず) |
| CI / pre-deploy | AI Red Teaming Agent + Evaluators | **PyRIT 直接利用 + 自前スコアラー** |
| Runtime (action) | AGT (Microsoft Agent Framework hook) | **AGT (LangChain / CrewAI / Google ADK hook)** — そのまま残せる |
| Runtime monitoring | Defender for Cloud AI threat protection | **自前ログ + APIM `llm-content-safety` 観測 + Sentinel 直流し** |
| SOC | Defender XDR / Sentinel | Sentinel (Defender XDR 統合は限定的) |

つまり PyRIT と AGT は「どの構成でも残る共通基盤」、Red Teaming Agent と Defender for AI は「Foundry / Azure OpenAI 経路に寄せたときに得られる自動化」という関係になります。AGT が framework-agnostic で OSS であることは、**Azure 外モデル / 非 Foundry エージェントを採用したときに最も効きます**。

## 設計でハマりやすい勘違い

### 1. Prompt Shields があるから Defender for AI は不要

違います。

- Prompt Shields は止める / 検出する側
- Defender for AI は、その検出結果を SOC に届けて相関する側

`AI.Azure_Jailbreak.ContentFiltering.BlockedAttempt` のように、**Defender for AI のアラート自体が Prompt Shields の検出結果に依存している**ものが多いです。前後関係であって代替ではありません。

### 2. AI Red Teaming Agent があるから PyRIT を知らなくてよい

違います。

Red Teaming Agent は preview の supportability matrix が厳しく、`Foundry workflow agents` / `Non-Foundry agents` / `Function tool calls` / `Browser automation` / `Computer Use tool calls` がいずれも非対応です。これらを red team したい場合は **PyRIT 直接利用に落ちます**。

エージェントの実装方針が固まる前から「PyRIT を扱える人」と「Red Teaming Agent SDK を扱える人」の両方を組織に置くのが安全です。

### 3. Red Teaming Agent を回せば OWASP LLM / Agentic Top 10 をカバーできる

違います。

Microsoft の [公式 Q&A](https://learn.microsoft.com/en-us/answers/questions/5621948/with-the-red-team-sdk-can-we-test-only-safety-risk) でも明言されているとおり、Red Team Agent SDK は **safety 中心**で、supply chain / model theft / インフラ設定不備などは対象外です。

- 対象になりやすい: prompt injection、indirect prompt injection、jailbreak、deception、harmful content
- 対象になりにくい: 不適切な plugin 統合、モデル盗難、インフラ misconfig

OWASP LLM Top 10 全体は伝統的なセキュリティテストと組み合わせる前提で考えてください。**OWASP Agentic Top 10 を "runtime で" 押さえたい**場合は、別軸として **Agent Governance Toolkit** を入れるのが Microsoft 公式の置き方です (AGT は「全 10 項目をカバーする初のツールキット」と謳われています)。

### 4. Risk and Safety Evaluators の `prohibited_actions` を model evaluation に使える

違います。

`prohibited_actions` と `sensitive_data_leakage` は **agent only** で、入力として `tool_calls` が必須です。dataset / model 評価には現在出ません。「モデル単体評価で済ませたい」ケースでは、これら 2 つは原理的に空欄になります。

### 5. Defender for AI を入れれば画像・音声攻撃も拾える

違います。

公式 docs に **text tokens only** と明示されています。マルチモーダル入力での攻撃 (画像内の隠し命令、音声プロンプトインジェクション) は別経路で監視する必要があります。

### 6. PyRIT を CI で回せば runtime ガードレールは不要

違います。

PyRIT / Red Teaming Agent でカバーできるのは「自分が seed prompt に入れた攻撃」だけです。本番に到達してから出てきた **未知の攻撃** は、結局 Prompt Shields + Defender for AI + SOC で拾うしかありません。

CI と runtime は **互いの保険**であって、片方で片方を要らなくするものではありません。

## まとめ

前回記事は Azure を軸にした **runtime ガードレール**を整理しました。本稿はその外側を埋めました。

最終的に、Microsoft の AI セキュリティスタックは次の 6 つの責務に分解できます。

1. **開発 / lab**: PyRIT で手元の手探り red teaming (Azure 外モデルも含む)
2. **CI / pre-deploy**: AI Red Teaming Agent + Risk and Safety Evaluators で ASR とカテゴリスコアをゲートにする
3. **runtime (text レベル)**: 前回記事の Content Safety / Foundry filters / APIM
4. **runtime (action レベル)**: Agent Governance Toolkit でツール呼び出し / ID / sandbox を policy で縛る
5. **runtime monitoring**: Defender for Cloud AI threat protection がアラートを生む
6. **SOC**: Defender XDR / Microsoft Sentinel で相関・追跡する

そして個々のツールには明確な境界線があります。

- PyRIT は **OSS** で、対象は任意。Red Teaming Agent は **preview** で、対象は Foundry hosted prompt / container agent に限定。
- Risk and Safety Evaluators は **agent only** カテゴリ (`prohibited_actions` / `sensitive_data_leakage`) を持っており、モデル単体評価では取り切れない。
- Agent Governance Toolkit は **OSS (MIT)** で **framework-agnostic**。Azure 外モデルや非 Foundry エージェントでも使えるが、公式自身が「コンプライアンス遵守を保証するものではない」と明示している。
- Defender for AI は **GA** だが **text only** で、Azure OpenAI / AI Model Inference 経路でなければ機能しない。

「ガードレールを置く」ことと「ガードレールが効くかを継続的に確かめる」ことを **同じレベルで予算化する** こと。これが Microsoft 公式 4 層モデルの要求であり、本稿で整理したかったことのすべてです。

## 参考リンク

### 前回記事

- [Azure中心で設計するLLMガードレール実践：入力・出力をどこで判定し、どこで止めるか](https://zenn.dev/headwaters/articles/4ba708199f684e)

### Microsoft 公式 (build / CI / pre-deploy)

- [PyRIT — Python Risk Identification Tool](https://microsoft.github.io/PyRIT/)
- [AI Red Teaming Agent (preview)](https://learn.microsoft.com/en-us/azure/foundry/concepts/ai-red-teaming-agent)
- [Run AI Red Teaming Agent in the cloud (Foundry SDK)](https://learn.microsoft.com/en-us/azure/foundry/how-to/develop/run-ai-red-teaming-cloud)
- [Risk and Safety Evaluators](https://learn.microsoft.com/en-us/azure/foundry/concepts/evaluation-evaluators/risk-safety-evaluators)
- [Observability in Generative AI (pre/post-prod)](https://learn.microsoft.com/en-us/azure/foundry/concepts/observability)

### Microsoft 公式 (runtime — action レベル)

- [Introducing the Agent Governance Toolkit (Microsoft Open Source Blog)](https://opensource.microsoft.com/blog/2026/04/02/introducing-the-agent-governance-toolkit-open-source-runtime-security-for-ai-agents)
- [AGT Architecture Deep Dive (Tech Community)](https://techcommunity.microsoft.com/blog/linuxandopensourceblog/agent-governance-toolkit-architecture-deep-dive-policy-engines-trust-and-sre-for/4510105)
- [Shift-Left Governance for AI Agents](https://techcommunity.microsoft.com/blog/linuxandopensourceblog/shift-left-governance-for-ai-agents-how-the-agent-governance-toolkit-helps-you-c/4516481)
- [AGT + Microsoft Agent Framework Better Together](https://devblogs.microsoft.com/agent-framework/governance-at-the-speed-of-agents-microsoft-agent-framework-and-agent-governance-toolkit-better-together)
- [Govern AI Agents on App Service with AGT](https://techcommunity.microsoft.com/blog/appsonazureblog/govern-ai-agents-on-app-service-with-the-microsoft-agent-governance-toolkit/4510962)
- GitHub: [github.com/microsoft/agent-governance-toolkit](https://github.com/microsoft/agent-governance-toolkit)

### Microsoft 公式 (runtime monitoring / SOC)

- [Overview - AI threat protection (Defender for Cloud)](https://learn.microsoft.com/en-us/azure/defender-for-cloud/ai-threat-protection)
- [Alerts for AI services](https://learn.microsoft.com/en-us/azure/defender-for-cloud/alerts-ai-workloads)
- [AI security recommendations reference](https://learn.microsoft.com/en-us/azure/defender-for-cloud/recommendations-reference-ai)
- [Security alerts and incidents - Defender for Cloud](https://learn.microsoft.com/en-us/azure/defender-for-cloud/alerts-overview)

### Microsoft 公式 (組織ガバナンス / 4 層モデル)

- [Governance and security for AI agents across the organization](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ai-agents/governance-security-across-organization)
