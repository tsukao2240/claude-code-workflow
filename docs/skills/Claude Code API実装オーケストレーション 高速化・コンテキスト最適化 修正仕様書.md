# Claude Code API実装オーケストレーション  
# 高速化・コンテキスト最適化 修正仕様書

## 1. 文書概要

| 項目 | 内容 |
|---|---|
| 文書状態 | Draft v3（レビュー反映版） |
| 対象 | 連続API実装オーケストレーション / APIレビューSkill |
| 目的 | 実行時間、Token消費、コンテキスト肥大、不要なAgent間反復の削減 |
| 基本方針 | 既存の安全境界・レビュー品質を維持したまま効率化する |
| 主な変更 | 計測、Context再利用、Agent間契約、Diff Fingerprint、決定論的Consolidator、探索制御、並列化、再レビュー軽量化、Prompt Cache最適化 |
| v2変更点 | Diff Fingerprint定義（4.4）、Findings契約拡張とSchema化（6.2）、Consolidator仕様（6.3）、PARTIAL分担の機械化（8.1）、再Review入力を累積Diffへ（10.2）、PASS確定条件（10.3）、進展なし判定の決定論化（10.4）、Full Validationとの順序（11.2）、Prompt Cache規約（12.0）、評価Fixture整備（16 Phase 1）、Phase 2/3入替 |
| v3変更点 | 進展なし判定の条件C削除（10.4）、Team Skill / Spec Reviewer完全分離とReviewer入力からTeam Skill Findings除外（7.1、9、10.1、10.2）、PARTIAL分担をCoverage Matrixへ静的化しcheckedAspectsは事後検証用へ（8.1）、Fingerprint生成スクリプトの単一実装・正規化ルール・ツールハッシュ照合（4.4） |

---

## 2. 背景

現行構成では、以下のワークフローを実行している。

```text
Ticket取得
  ↓
仕様探索
  ↓
Implementation
  ↓
Build / Test
  ↓
Team Backend API Skill
  +
Specification Reviewer
  ↓
Findings統合
  ↓
必要に応じて修正
  ↓
Build / Test
  ↓
再レビュー
  ↓
Host側独立検証
  ↓
Commit
```

安全性およびレビュー品質を重視した構成である一方、実運用では以下が発生している。

- Claude Codeによる探索時間が長い
- サブエージェントが必要以上に関連情報を調査する
- 修正後に同一仕様を再探索する
- Agent間の自然言語による情報引き継ぎが長大化する
- Review/Fix LoopでContextが増大する
- Fresh Reviewerが毎回Repositoryや仕様を再理解する
- Team Backend API SkillとSpecification Reviewerで探索が重複する
- Build / Test / Reviewの反復によりwall timeが増大する
- どの工程が実際のボトルネックか定量的に確認しづらい

本修正では、レビュー品質を下げることではなく、

```text
同じ情報を何度も探索しない
必要な情報だけ次工程へ渡す
LLMに判断させる必要がない部分は決定論的に制御する
```

ことを中心に改善する。

---

## 3. 非変更事項

以下の既存原則は変更しない。

### 3.1 Security Boundary

- Claude CodeはPodman Sandbox内で実行する
- `/workspace`以外への不要な書き込みを許可しない
- `/reference`はRead Onlyとする
- Git metadataをWorkerへ公開しない
- Git操作はWindows Host側だけが行う
- Workerへremote資格情報を渡さない
- push / PR作成 / mergeを自動実行しない
- SecretをRepository、ログ、Promptへ保存しない
- Reviewer系Agent（Team Backend API Skill / Specification Reviewer / Consolidator）は`/workspace`へ一切書き込まない
  - Build成果物・一時ファイルの生成も禁止する
  - 出力先は`/run/output/`配下のみとする

### 3.2 Human Gate

全Ticket完了後の最終差分確認は人間が行う。

```text
Claude Code
  ↓
Implementation / Validation / Review
  ↓
Host Commit
  ↓
全Ticket完了
  ↓
Human Review
  ↓
Human Push / PR
```

PR作成そのものは自動化しない。

### 3.3 Review Independence

Specification ReviewerはMain Agentとは別のFresh Contextで実行する。

実装時の試行錯誤、自己正当化、chain-of-thought等をReviewerへ渡さない。

### 3.4 Team Backend API Skill

既存Team Backend API SkillはSingle Source of Truthとして維持する。

そのロジックをSpecification Reviewerへ複製しない。

---

## 4. 改善方針

本修正ではContextを以下の3種類に分離する。

```text
Static Context
  ↓
Ticket Context
  ↓
Cycle Context
```

### 4.1 Static Context

複数Ticket・複数Reviewで原則変化しない情報。

例:

```text
Claude / Skill Instructions
Review Severity Definition
AI-generated Code Review Rules
Review Noise Control
Team Coding Convention
Team Backend Skill Coverage Matrix
共通のRepository構造説明
```

Static Contextは可能な限り安定したPrefixとして構成する。

Prompt Cacheを利用可能な場合、キャッシュ対象として扱う。

---

### 4.2 Ticket Context

1 Ticketの実行中は原則変化しない仕様情報。

例:

```text
Redmine
API IF
Design
ER
Specification Source Status
Ticket Acceptance Criteria
対象Endpoint
仕様上必要なRepository Pattern
```

Ticket ContextはContext Collectorが1回生成し、修正ループ中は原則再生成しない。

成果物:

```text
/run/output/context/specification-bundle.json
```

---

### 4.3 Cycle Context

ImplementationまたはFixによって変化する情報。

例:

```text
Current Diff
Changed Files
Current Tests
Build Result
Test Result
Team Backend Skill Findings
Previous Blocking Findings
Diff Fingerprint
```

成果物例:

```text
/run/output/context/cycle-001.json
/run/output/context/cycle-002.json
```

Cycle Contextだけを修正ごとに更新する。

---

### 4.4 Diff Fingerprint

Review PASSの有効性を機械的に判定するための識別子。

#### 定義

```text
Diff Fingerprint
  = SHA-256( Ticket Branch起点からの累積diff（unified形式、文脈行0、
             ファイルパス昇順、改行コード正規化済み） )
```

Git metadataはWorkerへ公開しないため、Worker内ではHost Orchestratorが
Ticket開始時に`/run/input/baseline/`へ配置したスナップショットとの比較で累積diffを生成する。

#### 生成スクリプト

Fingerprint生成はGitに依存せず、以下の専用スクリプト1つだけで行う。

```text
/run/tools/diff-fingerprint.py
```

- 入力は「ベースラインスナップショット」と「現在のworkspace」の2ディレクトリのみとする
- HostとWorkerは同一ファイルをそのまま実行する。Host側はPodman経由で実行するか、同一ファイルをHost上で実行する
- **重複実装を禁止する。** Host用・Worker用に別実装を持ってはならない。Git diffやIDEのdiff出力で代用してはならない
- 正規化ルールはスクリプト内に定数として持ち、本仕様書はスクリプトを正とする

#### 正規化ルール（スクリプト内で固定）

| 項目 | 扱い |
|---|---|
| 改行コード | LFへ正規化 |
| BOM | 除去 |
| 末尾改行 | 末尾に1つのLFを付与して統一 |
| File mode | 無視する |
| Binary file | 内容のSHA-256をdiff行の代わりに使用する |
| New file | 全行を追加として扱う |
| Deleted file | 全行を削除として扱う |
| Rename | 検出しない。Delete + New fileとして扱う |
| Generated file | 除外パターン（`controller.json`等）をスクリプト内に定数で保持し、Fingerprintに含めない |

Generated fileの除外パターンを仕様書側で個別に管理しない。
除外パターンの変更はスクリプトの変更として扱い、Host側のスクリプトハッシュ照合（下記）で検出する。

#### スクリプト改変の検出

`review-pass.json`にはFingerprintに加えて、使用したスクリプト自体のSHA-256を記録する。

```json
{
  "diffFingerprint": "sha256:...",
  "fingerprintToolHash": "sha256:...",
  "cycle": 2
}
```

Host Independent Validation時、HostはHost側保管のスクリプトハッシュと`fingerprintToolHash`を照合し、
不一致の場合はFingerprintの一致如何にかかわらずSUCCESSにしない。
Worker側でスクリプトが改変された場合の検出手段とする。

#### ライフサイクル

| タイミング | 担当 | 動作 |
|---|---|---|
| Implementation / Fix完了時 | Main Agent | 計算し`cycle-NNN.json`へ記録 |
| Review開始時 | Consolidator | 入力diffのFingerprintを再計算し、cycle記録と一致しない場合はFail Closed |
| Review PASS時 | Consolidator | PASSした時点のFingerprintを`review-pass.json`へ記録 |
| Host Independent Validation時 | Host Orchestrator | Worker成果物から独立に再計算し、`review-pass.json`と一致しない場合はSUCCESSにしない。スクリプトハッシュも照合する |

#### 原則

- Fingerprintが一致しないReview PASSは無効とする
- 生成ロジックはWorker / Hostで同一ファイルを使用し、重複実装を禁止する
- FingerprintはLLMに計算させず、決定論的スクリプトで算出する
- スクリプトハッシュが一致しないFingerprintは無効とする

---

## 5. Context Collector修正

### 5.1 基本原則

現行原則を維持する。

```text
Search Scope  = broad
Context Scope = narrow
```

ただし、検索結果をそのまま後続Agentへ渡してはならない。

Context Collectorは以下を実行する。

```text
広く検索
  ↓
候補を絞る
  ↓
必要部分のみ読む
  ↓
Sourceを記録
  ↓
Specification Bundleへ圧縮
```

---

### 5.2 Specification Bundle

以下を保持する。

```text
Ticket
API IF
Design
ER / DB
Explicit Requirements
Relevant Repository Pattern
Required Tests
Source Status
Source References
```

原資料全文をBundleへコピーしない。

API IFについては対象sheet/rangeのみを保持する。

設計書についても対象sectionのみを保持する。

---

### 5.3 Immutable化

Specification Bundle生成後は、以下の場合を除き再生成しない。

- Ticket情報自体が更新された
- 必須Referenceの不足が判明した
- Reviewerが`category: "INSUFFICIENT_EVIDENCE"`のFindingを出力した（6.2参照）
- Specification Source間の競合調査が必要になった
- 人間が再取得を指示した

単純な実装修正を理由に仕様探索を再実行しない。

`INSUFFICIENT_EVIDENCE` FindingはFix Loopではなく、Consolidatorが
Context Collectorの部分再実行（不足Referenceのみ追加取得）へルーティングする。
再生成後のBundleは`specification-bundle.json`を上書きせず、
`specification-bundle-v2.json`のようにversionを付与して保存する。

---

## 6. Agent間通信

### 6.1 基本原則

Agent間の自由な自然言語会話を主経路としない。

Agentは構造化成果物を介して連携する。

```text
Context Collector
    ↓
specification-bundle.json

Team Backend API Skill
    ↓
team-findings.json

Specification Reviewer
    ↓
spec-findings.json

Consolidator
    ↓
consolidated-findings.json

Main Agent
    ↓
Source変更
```

---

### 6.2 Findings契約

Findingは最低限以下を持つ。

```json
{
  "id": "SPEC-0003",
  "source": "SPEC_REVIEWER",
  "severity": "MAJOR",
  "category": "API_IF",
  "scope": "IN_SCOPE",
  "file": "src/Example/Controllers/ExampleController.cs",
  "location": {
    "symbol": "ExampleController.CreateAsync",
    "startLine": 42,
    "endLine": 58,
    "diffFingerprint": "sha256:..."
  },
  "specificationSource": {
    "type": "API_IF",
    "file": "/reference/if/example.xlsx",
    "sheet": "API001",
    "range": "B12:N35"
  },
  "problem": "...",
  "expectedBehavior": "...",
  "evidence": "..."
}
```

#### フィールド規約

| Field | 規約 |
|---|---|
| `id` | `<SOURCE>-<連番>`。Cycle内で一意 |
| `source` | `TEAM_SKILL` / `SPEC_REVIEWER` / `CONSOLIDATOR` |
| `severity` | `BLOCKER` / `MAJOR` / `MINOR` / `NIT` |
| `category` | Coverage Matrix（8章）のCategoryに加え、`INSUFFICIENT_EVIDENCE`を許可する |
| `scope` | `IN_SCOPE` / `RELATED_EXISTING_RISK` / `OUT_OF_SCOPE`（7.3参照） |
| `file` | Repositoryルートからの相対パス |
| `location.startLine` / `endLine` | 変更後ファイルの行範囲。Fingerprintと組で解釈する |
| `location.diffFingerprint` | このFindingが対象としたdiffのFingerprint（4.4参照） |

`INSUFFICIENT_EVIDENCE`はReviewerが「仕様根拠が不足しており判定できない」場合にのみ使用する。
この場合`severity`は`MAJOR`固定、`specificationSource`には不足しているReferenceの期待所在を記述する。

JSON Schemaは以下に配置し、15章の「Structured Result不正」判定に使用する。

```text
/run/schema/finding.schema.json
/run/schema/findings-file.schema.json
```

ReviewerからMain Agentへ長大なレビュー文章を渡さない。

---

### 6.3 Consolidator

Team Backend API SkillとSpecification Reviewerの出力を統合し、Fix Loopへの入力を確定する。
LLMを使用せず、決定論的スクリプトとして実装する。

#### 入力

```text
team-findings.json
spec-findings.json
cycle-NNN.json
```

#### 処理

```text
Schema検証
  ↓
Fingerprint照合（両Findingsが同一diffを対象としているか）
  ↓
重複判定
  ↓
Severity解決
  ↓
Scope Filter
  ↓
INSUFFICIENT_EVIDENCEルーティング
  ↓
consolidated-findings.json
```

#### 重複判定

以下がすべて一致する場合、同一Findingとみなす。

```text
file
category
location行範囲の重なり（startLine..endLineが1行以上重複）
```

重複した場合は1件に統合し、`duplicateOf`として両方の`id`を保持する。
統合件数は13.3の`Duplicate Finding count`へ記録する。

#### Severity解決

重複Findingの`severity`が異なる場合、より高いSeverityを採用する。
`source`は`CONSOLIDATOR`とし、元Findingの両方を`mergedFrom`へ保持する。

矛盾するFinding（一方が「A すべき」、他方が「A すべきでない」）はConsolidatorでは解決せず、
`category: "CONFLICT"`、`severity: "BLOCKER"`として出力し、Fail Closedとする。

#### Scope Filter

`scope: "OUT_OF_SCOPE"`のFindingは`consolidated-findings.json`の`deferred`配列へ移し、
Fix Gate判定（10.4）から除外する。破棄はしない。

#### 出力

```json
{
  "cycle": 2,
  "diffFingerprint": "sha256:...",
  "gate": {
    "blocker": 1,
    "major": 0,
    "pass": false
  },
  "findings": [ ... ],
  "deferred": [ ... ],
  "insufficientEvidence": [ ... ]
}
```

---

## 7. Reviewer探索制御

### 7.1 初期入力

Specification Reviewerへ最初に渡す情報は以下に限定する。

```text
Static Review Instructions（Coverage Matrixを含む）
Specification Bundle
Current Diff
Changed Files
Relevant Tests
必要最小限の関連コード
```

Team Backend Skill Findingsは渡さない。
Team Backend API SkillとSpecification Reviewerは並列実行（9章）のため、
Reviewer起動時点でTeam Skillの結果は存在しない。
重複回避はStatic Contextに含めるCoverage Matrix（8章）だけで行う。

Repository全体を最初から再探索しない。

---

### 7.2 On-Demand Retrieval

Reviewerが判定に追加情報を必要とした場合だけ追加探索を許可する。

例:

```text
Finding候補
  ↓
Evidence不足
  ↓
必要なsymbol/fileを特定
  ↓
対象箇所だけRead
  ↓
判定
```

以下は禁止する。

```text
「念のため」広範囲を読む
Repository全体から類似コードを大量検索する
Findingと無関係な設計を調査する
将来拡張を検討する
```

---

### 7.3 Scope Boundary

Reviewerは今回のdiff中心に評価する。

```text
IN_SCOPE
RELATED_EXISTING_RISK
OUT_OF_SCOPE
```

OUT_OF_SCOPE問題を今回のFix Loopへ含めない。

---

## 8. Team Backend API Skillとの分担

Coverage Matrixを機械的に利用する。

例:

| Category | Team Skill Coverage | Specification Reviewer |
|---|---|---|
| Controller Structure | YES | 原則確認しない |
| Naming | YES | 原則確認しない |
| Validation | PARTIAL | 仕様固有部分のみ |
| API IF | NO | 確認する |
| Redmine Requirement | NO | 確認する |
| DB Transaction | PARTIAL | 不足部分のみ |
| Test Quality | PARTIAL | Requirement基準で確認 |
| AI-generated Hygiene | NO | 確認する |

Team Skillが`YES`の項目をSpecification Reviewerが再探索しない。

### 8.1 PARTIAL項目の分担方法

`PARTIAL`項目は「Reviewerが不足部分を判断する」方式にしない。
判断をLLMに委ねると重複探索が再発するためである。

また、ReviewerがTeam Skillの実行結果を読んで分担を決める方式にもしない。
両者は並列実行（9章）であり、Reviewer起動時点でTeam Skillの結果は存在しないためである。

分担はCoverage Matrixに**静的に**Aspect単位で列挙し、Static Contextとして両Agentへ渡す。
Reviewerは自分の担当Aspectだけを確認し、Team Skill担当のAspectを確認しない。

例（Validation）:

| Aspect | 担当 |
|---|---|
| RequiredAttribute | Team Skill |
| StringLength | Team Skill |
| 型・フォーマット | Team Skill |
| 仕様固有の値域（API IF記載） | Specification Reviewer |
| 仕様固有の相関チェック | Specification Reviewer |

Aspect表はCategoryごとにCoverage Matrixへ含め、Ticketごとに変更しない（12.0参照）。

#### checkedAspectsによる事後検証

Team Backend API Skillは`team-findings.json`に、実際に確認したAspectを機械可読で出力する。

```json
{
  "checkedAspects": [
    { "category": "Validation", "aspect": "RequiredAttribute", "files": ["..."] },
    { "category": "Validation", "aspect": "StringLength", "files": ["..."] },
    { "category": "DB Transaction", "aspect": "TransactionScope", "files": ["..."] }
  ]
}
```

これはReviewerへの入力ではなく、Consolidatorが以下を事後検証するために使う。

```text
Coverage MatrixでTeam Skill担当のAspectが checkedAspects に含まれているか
  ↓
欠落がある場合
  → category: "COVERAGE_GAP", severity: "MAJOR" のFindingを Consolidator が生成
  → Fail Closed（Team Backend Skill異常として扱う）
```

Team Skillが担当Aspectを確認しなかったことを、Reviewer側で補うのではなく、異常として検出する。

### 8.2 Coverage Matrix改善

同一Findingの重複検出率（6.3で統合された件数）を計測し、Coverage Matrixの改善材料とする。

---

## 9. Review Pipeline並列化

Team Backend API SkillとSpecification Reviewerは、同一diffに対して並列実行する。

```text
                  ┌─ Team Backend API Skill ─┐
Latest Diff ──────┤                          ├─ Consolidator
                  └─ Specification Reviewer ─┘
```

両者は完全に分離し、一方の出力を他方の入力にしない。
Reviewer側の重複回避はCoverage Matrix（Static Context）だけで行う（7.1、8.1参照）。

以下の場合だけ、直列実行へFallbackする。この場合も入力定義は変えない。

- 実環境上、同時実行がResource Limitを超える場合
- 並列実行により同一Workspaceへの競合アクセスが発生する場合

直列実行時もReviewerへTeam Skill Findingsを渡してはならない。
並列・直列でReviewerの入力と判定結果が変わらないことを保証する。

ReviewerとTeam Skillはいずれもコードを書き換えない。
`/workspace`への書き込み禁止は3.1で規定し、並列化の前提条件とする。

両者は同一の`cycle-NNN.json`（同一Fingerprint）を入力とし、
Consolidatorが両出力のFingerprintを照合する。不一致の場合はFail Closedとする。

---

## 10. Review / Fix Loop

### 10.1 初回Review

初回は通常Reviewを行う。

入力:

```text
Specification Bundle
Current Diff
Relevant Code
Tests
```

Team Skill Resultは入力に含めない（7.1参照）。

---

### 10.2 再Review

修正後はFresh Reviewerを起動する。

#### 基本入力

```text
Specification Bundle（固定）
累積Diff（Ticket Branch起点から現在までの全Diff）
変更影響範囲
最新Tests
```

「前回Review以降のDiff」だけを入力にしてはならない。
修正が新たな仕様違反を生んだ場合、前回指摘に含まれない箇所を見落とすためである。

#### 補助入力

```text
前回BLOCKER / MAJOR（解消確認リスト）
```

前回Findingsは「解消されたかの確認対象」としてのみ渡す。
Reviewerは前回Findingsの正当性を前提としてはならず、
累積Diffと Specification Bundleに基づき独立に判定する。
実装側の修正理由・説明文は渡さない（3.3参照）。

#### 削減対象

再Reviewで削減するのは以下である。

```text
仕様探索（Bundleを再利用）
Repository全体の再理解（7.1の初期入力に限定）
```

累積Diffの入力量は削減しない。

ReviewerそのものはFresh Contextとするが、原仕様を再探索する必要はない。

---

### 10.3 Review PASSの扱い

diffが変化した場合、以前のPASSは無効とする。

判定はDiff Fingerprint（4.4）の一致で機械的に行う。
`review-pass.json`のFingerprintと現在の累積DiffのFingerprintが一致しない場合、PASSは存在しないものとして扱う。

Review PASSが有効となる条件を以下に定める。

```text
consolidated-findings.json の gate.pass = true
かつ
gate判定時のFingerprint = Full Validation（11.2）実行時のFingerprint
かつ
Full Validation成功
```

Fast Validation（11.1）のみを通過した状態でのPASSは暫定PASSとし、
Full Validation成功後にFingerprintが一致して初めて確定PASSとする。

ただし、

```text
以前の仕様探索結果
以前に確定したSpecification Bundle
```

まで無効化する必要はない。

---

### 10.4 Fix Loop終了条件

以下を明示する。

```text
BLOCKER = 0
MAJOR   = 0
```

の場合、Review PASS。

MINOR / NITだけではFix Loopを継続しない。

最大Review Cycleを設定する。

初期値:

```text
max_review_cycles = 3
```

#### 進展なし判定

以下のいずれかを満たす場合、上限前でも停止しFail Closedとする。
判定はConsolidatorが決定論的に行い、LLMに委ねない。

```text
条件A: 同一Finding残存
  前Cycleと現Cycleの consolidated-findings.json において、
  (file, category, 行範囲の重なり) が一致する BLOCKER / MAJOR が
  2 Cycle連続で残存している

条件B: Diff不変
  Fix後のDiff Fingerprintが前CycleのFingerprintと一致している
  （Main Agentが修正を行わなかった）
```

条件Aの行範囲比較は、Fix後に行番号がずれることを考慮し、
`file` + `category` + `location.symbol` の一致を優先し、symbolが無い場合のみ行範囲で比較する。

BLOCKER / MAJORの**件数**を進展判定に使わない。
件数が減らなくても、前回指摘が解消され別の問題が新たに検出されたケースは正常な進行であり、
件数比較ではそれを「進展なし」と誤判定するためである。

毎Cycle異なるFindingが出続けるケース（入れ替わり）は条件A・Bでは停止しない。
これは`max_review_cycles`による上限停止に委ねる。

---

## 11. Build / Test最適化

安全性を変更しない範囲で、検証を以下に分類する。

```text
Fast Validation
Full Validation
Independent Host Validation
```

### 11.1 Fast Validation

Fix Loop途中では可能な場合に利用する。

例:

```text
対象Project Build
変更対象Test
Formatter
Analyzer
```

Repositoryルール上許されない場合は使用しない。

Fast Validation通過はFull Validation通過を意味しない。
Fast Validationのみで得たReview PASSは暫定扱いとする（10.3参照）。

---

### 11.2 Full Validation

TicketをSUCCESS候補にする前には従来通りFull Validationを行う。

```text
Repository指定Build
API回帰Test一式
Team Backend API Skill
Specification Reviewer
controller.json検証
```

#### 実行順序

```text
Fix Loop（Fast Validation + 暫定PASS）
  ↓
Full Validation（Build + 回帰Test）
  ↓
Fingerprint一致確認（暫定PASS時のFingerprint = 現在のFingerprint）
  ↓
確定PASS
  ↓
Host Independent Validation
```

Full Validationで失敗した場合、修正後のdiffはFingerprintが変わるため、
暫定PASSは自動的に無効となりFix Loopへ戻る。
Full Validation後にReviewを再実行しない経路を作ってはならない。

---

### 11.3 Independent Host Validation

既存のHost側独立検証は維持する。

Worker自己申告だけでSUCCESSとしない。

高速化を理由に以下を削除しない。

```text
Host側Build結果取得
Host側Test結果取得
Independent Review
Verified Tree SHA
Commit Tree一致確認
```

---

## 12. Prompt Cache方針

Prompt Cacheが利用可能な場合、Static Contextを可能な限り同一Prefixとして構成する。

### 12.0 前提

Claude Code経由でAgentを実行する場合、Prompt Cacheの制御はClaude Code側に委ねられる。
本仕様で制御できるのはPromptの安定性だけである。
以下は「Cacheを直接操作する」仕様ではなく、「Cacheが効く構造を壊さない」ための規約である。

#### 禁止事項

```text
Skill / CLAUDE.md / Reviewer Role定義をTicketごとに書き換えない
Specification BundleをSkill本文へ埋め込まない
Static Contextに日時・Ticket ID・Cycle番号を含めない
Static Contextの記述順序をCycleごとに変えない
Findings / Diff / Test結果をStatic Contextより前に置かない
```

#### 配置順序

```text
[Static Context]     Skill Instructions / Rules / Coverage Matrix
[Ticket Context]     Specification Bundle
[Cycle Context]      Diff / Findings / Test Result
```

### 12.1 Cache対象候補

```text
Skill Instructions
Reviewer Role
Severity Rules
AI-generated Code Rules
Noise Control Rules
Coverage Matrix
Team Coding Convention
共通Repository Guidance
```

### 12.2 Cache対象外

以下はTicketまたはCycleごとに変化する。

```text
Redmine Ticket
Specification Bundle
Current Diff
Changed Files
Test Result
Findings
Runtime Evidence
```

### 12.3 原則

Prompt CacheをContext削減手段として扱わない。

```text
Prompt Cache
  = 再入力コスト / latency削減

Context Optimization
  = Claudeへ与える情報量そのものの削減
```

両者を別の最適化として扱う。

---

## 13. Observability

高速化前に必ず計測を追加する。

### 13.1 Phase Metrics

最低限以下を記録する。

```text
phase_name
start_time
end_time
wall_time_ms
model
input_tokens
output_tokens
cache_read_tokens
cache_write_tokens
tool_call_count
files_read
bytes_read
```

取得不能な項目は`UNKNOWN`として扱う。

推測値を記録しない。

`cache_read_tokens` / `cache_write_tokens`は実行環境によって取得できない場合がある。
その場合は`UNKNOWN`とし、Phase 6（16章）のCache Hit率計測は
「取得可能な環境でのみ実施する」ものとする。
取得不能な場合でも12章の配置規約は適用する。

---

### 13.2 Ticket Metrics

```text
Ticket ID
Total Wall Time
Specification Discovery Time
Implementation Time
Build Time
Test Time
Team Skill Time
Specification Review Time
Fix Time
Review Cycle Count
Total Tool Calls
Total Files Read
Total Input Tokens
Total Output Tokens
Cache Hit / Read
```

---

### 13.3 Review Metrics

Production評価用に以下も保持する。

```text
BLOCKER count
MAJOR count
MINOR count
NIT count
False BLOCKER / MAJOR count
Duplicate Finding count
Team Skill overlap count
Human additional findings
Human rejected findings
```

---

## 14. 実行レポート例

```text
Ticket 161

Context Collector
  Time        : 42s
  Files Read  : 6
  Input Token : 18,240
  Cache Read  : 4,100      # Static Context（Skill Instructions等）のヒット分

Implementation
  Time        : 96s
  Tool Calls  : 14

Build
  Time        : 38s

Test
  Time        : 121s

Team Skill
  Time        : 54s

Spec Reviewer (Cycle 1)
  Time        : 71s
  Input Token : 24,180
  Cache Read  : 11,250     # Static Context + Specification Bundle

Spec Reviewer (Cycle 2)
  Time        : 48s
  Input Token : 19,600
  Cache Read  : UNKNOWN    # 取得不能な場合の例

Review Cycle
  Count       : 2
  Stop Reason : PASS

Diff Fingerprint
  Final       : sha256:3f9a...
  Host Verify : MATCH

Total
  Time        : 7m42s
```

Cache Readは同一Ticket内の2回目以降、または前TicketとStatic Contextを共有する場合に発生する。
Ticket固有情報はCache対象外（12.2）のため、Context CollectorのCache ReadがInput Tokenに近づくことはない。

これによりボトルネックを実測で判断する。

---

## 15. Fail Closed

既存Fail Closed原則を維持する。

以下の場合は高速化処理を優先せず停止する。

```text
Specification Bundle不足
Required Reference欠損
Source競合
Team Backend Skill異常
Reviewer異常
Structured Result不正（JSON Schema検証失敗）
Finding CONFLICT（6.3）
COVERAGE_GAP（8.1）
Fingerprint Tool Hash不一致（4.4）
BLOCKER / MAJOR未解消
Fix Loop進展なし（10.4）
max_review_cycles到達
Diff Fingerprint不整合
Test失敗
Build失敗
Host Verification失敗
```

Cache missや最適化機能の失敗を理由に検証を省略してはならない。

最適化機能が利用不能な場合は、原則として従来経路へFallbackする。

---

## 16. 導入手順

以下の順序で導入する。

### Phase 1: Measurement

既存挙動を変更せず計測のみ追加する。

取得対象:

```text
Wall Time
Token
Tool Call
Files Read
Review Cycle
Cache
```

少なくとも複数TicketでBaselineを取得する。

#### 評価用Fixtureの整備

18章の`BLOCKER/MAJOR Recall`を測定するには正解セットが必要である。
Phase 1で以下を整備する。

```text
過去Ticketから3件以上を選定
各Ticketについて人間が確定したBLOCKER / MAJOR一覧を作成
Finding契約（6.2）と同形式で /run/fixture/<ticket>/expected-findings.json へ保存
```

Fixtureが無い状態でPhase 4以降へ進まない。

---

### Phase 2: Structured Agent Handoff

Agent間の主要な引き継ぎをJSON等の構造化成果物へ変更する。

```text
finding.schema.json 策定
Diff Fingerprint生成スクリプト
Consolidator（決定論的）
```

Specification Bundleの保存形式もここで確定する。

---

### Phase 3: Specification Bundle Reuse

Context Collector結果をTicket単位で保存する。

Fix Loopで再利用できることを確認する。

Phase 2の構造化形式に依存するため、Phase 2の後に実施する。

---

### Phase 4: Reviewer Retrieval制御

Reviewerを、

```text
Repositoryを広く探索する方式
```

から、

```text
与えられたEvidenceを基本に判定し、
不足した場合だけ追加Readする方式
```

へ変更する。

---

### Phase 5: Parallel Review

Team Backend API SkillとSpecification Reviewerを並列実行する。

---

### Phase 6: Prompt Cache最適化

Static Prefixを安定させ、Cache Hit率を計測する。

---

### Phase 7: Review Loop最適化

2回目以降のReviewer入力を、

```text
固定Specification Bundle
累積Diff
影響範囲
前回Blocking Findings（解消確認リストとして）
```

中心へ変更する（10.2参照）。

Phase 7の前後でPhase 1のFixtureを用いたRecallを比較し、
悪化した場合はPhase 7を採用しない。

---

## 17. Acceptance Criteria

本修正は以下を満たした場合に受入可能とする。

```text
[ ] 既存Security Boundaryを変更していない
[ ] push / PR / mergeが自動化されていない
[ ] 最終Human Gateが維持されている
[ ] Specification BundleをTicket内で再利用できる
[ ] 修正だけを理由にReference全探索を再実行しない
[ ] Agent間で構造化成果物を利用する
[ ] Reviewerが無条件にRepository全体を探索しない
[ ] 必要時のみ追加Evidenceを取得する
[ ] Team Skill Coverage YES項目をReviewerが重複実装しない
[ ] Team SkillとSpec Reviewerを安全に並列実行できる
[ ] Fresh Reviewer原則が維持される
[ ] 再Reviewの基本入力が累積Diffである
[ ] Diff変更時に以前のReview PASSを流用しない
[ ] Review PASS無効化がDiff Fingerprint一致で機械的に行われる
[ ] Full Validation後にFingerprintが一致しないPASSを確定PASSとしない
[ ] FindingsがJSON Schemaで検証される
[ ] Consolidatorが決定論的に重複排除・Severity解決を行う
[ ] INSUFFICIENT_EVIDENCEがFix LoopではなくContext Collector再実行へ流れる
[ ] Team Skill PARTIAL項目の分担がCoverage Matrixに静的にAspect単位で列挙されている
[ ] ReviewerがTeam Skill Findingsを入力として受け取らない
[ ] Consolidatorが checkedAspects で Team Skill の担当Aspect欠落を検出する
[ ] Diff Fingerprint生成スクリプトが単一実装で、Host / Worker が同一ファイルを実行する
[ ] Host が fingerprintToolHash を照合する
[ ] BLOCKER / MAJORのみFix Gateとする
[ ] Review Loopに明示的な最大回数がある
[ ] 進展なし判定が決定論的条件で行われる
[ ] Prompt Cache有無でReview判定結果の意味が変化しない
[ ] Ticket単位でphase別wall timeを計測できる
[ ] Token / Tool Call / Files Readを可能な範囲で取得できる
[ ] Cache Read / Writeを取得可能な場合記録する
[ ] Host側独立検証が維持される
[ ] Worker自己申告だけでSUCCESSにならない
[ ] 最適化失敗時に安全な従来経路へFallbackできる
```

---

## 18. 評価方法

修正前後で同等のTicketまたは過去Fixtureを用いて比較する。

`BLOCKER/MAJOR Recall`および`False BLOCKER/MAJOR`は、
Phase 1で整備した`expected-findings.json`との照合で算出する。
照合は6.3の重複判定と同一ルール（file + category + symbol / 行範囲）を用いる。

最低限以下を比較する。

| Metric | Before | After |
|---|---:|---:|
| Total Wall Time | | |
| Context Collector Time | | |
| Review Time | | |
| Review Cycle Count | | |
| Tool Calls | | |
| Files Read | | |
| Input Tokens | | |
| Cache Read Tokens | | |
| BLOCKER/MAJOR Recall | | |
| False BLOCKER/MAJOR | | |
| Human Additional Findings | | |

高速化しても重大Findingの検出率が明確に悪化する場合は採用しない。

---

## 19. 最終アーキテクチャ

```text
Windows Host Orchestrator
        │
        ├─ Preflight
        │
        ├─ Ticket Branch
        │
        ▼
Podman Worker
        │
        ├─ Context Collector
        │      ↓
        │  Specification Bundle
        │      │
        │      └───────────────┐
        │                      │
        ├─ Main Agent          │
        │   Implementation     │
        │        ↓             │
        │   Build / Test       │
        │        ↓             │
        │   Latest Diff        │
        │        │             │
        │    ┌───┴─────────────┴─────┐
        │    │                       │
        │    ▼                       ▼
        │ Team Backend Skill    Spec Reviewer
        │    │                       │
        │    └─────────┬─────────────┘
        │              ▼
        │     Consolidated Findings
        │              │
        │        BLOCKER / MAJOR?
        │          │           │
        │         YES          NO
        │          │           │
        │          ▼           ▼
        │       Main Fix    暫定PASS
        │          │           │
        │    Fast Validation   │
        │          │           │
        │    Fresh Re-Review   │
        │    (累積Diff)        │
        │                      ▼
        │             Full Validation
        │                      │
        │          Fingerprint一致確認
        │                      │
        │                  確定PASS
        │
        ▼
Host Independent Validation
        │
        ├─ Build
        ├─ Full Regression Test
        ├─ Independent Review
        ├─ Generated Artifact Verification
        ├─ Diff Fingerprint再計算 / review-pass.json照合
        └─ Verified Tree SHA
        │
        ▼
Host Commit
        │
        ▼
Next Ticket
        │
        ▼
Human Gate
```

---

## 20. 最終原則

本修正の目的はClaude Codeにより多く考えさせることではない。

```text
必要な情報を
必要なAgentへ
必要なタイミングで
必要な量だけ渡す
```

ことを目的とする。

特に以下を基本原則とする。

```text
Search Broadly
Carry Narrowly
Reuse Stable Context
Refresh Mutable Context
Prefer Structured Handoffs
Avoid Duplicate Exploration
Parallelize Independent Checks
Bound Every Loop
Decide Deterministically Where Possible
Measure Before Optimizing
Human Owns Final Change
```

安全性・レビュー品質を維持した上で、Agentが同じ情報を繰り返し探索・解釈する処理を削減し、API実装からHuman Gate到達までのwall timeを短縮する。