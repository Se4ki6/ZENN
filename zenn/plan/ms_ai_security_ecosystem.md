---
title: "Azure ガードレールの先 — PyRIT・Red Teaming Agent・Risk & Safety Evaluators・Defender for AI で “攻めて測って見張る”"
status: plan
---

## 想定読者と前提

- 前回記事 [Azure中心で設計するLLMガードレール実践](https://zenn.dev/headwaters/articles/4ba708199f684e) (または同等の知識) で、Azure AI Content Safety / Azure OpenAI / Foundry / APIM `llm-content-safety` を使った **runtime のガードレール設計** は理解している
- そのうえで次の問いに即答できない人を想定する
  - 「Prompt Shields を入れたが、それが本当に効いているか試したことがあるか?」
  - 「本番稼働後の jailbreak / 認証情報漏えいを、誰がどう拾うのか?」
  - 「PyRIT と AI Red Teaming Agent は何が違うのか?」
- Foundry SDK / Defender for Cloud / Sentinel の存在は知っているが、**AI 文脈での役割分担**が言語化できていない中級以上のエンジニア

ジュニア向けの「生成 AI とは」「Azure OpenAI とは」は扱わない。

## この記事が答える問い

1. 前回扱った runtime ガードレールだけで足りない理由 (Microsoft 公式の 4 層モデル)
2. **PyRIT** は何で、AI Red Teaming Agent と何が違うのか
3. **AI Red Teaming Agent (preview)** はどこまで自動化してくれるのか、何が対象外か
4. **Risk and Safety Evaluators** をいつ CI ゲートとして使うか、`prohibited_actions` / `sensitive_data_leakage` がなぜエージェント専用なのか
5. **Defender for Cloud AI threat protection** は Prompt Shields と役割分担としてどう違うのか
6. これら 4 つを build → CI → deploy → runtime → SOC でどう組み合わせるか
7. 設計でハマる典型的な勘違い

## 記事タイトル案

- 第1案: 「Azure ガードレールの先 — PyRIT・Red Teaming Agent・Risk & Safety Evaluators・Defender for AI で“攻めて測って見張る”」(採用)

## セクション構成

### 1. なぜ runtime ガードレールだけでは足りないのか

- 前回記事は **runtime gatekeeping** (リクエスト時点で入出力を弾く) しか扱っていない
- だが、Microsoft 公式の Cloud Adoption Framework は AI セキュリティを 4 層で整理している
  - Data governance and compliance (Purview)
  - Agent observability (Defender for Cloud / Agent 365 / Log Analytics)
  - **Agent security (Defender for Cloud AI threat protection / Content Safety / AI Red Teaming Agent / RBAC / Sentinel)**
  - Agent development (Foundry SDK / Agent Framework / MCP)
- そして CAF は「Mandate adversarial testing」を明示している (production release 前 + 重要更新後)
- 主張: **「フィルタを置く」ことと「フィルタが効くと確認する」ことと「漏れたものを SOC で拾う」ことは別の責務**
- 根拠 URL: [Governance and security for AI agents](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ai-agents/governance-security-across-organization)

### 2. 先に結論 — Microsoft エコシステムの責務分解

- 表形式で整理する

| ライフサイクル位相 | 責務 | 主役ツール |
| --- | --- | --- |
| 開発 (local / lab) | 手動・自動の adversarial probing | **PyRIT** (OSS framework) |
| Pre-deployment / CI | 攻撃自動化 + safety スコア化 + CI ゲート | **AI Red Teaming Agent** + **Risk and Safety Evaluators** |
| Deployment (text レベル) | runtime での入出力テキストブロック | (前回記事) Content Safety / Foundry filters / APIM |
| Deployment (action レベル) | ツール呼び出し / 行動の policy enforcement | **Agent Governance Toolkit (OSS)** |
| Runtime monitoring | 漏れた攻撃の検知・可視化 | **Defender for Cloud AI threat protection** |
| SOC / 相関分析 | XDR / SIEM 相関、incident 化 | **Defender XDR** / **Microsoft Sentinel** |

- 「runtime で止める」と「事前に試す」と「事後に拾う」を分けるのが今回の整理軸
- 根拠 URL: [Observability in Generative AI](https://learn.microsoft.com/en-us/azure/foundry/concepts/observability) (pre-prod / post-prod 評価の公式整理)

### 3. PyRIT — “攻める側” の OSS フレームワーク

- **PyRIT (Python Risk Identification Tool)** は Microsoft が公開する OSS の AI red teaming フレームワーク
- 3 つの使い方 (公式トップに併記されている)
  - **Scanner**: `pyrit_scan` / `pyrit_shell` で組み込みシナリオを CLI 実行
  - **CoPyRIT (GUI)**: 人手 red teaming 向け Web UI
  - **Framework**: Python から `PromptSendingAttack` などを直接組む
- できること
  - multi-turn 攻撃戦略: **Crescendo / TAP / Skeleton Key** (公式トップで明示) + single-turn
  - ターゲット非依存: OpenAI / Azure / Anthropic / Google / HuggingFace / 任意 HTTP / WebSocket / Playwright ベース Web アプリ
  - メモリ: SQLite または Azure SQL に会話・スコア・結果を保存
  - スコアラー: true/false / Likert / classification / 任意ロジック、`Azure AI Content Safety` を採点器に流せる
- 「Azure 専用ツール」ではない。**Azure 外モデルも同じスクリプトで叩ける**ことが PyRIT の強み
- 設計上の位置づけ
  - PyRIT 単体は **自動化可能なフレームワーク** であって、CI のレポート機能や ASR ダッシュボードは持たない
  - その上に Foundry がレポーティング・スケジューリングをかぶせたのが AI Red Teaming Agent
- 根拠 URL: [PyRIT](https://microsoft.github.io/PyRIT/)

### 4. AI Red Teaming Agent (preview) — PyRIT のクラウド自動化

- Foundry が PyRIT を「自分のプロジェクトの中で・自分のエージェントに対して」回せる形にした **preview** 機能
- 主要な追加価値
  - **curated seed prompts** がリスクカテゴリ別に提供される (自前で攻撃文を書かなくてよい)
  - **fine-tuned adversarial LLM** が攻撃シミュレーションと採点を肩代わりする
  - 主要指標として **ASR (Attack Success Rate)** を出す
  - **pre-deployment + post-deployment scheduled run** に分かれている (CI 用と継続監視用)
- 対応する攻撃戦略 (PyRIT 由来、抜粋)
  - AnsiAttack / AsciiArt / AsciiSmuggler / Atbash / Base64 / Binary / Caesar / Flip / IndirectJailbreak / Crescendo / Multi turn / …
- SDK 実行 (短いコード片で見せる)
  - `pip install "azure-ai-projects>=2.0.0"`
  - `client.evals.runs.create()` に `azure_ai_red_team` データソース、`attack_strategies` 配列、`num_turns`、taxonomy file_id、対象エージェントを渡す
- **しっかり書くべき制約 (preview)**
  - 対応: Foundry hosted prompt agents / container agents、Azure tool calls
  - **非対応**: Foundry workflow agents、Non-Foundry agents、Non-Azure tools、Function tool calls、Browser automation、Connected Agent tool calls、**Computer Use tool calls**
  - **テキスト専用 (text-based scenarios only)** — 画像・音声は対象外
  - 必要ロール: Foundry プロジェクトに `Azure AI User` ロール
- 根拠 URL: [AI Red Teaming Agent](https://learn.microsoft.com/en-us/azure/foundry/concepts/ai-red-teaming-agent) / [Run AI Red Teaming Agent in the cloud](https://learn.microsoft.com/en-us/azure/foundry/how-to/develop/run-ai-red-teaming-cloud)

### 5. Risk and Safety Evaluators — CI ゲートになる “採点器”

- Foundry の評価機構の一部で、**dataset / model / agent** に対して safety スコアを返す
- カテゴリ整理 (公式 docs の表をベースに、本記事用に再整理)

| 評価カテゴリ | 対応対象 | 何を見るか |
| --- | --- | --- |
| Hate / Sexual / Violence / Self-harm | Model + agents | 有害コンテンツ生成 |
| Indirect Attack (XPIA) | Model only | 文書・ソース埋め込み型の jailbreak への耐性 |
| Protected materials | Model + agents | 著作物テキストとの近さ |
| Code vulnerability | Model + agents | path-injection / sql-injection / code-injection / stack-trace-exposure 等 (Python・Java・C++・C#・Go・JS・SQL) |
| Ungrounded attributes | Model + agents | 人物属性 (demographics / emotional state) に対する根拠なき推論 |
| Prohibited actions (preview) | **Agents only** | エージェントが禁止アクション・禁止ツール呼び出しに乗るか |
| Sensitive data leakage (preview) | **Agents only** | financial / PII / health の漏えい |

- XPIA はさらに **Manipulated content / Intrusion / Information gathering** の 3 カテゴリで内訳判定する (XPIA = cross-domain prompt injected attacks)
- 「`prohibited_actions` と `sensitive_data_leakage` がエージェント限定」理由 — どちらも **tool_calls の検査が必須** だから (ツール呼び出しがない bare model にはそもそも判定対象がない)
- 設計上の位置づけ
  - CI で「ASR が閾値を超えたらマージ拒否」「protected materials がヒットしたら revision workflow」のように **shift-left のゲート** に使う
- 根拠 URL: [Risk and Safety Evaluators](https://learn.microsoft.com/en-us/azure/foundry/concepts/evaluation-evaluators/risk-safety-evaluators)

### 6. Agent Governance Toolkit (OSS) — エージェントの “行動レベル” ガバナンス

- 2026-04 に Microsoft Open Source Blog で公開された **OSS (MIT)** のツールキット
- 立ち位置: 前回記事や Defender for AI が扱う **"テキストレベル"** とは別軸の、**"action / tool call レベル"** のランタイムガバナンス
- 公式の謳い文句: **OWASP Agentic AI Top 10 の 10 項目をすべてカバー**する初のツールキット、deterministic で sub-millisecond の policy enforcement
- 構成
  - 7 パッケージ (policy engine、zero-trust identity、execution sandbox、audit log、Agent Mesh、SRE 機構、CI / コンパイル時チェック)
  - **framework-agnostic** — LangChain (callback handler)、CrewAI (task decorator)、Google ADK (plugin system)、Microsoft Agent Framework (middleware pipeline) のいずれにも hook 可能
  - **shift-left 側も持つ** — .NET SDK は MSBuild ベースで **コンパイル時にポリシー違反を弾く**、language-agnostic CI チェックも提供
- 設計上の位置づけ
  - Content Safety / Prompt Shields は **「LLM が何を言ったか」** に効く
  - AGT は **「エージェントが何をしようとしたか (tool 呼び出し / decision)」** に効く
  - 「Red Teaming Agent SDK では OWASP は対象外」と公式 Q&A が言っていた穴を **runtime 側から埋める** ツールとして読むのが自然
  - Microsoft Agent Framework と「build / orchestrate」と「govern / audit」で対になる位置づけ
- 注意点 (記事で必ず断る)
  - 公式 blog 自身が **「サンプルは production-ready ではない / GDPR / HIPAA / EU AI Act / Colorado AI Act の遵守を保証しない」** と明示 → 法務・コンプラ判断は別途必要
- 根拠 URL:
  - [Introducing the Agent Governance Toolkit (Microsoft Open Source Blog)](https://opensource.microsoft.com/blog/2026/04/02/introducing-the-agent-governance-toolkit-open-source-runtime-security-for-ai-agents)
  - [Architecture Deep Dive (Tech Community)](https://techcommunity.microsoft.com/blog/linuxandopensourceblog/agent-governance-toolkit-architecture-deep-dive-policy-engines-trust-and-sre-for/4510105)
  - [Shift-Left Governance for AI Agents](https://techcommunity.microsoft.com/blog/linuxandopensourceblog/shift-left-governance-for-ai-agents-how-the-agent-governance-toolkit-helps-you-c/4516481)
  - [Govern AI Agents on App Service with AGT](https://techcommunity.microsoft.com/blog/appsonazureblog/govern-ai-agents-on-app-service-with-the-microsoft-agent-governance-toolkit/4510962)
  - [AGT + Microsoft Agent Framework Better Together](https://devblogs.microsoft.com/agent-framework/governance-at-the-speed-of-agents-microsoft-agent-framework-and-agent-governance-toolkit-better-together)
  - GitHub: https://github.com/microsoft/agent-governance-toolkit

### 7. Defender for Cloud AI threat protection — “漏れた後” を SOC で拾う

- **GA**、Defender for Cloud / Defender for AI Services プランの一部
- 役割: **「フィルタが効いた瞬間」と「フィルタを抜けた挙動」を SOC に届けるための監視層**
  - Prompt Shields はリクエストを止める / 検出する
  - Defender for AI は **その検出を含めたインシデントを Defender XDR ポータルに送る**
  - つまり**ブロック装置の代替ではなく、ブロックを観測可能にする装置**
- 主要アラート (抜粋、いずれも MITRE タクティクス付き)
  - `AI.Azure_CredentialTheftAttempt` — モデル応答内に認証情報 (Credential Access / Lateral Movement / Exfiltration)
  - `AI.Azure_Jailbreak.ContentFiltering.BlockedAttempt` / `DetectedAttempt` — Prompt Shields によるブロック / 検出
  - `AI.Azure_Agentic_Jailbreak` — エージェントへの jailbreak 検出
  - (preview) `AI.Azure_Agentic_ASCIISmuggling` — ASCII smuggling 経由の indirect prompt injection (High)
  - (preview) `AI.Azure_Agentic_DOWVolumeAnomaly` — “wallet attack” 過剰利用攻撃
  - (preview) `AI.Azure_Agentic_InstructionLeakage` — system prompt 漏えい
  - (preview) `AI.Azure_LLMReconnaissance` / `AI.Azure_Agentic_LLMReconaissance` — capability / safety guardrail の reconnaissance
  - (preview) `AI.Azure_AnomalousToolInvocation` — エージェントの異常なツール呼び出し
  - (preview) `AI.Azure_Agentic_AccessFromSuspiciousIP` / `AccessFromAnonymizedIP` — 脅威インテリ IP / Tor からのアクセス
- 制約 (本記事で必ず書く)
  - **テキスト トークン専用** (画像・音声トークンはスキャン対象外)
  - 対応モデル: Azure OpenAI / Azure AI Model Inference の対応モデル
  - サブスクリプションレベルで有効化、必要ロールは Owner 相当
  - 30 日無料トライアル、最大 **75 Billion トークン**スキャンまで (超えると課金)
  - 商用 Azure / Azure Government / 21Vianet / **接続済み AWS アカウント**にも対応
- Sentinel / Defender XDR への流し込み
  - アラートは Sentinel など SIEM へストリーミング可能 ([Security alerts and incidents - Defender for Cloud](https://learn.microsoft.com/en-us/azure/defender-for-cloud/alerts-overview))
- 根拠 URL: [Overview - AI threat protection](https://learn.microsoft.com/en-us/azure/defender-for-cloud/ai-threat-protection) / [Alerts for AI services](https://learn.microsoft.com/en-us/azure/defender-for-cloud/alerts-ai-workloads)

### 8. 全体像 — build → CI → runtime → SOC をどう繋ぐか

- ASCII フロー図で 1 つにまとめる
- 各層と前回記事の対応を再確認
  - 前回記事 = **runtime 層 (Content Safety / Foundry filter / APIM)**
  - 本記事 = それ以外すべて (build/CI/runtime monitoring/SOC)
- 「Foundry でデプロイしていないモデル / 自前 HTTP backend」のケースだけ別パターンも添える (PyRIT は直接叩ける / Red Teaming Agent は対象外 / Defender は Azure OpenAI 経路でないと拾えない)

### 9. 設計でハマりやすい勘違い

- 1. **「Prompt Shields があるから Defender for AI は不要」**
  - 違う。Prompt Shields は止める、Defender for AI は **検出を SOC に流して相関する**
  - そもそも Defender for AI のアラート (`*.BlockedAttempt` 系) は **Prompt Shields の判定結果に依存している**
- 2. **「AI Red Teaming Agent があるから PyRIT を知らなくてよい」**
  - 違う。Foundry hosted prompt / container agent **以外** には Red Teaming Agent は使えない
  - 関数ツール、ブラウザ自動化、Computer Use、非 Azure モデル → **PyRIT 直接利用に落ちる**
- 3. **「Red Teaming Agent を回せば OWASP LLM / Agentic Top 10 をカバーできる」**
  - 違う。Microsoft の公式 Q&A でも明言されている (safety 中心、supply chain / model theft / infra misconfig は対象外)
  - 一部 (prompt injection / indirect injection / data poisoning 風プロンプト) は adversarial として再現可能
  - **OWASP Agentic Top 10 を runtime で押さえたいなら、別軸として Agent Governance Toolkit を入れる**のが Microsoft 公式の置き方
- 4. **「Risk and Safety Evaluators の `prohibited_actions` を model evaluation に使える」**
  - 違う。`prohibited_actions` / `sensitive_data_leakage` は **agent only** で、tool_calls が必要
- 5. **「Defender for AI を入れれば画像・音声攻撃も拾える」**
  - 違う。**text tokens only**
- 6. **「PyRIT を CI で回せば runtime ガードレール不要」**
  - 違う。CI は「自分が想定した攻撃」しかカバーしない。**未知攻撃は runtime + Defender で拾う**前提を捨てない

### 10. まとめ

- 前回の runtime ガードレール設計の **外側** にある責務を、Microsoft が用意しているツール群に対応付けた
- 結論として、ガードレール設計は次の 6 つを **誰が何を担当するか** で分解する
  1. 開発 / lab: **PyRIT** で手元の手探り red teaming
  2. CI / pre-deploy: **AI Red Teaming Agent + Risk and Safety Evaluators** で ASR とカテゴリスコアをゲートにする
  3. runtime (text レベル): **前回記事の Content Safety / Foundry filters / APIM**
  4. runtime (action レベル): **Agent Governance Toolkit (OSS)** で OWASP Agentic 系の tool 呼び出しガバナンス
  5. runtime monitoring: **Defender for Cloud AI threat protection** がアラートを生む
  6. SOC: **Defender XDR / Microsoft Sentinel** で相関・追跡する
- 「ガードレールを置く」ことと「ガードレールが効くかを継続的に確かめる」ことを **同じレベルで予算化する** ことが、結局のところ Microsoft 公式 4 層モデルが要求している姿

## 参考リンク (本文末用、暫定)

### Microsoft 公式 (build / CI / pre-deploy)

- [PyRIT](https://microsoft.github.io/PyRIT/)
- [AI Red Teaming Agent (preview)](https://learn.microsoft.com/en-us/azure/foundry/concepts/ai-red-teaming-agent)
- [Run AI Red Teaming Agent in the cloud (Foundry SDK)](https://learn.microsoft.com/en-us/azure/foundry/how-to/develop/run-ai-red-teaming-cloud)
- [Risk and Safety Evaluators](https://learn.microsoft.com/en-us/azure/foundry/concepts/evaluation-evaluators/risk-safety-evaluators)
- [Observability in Generative AI (pre/post-prod)](https://learn.microsoft.com/en-us/azure/foundry/concepts/observability)

### Microsoft 公式 (runtime — action レベル)

- [Introducing the Agent Governance Toolkit](https://opensource.microsoft.com/blog/2026/04/02/introducing-the-agent-governance-toolkit-open-source-runtime-security-for-ai-agents)
- [AGT Architecture Deep Dive](https://techcommunity.microsoft.com/blog/linuxandopensourceblog/agent-governance-toolkit-architecture-deep-dive-policy-engines-trust-and-sre-for/4510105)
- [Shift-Left Governance for AI Agents](https://techcommunity.microsoft.com/blog/linuxandopensourceblog/shift-left-governance-for-ai-agents-how-the-agent-governance-toolkit-helps-you-c/4516481)
- [AGT + Microsoft Agent Framework Better Together](https://devblogs.microsoft.com/agent-framework/governance-at-the-speed-of-agents-microsoft-agent-framework-and-agent-governance-toolkit-better-together)
- [Govern AI Agents on App Service with AGT](https://techcommunity.microsoft.com/blog/appsonazureblog/govern-ai-agents-on-app-service-with-the-microsoft-agent-governance-toolkit/4510962)
- GitHub: https://github.com/microsoft/agent-governance-toolkit

### Microsoft 公式 (runtime monitoring / SOC)

- [Overview - AI threat protection](https://learn.microsoft.com/en-us/azure/defender-for-cloud/ai-threat-protection)
- [Alerts for AI services](https://learn.microsoft.com/en-us/azure/defender-for-cloud/alerts-ai-workloads)
- [AI security recommendations reference](https://learn.microsoft.com/en-us/azure/defender-for-cloud/recommendations-reference-ai)
- [Security alerts and incidents - Defender for Cloud](https://learn.microsoft.com/en-us/azure/defender-for-cloud/alerts-overview)

### Microsoft 公式 (組織ガバナンス / 4 層モデル)

- [Governance and security for AI agents across the organization](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ai-agents/governance-security-across-organization)

### 前回記事

- [Azure中心で設計するLLMガードレール実践](https://zenn.dev/headwaters/articles/4ba708199f684e)

## frontmatter 想定 (publish 時)

```yaml
title: "Azure ガードレールの先 — PyRIT・Red Teaming Agent・Risk & Safety Evaluators・Defender for AI で “攻めて測って見張る”"
emoji: "🛡️"
type: "tech"
topics: ["azure", "security", "openai", "llm", "foundry"]
published: false
```

(前回記事で使われている `azure / security / openai / llm` を引き継ぎ、`apim` を `foundry` に差し替え)
