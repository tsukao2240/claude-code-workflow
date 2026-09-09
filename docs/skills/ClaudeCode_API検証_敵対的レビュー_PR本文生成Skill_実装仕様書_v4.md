# Claude Code API検証・敵対的レビュー・PR本文生成Skill 実装仕様書 v4

## 1. 目的

既存の「Redmine情報からプロジェクト既定テンプレートに沿ったPR本文を生成するSkill」を拡張し、API実装後の仕様照合、敵対的レビュー、実行確認、PR本文ドラフト生成までを一つの明示的ワークフローとして実行する。

PRそのものは作成しない。最終的なPR作成は必ず人間が行う。

## 2. 絶対条件

### 2.1 既存PRテンプレートを変更しない

セクション、見出し、順序、表現、Markdown構造、既存の入力欄のいずれも変更しない。テンプレート改善を理由とした独自変更も禁止。

PR本文に入れる情報は既存欄へだけ投入する。既存欄へ対応しない内部情報はSkillの実行結果として別途表示してよいが、テンプレートへ新規セクションを追加しない。

### 2.2 PR作成は禁止

Skillの終了点は「完成したPR本文ドラフト」。以下を実行しない。

``` text
gh pr create
gh pr merge
GitHub REST/GraphQL によるPR作成・更新・merge
既存のPR作成script
```

既存SkillにPR作成処理が存在する場合は、本Skillの実行経路からそこへ到達しないことを確認する。`gh`はread操作と既存運用上のpushには使ってよい。

### 2.3 SharePointへアクセスしない

API IFは人間が事前にダウンロードし、Read Onlyの`/reference/if`へ配置する。SharePoint IFリンクは既存テンプレートの手動欄に人間が貼る。ClaudeはURLを検索・生成・推測しない。

### 2.4 明示起動のみ

本Skillはユーザーが`/<skill名>`で起動したときだけ実行する。

Claude CodeのSkillはdescriptionが会話内容に一致するとClaudeが自動起動できるため、SKILL.mdのfrontmatterで禁止する。

``` yaml
disable-model-invocation: true
```

global Hook、SessionEnd Hook、Stop Hook、無関係なcommand Hookへ登録しない。既存Hookは今回の目的に不要なら変更しない。

## 3. 実行環境

正規実行環境は別紙「Claude Code 安全実行環境構築仕様書」で定義したPodman環境。

``` text
/workspace/repo-backend   R/W
/reference                RO (Podman/OS側で強制)
```

Skill側にも変更禁止を記載するが、Skillの指示をセキュリティ境界とはみなさない。WindowsネイティブClaudeからの実行を正規経路としない。

### 3.1 起動時環境チェック

Skillは処理を始める前に、scriptで以下を実際に確認する。

``` text
[ ] /workspace/repo-backend が存在し、git repositoryである
[ ] /reference が存在する
[ ] /reference 直下へのファイル作成が失敗する（touch等で実際に試す）
[ ] Excel解析ツールが呼び出せる
[ ] git の user.name / user.email が設定されている（WIP commitに必要）
```

1つでも満たさない場合は`ENV_PRECHECK_FAILED`として即時停止する。

## 4. 入力情報

### 4.1 Redmine

Ticket ID、Subject、Description、Notes/Comments、関連チケット、その他既存Skillが現在参照している情報。

Notes/Commentsを軽視しない。Descriptionより後に仕様変更が記載されている可能性を考慮する。

Ticket IDの取得、Redmineへのアクセス方法・認証は既存Skillのものをそのまま使う。本Skillで新しいRedmineクライアントや認証情報の読み込み方法を実装しない。

### 4.2 API IF

`/reference/if/`のExcel。読み取りは環境構築仕様書6.1のExcel解析ツールで行い、Skill内で独自のxlsx解析を実装しない。

確認対象例: endpoint、HTTP method、path/query/header、request、response、required/optional、validation、status code、error response、field type、nullability、remarks。

対象IFを一意に特定できない場合は推測してレビューを継続しない。IFファイル名だけで仕様を推測しない。

### 4.3 設計・ER

`/reference/design/`、`/reference/er/`。人間が必要資料を配置する。

受け入れる形式:

``` text
.md / .txt   そのまま読む
.xlsx        Excel解析ツールで読む
.pdf         テキスト抽出できる場合のみ
```

`.docx`、`.pptx`、`.drawio`、画像等は読めない。必要なら人間が上記形式へ変換して配置する。変換されていない資料は`PARSE_FAILED`とし、読めたふりをしない。

確認対象: 対象機能の設計、DB table、column、relation、constraint、transaction、domain rule。

将来doc repositoryをRead Only mountする場合も、検索対象を追加するだけでReviewer構造は変更しない。

### 4.4 実装リポジトリ

`/workspace/repo-backend`から、git diff、changed files、tests、CLAUDE.md、coding conventions、関連する既存実装、同種API、naming/style patternsを取得する。

### 4.5 検索除外

以下はSpecification Bundleへ含めず、Grep/Read対象からも除外する。

``` text
.env, .env.*
*.pem, *.key, *.p12
credentials*, secrets*
application-local.*, application-secret.* 等のprofile固有設定
~/.claude/ 配下
.git/ 配下
```

除外パターンはproject側の規約に合わせて追加できるようにし、Skill本体へハードコードしない。

### 4.6 Source Status

各仕様ソースについて「記載がない」と「取得・解析できなかった」を区別する。

``` text
FOUND         対象仕様を特定・解析できた
NOT_REQUIRED  今回の変更では不要と合理的に判断できる（理由必須）
NOT_FOUND     必要と判断したが発見できない
AMBIGUOUS     候補が複数あり一意に確定できない
PARSE_FAILED  ファイルはあるが必要内容を解析できない
UNAVAILABLE   外部サービス障害・権限等で取得不能
```

`NOT_FOUND`、`AMBIGUOUS`、`PARSE_FAILED`、必須ソースの`UNAVAILABLE`を「問題なし」と解釈しない。「情報がないので問題なし」と判定しない。

## 5. 構成

### 5.1 責務と実体

``` text
責務                          実体
----------------------------  ------------------------------------------
Orchestrator                  SKILL.md（手順のみ。判定ロジックは持たない）
環境チェック                   scripts/precheck
Ticket Resolver               既存Skillの処理を呼ぶ
Redmine Collector             既存Skillの取得処理をそのまま使用
Review Artifact Builder       scripts/build-artifacts（6.2参照）
Context Collector             .claude/agents/context-collector.md
Adversarial Reviewer          .claude/agents/spec-reviewer.md
Team Skill Runner             .claude/agents/team-skill-runner.md
Reviewer出力検証               scripts/validate-review
Verification Loop             SKILL.md の手順 + scripts/（cycle数管理、SHA照合）
Runtime Evidence Collector    scripts/（request実行とredact）
Existing PR Body Generator    既存Skillの処理をそのまま使用
```

方針:

-   判定・記録・検証のように決定的であるべき処理はscriptsに置き、Claudeの読解に依存させない
-   agentsは`model:`と`tools:`をfrontmatterで固定する
-   SKILL.mdはどのscript/agentをどの順に呼ぶかだけを書く
-   一つの巨大promptへ全責務を押し込まない

### 5.2 配置場所

本Skill、agents、scriptsは個人用として、環境構築仕様書5.3で永続化する`~/.claude/`配下に置く。

実装repo内の`.claude/`には置かない。Mutagenで Windows側へ同期され、チームリポジトリへのコミット対象になるため。既存のPR本文生成SkillやTeam Backend API Skillがrepo内にある場合、それらは動かさない。

### 5.3 モデル

``` text
Main Agent        Opus（起動側設定に依存。Skillからは指定できない）
Context Collector Sonnet
Reviewer          Sonnet
Team Skill Runner Team Skillの要件に従う
```

Main Agentのモデルは`--model`または`settings.json`で決まる。Opusで運用したい場合は起動側で担保する。

Reviewerを最初からOpusにする必要はない。ただしReviewerが見る情報はすべてCollectorが抽出したSpecification Bundleに限られる。見逃しが確認された場合は、まずBundleに該当仕様が含まれていたかを確認し、含まれていなければCollector側を先に改善する。モデル強化の順序はCollector、Reviewerの順。

モデル名・設定キーを推測しない。現行Claude Codeで指定方法が変わっている場合は公式仕様に従う。

### 5.4 ツール制限

``` yaml
# context-collector.md / spec-reviewer.md
model: sonnet
tools: Read, Grep, Glob
```

CollectorとReviewerにBash、Edit、Writeを与えない。Bashがあれば`echo >`や`git checkout`で書けるため、Edit/Writeを外すだけでは不十分。

CollectorとReviewerが必要とするdiff、test結果、Excel抽出結果は、6.2のscriptが事前に一時領域へ生成し、Read/Grepで読ませる。

Team Skill RunnerはTeam Backend API Skillが要求するツールに従う。Bashが必要な場合は許可するが、Edit/Writeは与えない。その場合のRead Only境界はPodmanのmountであることを最終報告の`Security Exceptions`に記載する。

## 6. ワークフロー

``` text
Main Agent (Opus)
  API実装 / test / lint
      ↓
ユーザーが /<skill名> で起動
      ↓
起動時環境チェック (3.1)
      ↓
WIP commit → reviewed_commit SHA固定 (6.1)
      ↓
Ticket ID取得 / Redmine取得（既存Skill）
      ↓
Review Artifact生成 (6.2)
      ↓
Context Collector (Sonnet) → Specification Bundle
      ↓
      ├─ Adversarial Reviewer (fresh, Sonnet)
      └─ Team Skill Runner   (fresh)
      ↓
Reviewer出力検証 (script)
      ↓
Consolidated Findings
      ↓
BLOCKER / MAJOR ?
  YES → Main Agentが修正 → test/lint → commit → 新SHAで再レビュー
        （最大cycle到達で人間へエスカレーション）
  NO
      ↓
ローカルAPI実行確認 → Request/Response Evidence
      ↓
HEAD == 最後にPASSしたSHA を確認
      ↓
既存PR本文生成処理
      ↓
完成PR本文ドラフト → STOP → Human Review
```

### 6.1 レビュー対象の固定

レビュー対象はcommit SHAで固定する。working treeのdiff fingerprintは使わない。

理由: 実装repoはMutagenでWindows側と双方向同期されているため、レビュー中にWindows側で誰かがファイルを触るとworking treeが変わる。

手順:

1.  working treeがcleanでなければWIP commitを作る（例: `wip: review target`）
2.  HEAD SHAを`reviewed_commit`としてすべてのagentへ渡す
3.  修正が入ったら再びcommitし、新しいSHAで再レビューする
4.  PR本文生成直前にHEADが最後にPASSしたSHAと一致することを確認する

WIP commitは人間がPR作成前にsquash等で整理する。Skillはpush・rebase・squashを行わない。レビュー中はWindows側で対象repoを編集しないことを人間へ提示する。

### 6.2 Review Artifact

Orchestratorのscriptが、agent起動前に以下を一時領域へ生成する。

``` text
/tmp/review/<SHA>/
  diff.patch            git diff <base>..<SHA>
  changed_files.txt
  test_result.txt       project標準のtest実行結果
  lint_result.txt
  reference/
    if/<file>/<sheet>.tsv     /reference/if 配下の全xlsxを全sheet展開
    design/...                md/txt はコピー、xlsx はtsv、pdf はテキスト抽出
    er/...
  ticket.md             Redmine取得結果
```

`/reference`は人間が手作業で厳選した小さなディレクトリなので、全展開してよい。展開後にCollectorがGrepで対象を絞る。

一時領域は実装repo内に置かない（Mutagenで同期され`git status`に現れる）。`/reference`にも書かない。Skill終了時に破棄する。

## 7. Context Collector

目的は「大量資料をReviewerへ丸投げすること」ではなく、広く検索して必要部分だけを抽出すること。

``` text
Search Scope  = broad
Context Scope = narrow
```

Specification Bundleの構造:

``` text
Ticket        ID / Subject / Description / Relevant Notes / Related tickets
API IF        Status / Source file / Sheet / Range / Endpoint / Request / Response / Validation / Status・Error
Design        Status / Source file / Section / Requirements・constraints
DB            Status / Source / Tables / Columns / Relations / Constraints / Transaction
Implementation  reviewed_commit / Changed files / Relevant diff / Relevant existing implementation / Tests / Conventions
```

各ソースに4.6のStatusを付ける。`NOT_REQUIRED`には理由を必須とする。

各要件に出典（file、sheet、range、section）を保持し、人間がBLOCKER/MAJORの根拠を辿れるようにする。

目安は10k〜20k tokens。ただしtoken削減を理由に必須仕様を欠落させない。

## 8. レビュー

### 8.1 Adversarial Reviewer

Main Agentとはfresh contextの別Subagentとして起動する。以下を渡さない: Main Agentのchain-of-thought、試行錯誤、自己正当化、以前の自己レビュー、実装時の推測過程。

Specification Bundleと実際のdiff/code/testを根拠として判定する。コードを修正しない（5.4のツール制限で強制）。

主要確認:

``` text
Redmine          ↔ Implementation
API IF           ↔ Implementation
Design           ↔ Implementation
ER/DB            ↔ DB behavior
Requirements     ↔ Tests
Repository style ↔ New code
```

#### Severity

``` text
BLOCKER  明確な仕様違反 / security issue / data loss risk / runtime error / 重大なAPI IF不一致
MAJOR    必須要件の欠落 / DB・設計不一致 / regression risk / 必須テスト不足
MINOR    不要な複雑化 / naming・style不一致 / maintainability
NIT      comment / 日本語表現 / formatting / 軽微なstyle
```

BLOCKERまたはMAJORが1件でも存在する場合はPASS禁止。MINOR/NITだけではFAILにしない。

#### Scope分類

``` text
IN_SCOPE               今回の変更diff
RELATED_EXISTING_RISK  変更に関連する既存コードのリスク
OUT_OF_SCOPE           今回と無関係な既存問題
```

`OUT_OF_SCOPE`は自動修正対象にしない。重大な既存security/data-loss問題を偶然発見した場合は人間へ別件として通知する。

#### AI生成コード特有の確認

コメント: コードを日本語へ翻訳しただけ、読めば分かる処理の説明、過剰なJavadoc/docstring、AIの説明文のようなコメント、実装変更後に古くなったコメント、不自然な日本語、プロジェクト内で使われていない造語。コメントはWHY、非自明な仕様、制約、workaround、設計理由に限定する。

過剰実装: 不要なinterface/factory/utility/generic化/fallback/config、「将来のため」の未要求拡張、scope creep。既存repositoryに同種実装があれば既存パターンへの整合を優先する。

#### Testレビュー

「実装に対してtestが通るか」だけで評価しない。Requirement → Expected behavior → Test の順で確認する。AIが実装とtestを同じ誤解に基づいて生成している可能性を前提とする。

確認例: 正常系、validation、boundary、null/empty、permission/auth、error status、DB state、transaction、regression。

#### Noise Control

MINOR/NITは`Non-blocking Observations`として分離し、重大Findingを埋没させない。同種NITはまとめ、代表例を提示し、最大5件程度を目安とする。format指摘は既存lint/formatterへ委譲する。

### 8.2 Team Backend API Skill

チーム既存の「Backend APIとして満たすべき実装・規約を確認するSkill」をレビュー工程で再利用する。

#### 実行コンテキスト

Main Agentのcontextでは起動しない。コードを書いた本人が自分の実装を規約チェックすることになるため。

fresh contextのSubagent（Team Skill Runner）を起動し、その中で呼び出す。`reviewed_commit`のSHAと`/tmp/review/<SHA>/`の場所だけを渡し、Main Agentの実装経緯を渡さない。Team Skill自体がSubagentとして実装されている場合はそのまま利用してよい。

#### 保護

禁止: Team Skillの変更、prompt変更、severity変更、自動修正機能追加、ロジックの本Skillへのコピー、本Skill専用fork。

Team Skill側の変更が必要と判断した場合は、勝手に変更せず人間へ理由を提示して停止する。

#### 実装前の解析

Skill location、Invocation method、Input、Output format、Model/subagent configuration、Checked categories、Severity definition、Whether it modifies files、Dependencies、Related scripts、Related CLAUDE.md / rules を確認する。

出力契約を判定する。

``` text
STRUCTURED   severity / location / problem を機械的に取り出せる
SEMI         一定の書式はあるが解釈が必要
FREE_TEXT    自由文
```

`SEMI`/`FREE_TEXT`の場合、本Skill側にConsolidated Findings形式へ変換するadapterを置く。adapterは出力を読むだけで、元の指摘内容・severityを改変しない。変換できない指摘は`UNPARSED`として元文を保持し人間へ提示する。`UNPARSED`をPASSの根拠にしない。adapterが成立しない場合は人間へ理由を提示して停止する。

#### Coverage Matrix

Team SkillをSingle Source of Truthとして再利用するが、そのPASSをコード全体の正しさの保証とはみなさない。解析時にCoverage Matrixを作成する。

``` text
Category                 Coverage (YES / PARTIAL / NO / UNKNOWN)
Controller structure     YES
Validation               YES
Security                 NO
DB transaction           NO
Explicit specification   NO
Test quality             PARTIAL
AI-generated hygiene     NO
```

実際の値は既存Skillの実装から判断し、推測で決めない。

-   `YES`: 同一ルールをReviewerへ重複実装しない
-   `PARTIAL`: 不足部分だけ補完
-   `NO`: 必要ならReviewerが担当
-   `UNKNOWN`: Team SkillのPASSを根拠に省略しない

Team Skill変更時に再評価できる構造にする。

#### 責務分担

``` text
Team Backend API Skill  → Project Standard / Existing Backend Convention
Specification Reviewer  → Ticket / IF / Design / ER / Tests / Diff固有の検証
```

Reviewer側で主に補完する候補: Redmine requirement consistency、Redmine Notesによる仕様変更、API IF consistency、Design consistency、ER/DB consistency、Tests proving requirements、Diff-specific regression、AI-generated code hygiene、Unrequested scope expansion。補完範囲は既存Skill解析後に確定する。

チーム固有のBackend規約はTeam SkillをSingle Source of Truthとし、本Skillへ同じ変更を二重反映しない構造を維持する。

#### 障害時

Team Skillが実行不能の場合は`TEAM_SKILL_UNAVAILABLE`としてFail Closed。チーム運用上「既存Skill障害時は人間確認で継続可能」という明示ルールがある場合のみそれに従う。代替チェックを即席で生成して成功扱いにしない。

### 8.3 出力契約と検証

ReviewerおよびTeam Skill Runner（adapter経由）は、出力の末尾に以下のJSONをフェンス付きコードブロック（言語指定`json`）で1つだけ置く。自由文はJSONの前に書いてよいが、ループ制御はJSONだけを読む。

``` json
{
  "source": "SPEC_REVIEWER | TEAM_BACKEND_SKILL",
  "review_state": "PASS | FAIL",
  "reviewed_commit": "<commit SHA>",
  "findings": [
    {
      "severity": "BLOCKER | MAJOR | MINOR | NIT",
      "original_severity": "Team Skill独自severityがあれば保持",
      "scope": "IN_SCOPE | RELATED_EXISTING_RISK | OUT_OF_SCOPE",
      "category": "...",
      "location": "path/to/file:line",
      "specification_source": "...",
      "problem": "...",
      "expected_behavior": "...",
      "evidence": "..."
    }
  ]
}
```

scriptで検証する。

``` text
[ ] json ブロックがちょうど1つある
[ ] 必須キーがすべて存在する
[ ] severity / scope が定義済みの値である
[ ] BLOCKER/MAJOR があるのに review_state が PASS になっていない
[ ] reviewed_commit が今回のSHAと一致する
```

1つでも失敗したら`REVIEWER_OUTPUT_INVALID`とし、修正を試みずfresh agentで再実行する。再実行でも不正なら人間へエスカレーションする。

### 8.4 Consolidated Findings

両方のFindingsを統合し、各Findingの`source`を保持する。Team Skillの元severityを失わず、指摘内容を書き換えて意味を変えない。

判断根拠の優先順位:

``` text
1. Redmine / API IF等の明示仕様
2. Team Backend API Skill
3. Repository existing patterns
4. 本Skillの汎用レビュー規則
```

### 8.5 SPEC_CONFLICT

明示仕様同士の矛盾（Description↔Notes、Redmine↔IF、IF↔Design、Design↔ER）、または明示仕様とTeam Skill指摘の衝突を検出した場合、Claudeが独断で片方を採用して自動修正しない。`SPEC_CONFLICT`として以下を提示し人間判断へエスカレーションする。

``` text
Explicit Specification
Conflicting Source / Team Skill Finding
Relevant Code
Conflict Description
Human Decision Required
```

Redmine Notes等に明確な変更指示があり、チーム運用で「後続Notesを正式仕様として扱う」等の明示ルールが存在する場合のみ、そのルールを適用してよい。運用ルール自体をClaudeが推測して作らない。

明示仕様が一貫しており既存コードだけが異なる場合は、`SPEC_CONFLICT`ではなくImplementation Findingとして扱う。既存コードが存在することを理由に明示仕様を無効化しない。

## 9. 修正ループ

Findingsを検出するAgentとコードを修正するAgentを分離する。修正はMain Agentだけが行う。

BLOCKER/MAJOR発生時:

1.  Main Agentが修正
2.  test / lint
3.  commit → 新SHA
4.  Review Artifact再生成
5.  fresh ReviewerとTeam Skill Runnerの両方を再実行

同一agent contextを継続使用しない。以前のPASSを新しいSHAへ流用しない。

最大3レビューサイクルを初期値とし設定可能にする。上限到達時は自動で仕様を曲げず、人間へエスカレーションする。

## 10. ローカルAPI実行確認

レビューPASS後、可能な範囲でローカルAPIを実際に実行し、Request/ResponseをPR本文の既存欄へ載せる。Swagger等での手動確認の自動化が目的。

Swagger UIの画面操作を必須方式にしない。既存環境を調査し、再現可能にrequestできる方法（curl、project test client、API client script、OpenAPIから得たendpoint情報）を選ぶ。認証や起動方法を推測しない。

本番DB・本番APIへ接続しない。接続先が不明な場合は実行せず人間へ確認する。

取得内容: method、URL/path、headers、body/query、status、response body。以下をredactする: Authorization、Cookie、access/refresh token、API key、session identifier、password/secret、project-defined secret。

実行確認不能なら`NOT VERIFIED`として人間へ返す。架空のRequest/Responseを生成しない。テンプレートの既存欄は維持し、人間が追記できる状態で止める。

## 11. PR本文生成とSTOP

既存SkillのRedmine→PR本文生成ロジックを再利用する。入力: Redmine情報、Specification Bundleの必要情報、Review PASS、実行確認Request/Response、既存PR template。

Request/Response欄があればそこへ実測値を投入する。欄がなければテンプレートを変更しない。IF URL欄は手動のまま。

PR本文ドラフト生成後、必ず停止する。人間が確認すべき未完項目を明示する。

``` text
- IF link: manual
- Request/Response: captured | NOT VERIFIED
- Adversarial Review: PASS (commit <SHA>)
- Team Backend API Skill: PASS
- PR creation: NOT EXECUTED
```

## 12. 安全境界

### 12.1 /reference

Read Only。write、edit、delete、rename、move、format conversionによる元ファイル更新、Excel workbookへの保存を行わない。この禁止規則は補助防御であり、実際の保証はPodman/OS側で行う。

### 12.2 一時成果物とsecret

一時成果物はコンテナの一時領域に置き、不要になったら破棄する。Specification Bundle、Reviewer出力、Excel変換物、Request/Response evidenceにsecretを残さない。access token、cookie、API key、password、secret、production data、private certificate/keyをGit管理対象へ追加しない。

### 12.3 Destructive Command

レビュー・PR本文生成のために以下を実行しない: database destructive migration、production deployment、force push、`git reset --hard`、`git clean -fd`、untracked user filesの削除、remote branch削除。必要性が発生した場合は人間へ返す。

## 13. Fail Closed

以下の場合、PASSやPR本文完成扱いにしない。失敗を埋めるために仕様・証跡を生成しない。

``` text
ENV_PRECHECK_FAILED
Ticket unresolved / Redmine unavailable
IF ambiguous / Required reference missing / Reference parse failed
TEAM_SKILL_UNAVAILABLE
Reviewer crashed
REVIEWER_OUTPUT_INVALID（再実行後も）
BLOCKER/MAJOR remains
Review cycle limit reached
SPEC_CONFLICT unresolved
Tests required by project failed
HEAD != last PASS commit
Runtime evidence required but unverifiable
Existing PR template cannot be identified
```

## 14. 運用上の制御

### Observability

各実行で以下を確認可能にする: Ticket ID、reviewed_commit、Reference Status、Team Skill Result、Reviewer Result、Review Cycle Count、Tests/Lint Result、Runtime Evidence Status、Final State。chain-of-thoughtは保存・出力しない。目的は失敗時に「どこで止まったか」を人間が判断できること。

### Cost Control

最大review cycle、探索範囲、subagent呼び出し回数に上限を設ける。初期運用では速度・token使用量を記録し、Production投入前評価に利用する。目安としてReviewer入力15k〜40k、初回＋再レビュー含めた総入力30k〜120k。設計上の概算であり保証値ではない。

### Version Drift

Team Skill、PR template、Claude Code、Model identifiers、Redmine conventions、IF format、Repository conventionsは将来変更され得る。これらを本Skillへハードコードせず、実行時に参照する構造を優先する。

## 15. 実装手順

### 15.1 既存実装の解析

新規にゼロから作る前に、現在のPR本文生成Skillと関連設定を特定する: Skill本体、scripts、agents、commands、CLAUDE.md、project-specific instructions、PR template、Redmine取得処理、branch名→ticket ID処理、`gh`利用箇所、Hook設定、既存のPR本文生成例、test/validation方法。

Team Backend API Skillについても8.2の解析を行う。

ユーザーへ場所を聞く前に、repository、`.claude`、Skill、commands、agents、CLAUDE.md等から自力で特定を試みる。不明点を推測して実装しない。

### 15.2 既存挙動の保護

regression禁止項目: Redmine取得、branch→Ticket ID、既存PR template、既存Markdown構造、既存IF link手動欄、既存の利用者操作。

今回の要件に不要なrefactorを行わない。既存Skillを全面的に書き直すより、責務を分離した最小差分の拡張を優先する。

### 15.3 実装計画の提示

既存実装解析後、実ファイル変更前に以下を提示する。

``` text
Existing Flow
Proposed Flow
Files to Change / Files to Add
Existing Behavior Preserved
Context Collector Design
Reviewer Design
Team Skill Integration（実行context、出力契約、Coverage Matrix）
Runtime Evidence Design
PR Body Integration
STOP Boundary
Test Plan
```

PR templateを変更する計画が含まれていたら、その計画は誤り。

### 15.4 導入順序

1.  環境構築仕様書に従いPodman環境を完成し、R/W・ROを実試験で確認
2.  既存PR本文生成Skillをバックアップ
3.  scripts（precheck、build-artifacts、validate-review）追加
4.  Context Collector追加
5.  Adversarial Reviewer追加
6.  Team Skill Runnerとadapter追加
7.  修正・再レビュー制御追加
8.  ローカルAPI実行確認追加
9.  既存PR本文生成処理へ接続
10. PR作成を行わずSTOPすることをテスト

## 16. 受入試験

本番ticketだけで検証しない。安全なfixtureまたは完了済みticket/diffを使う。

### 起動・境界

``` text
[ ] disable-model-invocation: true が設定され、/<skill名> 以外で起動しない
[ ] 起動時環境チェックが正規環境以外で停止する
[ ] Collector / Reviewer の tools が Read, Grep, Glob のみ
[ ] Team Skill Runner に Edit / Write がない
[ ] /reference を変更しない
[ ] 一時成果物が実装repo内に作られない
[ ] gh pr create / GitHub API によるPR作成が呼ばれない
[ ] Hookへ登録されていない
```

### 入力

``` text
[ ] branchからRedmine Ticketを取得できる（既存Skill経由）
[ ] Redmine Notesを含めて仕様化できる
[ ] /reference から対象IFをsheet/range付きで取得できる
[ ] design/ER を Specification Bundle へ含められる
[ ] Source Status を区別し、NOT_FOUND を問題なし扱いしない
[ ] NOT_REQUIRED に理由がある
[ ] 検索除外パターンのファイルが Bundle に含まれない
```

### レビュー

``` text
[ ] PASS case: 仕様と実装が一致 → PASS → PR本文生成まで到達 → PRは作成されない
[ ] Spec mismatch: IFと実装を意図的に不一致にしたfixtureでBLOCKER/MAJORを検出しFAIL
[ ] Missing reference: IFなしでFail Closed、別IFを勝手に採用しない
[ ] Reviewer / Team Skill Runner が fresh context で動作する
[ ] Team Skill が Main Agent ではなく subagent で実行される
[ ] Team Skill 自体を変更していない
[ ] Team Skill Finding の出所と元severityを保持する
[ ] Coverage Matrix が作成され、UNKNOWN を省略根拠にしない
[ ] 出力JSONが script で検証され、不正時に PASS 扱いにならない
[ ] MINOR/NIT だけで FAIL しない
[ ] OUT_OF_SCOPE 既存問題を修正対象にしない
[ ] SPEC_CONFLICT で停止する
[ ] TEAM_SKILL_UNAVAILABLE で無条件PASSしない
```

### ループ

``` text
[ ] BLOCKER/MAJOR 修正後に両レビューを新SHAで再実行する
[ ] PASS後にdiffを変えると以前のPASSが無効化される
[ ] cycle上限で人間へエスカレーションする
[ ] PR本文生成直前に HEAD == 最後のPASS SHA を確認する
```

### 出力

``` text
[ ] Request/Response を可能なら実測取得し、secret が redact される
[ ] 実行不能時は NOT VERIFIED で止まり、捏造しない
[ ] production へ接続しない
[ ] 改修前後のPR template構造が同一
[ ] IF link 手動欄が残る
[ ] PR本文ドラフト生成後に停止する
```

## 17. Production投入前評価

「Skillが実行できた」と「Skillが有効である」を区別する。

過去のAPI変更5〜10件以上（レビュー指摘や不具合があった変更、問題なくmergeされた変更、小規模変更、DBを含む変更、validation/error responseを含む変更を混ぜる）で評価する。機密情報・履歴の利用は会社ルールに従う。

評価観点: Known重大問題を検出できたか、誤BLOCKER/MAJOR数、平均レビュー時間、平均cycle数、人間が有用と判断したFinding、Team Skillとの重複Finding。

調整が必要な状態: 重大問題を繰り返し見逃す、誤BLOCKER/MAJORが多い、Team Skillと重複が多い、レビュー時間が開発を阻害する、MINOR/NITが多く重要Findingが読み飛ばされる。モデル変更の前にcontext不足、仕様抽出、責務分担、prompt、Coverage Matrixを確認する。

Rollout:

``` text
Phase 1: Shadow   人間の通常レビューと並行。Findingsと所要時間を記録
Phase 2: Recommended   実用性が確認できた範囲で通常利用
Phase 3: Gate     BLOCKER/MAJOR判定を正式Gateにするかチームで判断
```

正式Gate化は個人判断で行わない。

## 18. 最終報告形式

実装完了時に以下を報告する。「コードを書いた」だけでは完了ではなく、16章の受入試験を実行し、既存機能のregressionがなく、PRが作成されないことを確認して完了とする。

``` text
Changed Files / Added Files
Existing Skill Behavior Preserved
Context Collector / Specification Bundle
Reviewer / Team Skill Integration / Review Loop
Runtime Evidence
PR Template Compatibility
Reference Access
Tests Executed / PASS/FAIL Results
Known Limitations
Security Exceptions（なければ None）
PR Creation Performed: NO
```

## 19. Claudeへの開始指示

この仕様書を受け取ったClaude Codeは、最初に15.1の既存実装解析を行うこと。ユーザーへ既存ファイルの場所を聞く前に自力で特定を試み、不明点を推測して実装しない。既存実装を理解した後、15.3の形式で最小差分の実装計画を提示し、環境の承認フローに従って実装すること。
