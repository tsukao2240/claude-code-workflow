# Claude Code API検証・敵対的レビュー・PR本文生成Skill 実装仕様書 v3

## 1. 目的

既存の「Redmine情報からプロジェクト既定テンプレートに沿ったPR本文を生成するSkill」を拡張し、API実装後の仕様照合、敵対的レビュー、実行確認、PR本文ドラフト生成までを一つの明示的ワークフローとして実行する。

PRそのものは作成しない。

最終的なPR作成は必ず人間が行う。

## 2. 絶対条件

### 2.1 既存PRテンプレートを変更しない

既存Skillが利用しているプロジェクト既定PRテンプレートについて、以下を変更しない。

-   セクション
-   見出し
-   順序
-   表現
-   Markdown構造
-   既存の入力欄

テンプレート改善を理由とした独自変更も禁止。

### 2.2 PR作成は禁止

Skillの終了点は「完成したPR本文ドラフト」。

以下を実行しない。

``` text
gh pr create
GitHub APIによるPR作成
PRの自動更新
merge
auto merge
```

最終フロー:

``` text
Skill
 ↓
PR本文ドラフト
 ↓
STOP
 ↓
人間が確認
 ↓
人間がPR作成
```

### 2.3 SharePointへアクセスしない

ClaudeからSharePointへ直接アクセスしない。

API IFは人間が事前にダウンロードし、Read
Onlyの`/reference/if`へ配置する。

SharePoint
IFリンクは現行テンプレートの既存欄を維持し、人間が手動で貼る。

ClaudeはURLを推測・生成しない。

## 3. 実行環境

本Skillの正規実行環境は、別紙「Claude Code
安全実行環境構築仕様書」で定義したPodman環境。

``` text
/workspace/repo-backend   R/W
/reference                RO
```

`/reference`のRead OnlyはPodman/OS側で強制されることを前提とする。

Skill側にも変更禁止を記載するが、Skillの指示をセキュリティ境界とはみなさない。

WindowsネイティブClaudeから本Skillを実行することを正規経路としない。

### 3.1 起動時の環境チェック

Skillは処理を始める前に、正規環境で動いているかを実際に確認する。指示文だけで「コンテナで実行すること」と書いても、Windowsネイティブから起動された場合を検出できないため。

確認内容:

``` text
[ ] /workspace/repo-backend が存在し、git repositoryである
[ ] /reference が存在する
[ ] /reference 直下へのファイル作成が失敗する（touch等で実際に試す）
[ ] Excel解析ツールが呼び出せる
```

1つでも満たさない場合は`ENV_PRECHECK_FAILED`として即時停止し、Redmine取得やレビューへ進まない。

このチェックはscriptとして実装し、Skillの最初のステップで必ず呼ぶ。

## 4. 対象情報

### Redmine

-   Ticket ID
-   Subject
-   Description
-   Notes / Comments
-   関連チケット
-   その他既存Skillが現在参照している情報

特にNotes/Commentsを軽視しない。Descriptionより後に仕様変更が記載されている可能性を考慮する。

Ticket IDは現在のbranch名から既存Skillと同じ方法で取得する。

Redmineへのアクセス方法・認証も既存Skillのものをそのまま使う。本Skillで新しいRedmineクライアントや認証情報の読み込み方法を実装しない。コンテナ内から同じ方法で到達できることは環境構築仕様書5.4で担保する。

### API IF

``` text
/reference/if/
```

から対象Excelを検索する。

xlsxの読み取りには環境構築仕様書6.1で派生イメージに含めたExcel解析ツールを使う。Skill内で独自のxlsx解析処理を実装しない。

必要なsheet/rangeだけを抽出し、ReviewerへExcel全体を無条件に投入しない。

確認対象例:

-   endpoint
-   HTTP method
-   path/query/header
-   request
-   response
-   required/optional
-   validation
-   status code
-   error response
-   field type
-   nullability
-   remarks

対象IFを一意に特定できない場合は推測してレビューを継続しない。

### 設計・ER

初期運用では人間が必要資料を以下へ配置する。

``` text
/reference/design/
/reference/er/
```

受け入れる形式:

``` text
.md / .txt   そのまま読む
.xlsx        環境構築仕様書6.1のExcel解析ツールで読む
.pdf         テキスト抽出できる場合のみ
```

`.docx`、`.pptx`、`.drawio`、画像等はSkillから読めない。これらの資料が必要な場合は、人間が上記形式へ変換して配置する。変換されていない資料はContext Collectorが`PARSE_FAILED`として扱い、読めたふりをしない。

確認対象:

-   対象機能の設計
-   DB table
-   column
-   relation
-   constraint
-   transaction
-   domain rule

将来doc repositoryを直接Read Only mountする場合も、Context
Collectorの検索対象を追加するだけでReviewer構造は変更しない。

### application repository

``` text
/workspace/repo-backend
```

から以下を取得する。

-   git diff
-   changed files
-   tests
-   CLAUDE.md
-   coding conventions
-   関連する既存実装
-   同種API
-   naming/style patterns

## 5. ワークフロー

``` text
Main Agent (Opus)
  API実装
  test / lint
      ↓
ユーザーがPR本文生成Skillを明示的に起動
      ↓
起動時環境チェック (3.1)
      ↓
WIP commit → reviewed_commit SHA固定 (20.8)
      ↓
Ticket ID取得
      ↓
Redmine取得
Reference検索
git diff <base>..<SHA> 取得
      ↓
Context Collector (Sonnet)
      ↓
Specification Bundle
      ↓
Fresh Adversarial Reviewer (Sonnet)
      ↓
Findings
      ↓
BLOCKER / MAJOR ?
  YES → Main Agentへ修正要求
         ↓
       test / lint
         ↓
       Fresh Review
  NO
      ↓
PASS
      ↓
ローカルAPI実行確認
      ↓
Request / Response Evidence
      ↓
既存PR本文生成処理
      ↓
完成PR本文ドラフト
      ↓
STOP
      ↓
Human Review
```

グローバルHookやSession終了Hookとして実装しない。

本Skillが明示的に起動された場合だけレビューを実行する。

「明示的に起動された場合だけ」は指示文ではなく設定で担保する。Claude CodeのSkillはdescriptionが会話内容に一致するとClaudeが自動起動できるため、SKILL.mdのfrontmatterに以下を設定し、ユーザーの`/<skill名>`以外からは起動できないようにする。

``` yaml
disable-model-invocation: true
```

## 6. Context Collector

モデル: Sonnet。

`.claude/agents/`配下のSubagentとして定義し、`tools:`にRead、Grep、Glob、Bash（Excel解析ツール呼び出し用）だけを許可する。EditとWriteは与えない。

目的は「大量資料をReviewerへ丸投げすること」ではなく、広く検索して必要部分だけを抽出すること。

### 検索除外

Collectorは以下をSpecification Bundleへ含めない。Grep/Read対象からも除外する。

``` text
.env, .env.*
*.pem, *.key, *.p12
credentials*, secrets*
application-local.*, application-secret.* 等のprofile固有設定
~/.claude/ 配下
.git/ 配下
```

除外パターンはproject側の規約に合わせて追加できるようにし、Skill本体へハードコードしない。

原則:

``` text
Search Scope  = broad
Context Scope = narrow
```

### Specification Bundle

最低限以下を構造化する。

``` text
Ticket
- ID
- Subject
- Description
- Relevant Notes
- Related ticket information

API IF
- Source file
- Sheet
- Cell/range
- Endpoint
- Request
- Response
- Validation
- Status/Error

Design
- Source file
- Relevant section
- Requirements/constraints

DB
- Source
- Tables
- Columns
- Relations
- Constraints
- Transaction requirements

Implementation
- Commit/diff identifier
- Changed files
- Relevant diff
- Relevant existing implementation
- Tests
- Coding conventions
```

可能ならSpecification Bundleを10k〜20k tokens程度に抑える。

ただしtoken削減を理由に必須仕様を欠落させない。

## 7. Adversarial Reviewer

モデル: Sonnet。

Main Agentとはfresh contextの別Subagentとして起動する。

`.claude/agents/`配下に定義し、`tools:`にRead、Grep、Glob、Bash（git diff / git show / test実行等の読み取り用途）だけを許可する。EditとWriteは与えない。「Reviewerはコードを修正しない」は指示ではなくこのツール制限で強制する。

Reviewerへ原則渡さないもの:

-   Main Agentのchain-of-thought
-   試行錯誤
-   自己正当化
-   以前の自己レビュー
-   実装時の推測過程

Reviewerは「実装者の説明」ではなく、Specification
Bundleと実際のdiff/code/testを根拠として判定する。

Reviewerはコードを直接修正しない。

### Reviewerの主要確認

``` text
Redmine          ↔ Implementation
API IF           ↔ Implementation
Design           ↔ Implementation
ER/DB            ↔ DB behavior
Requirements     ↔ Tests
Repository style ↔ New code
```

### Severity

#### BLOCKER

-   明確な仕様違反
-   security issue
-   data loss risk
-   runtime error
-   重大なAPI IF不一致

#### MAJOR

-   必須要件の欠落
-   DB/設計不一致
-   regression risk
-   必須テスト不足

#### MINOR

-   不要な複雑化
-   naming/style不一致
-   maintainability問題

#### NIT

-   comment
-   日本語表現
-   formatting
-   軽微なstyle

BLOCKERまたはMAJORが1件でも存在する場合はPASS禁止。

## 8. AI生成コード特有のレビュー

以下を明示的に探す。

### コメント

-   コードを日本語へ翻訳しただけのコメント
-   読めば分かる処理の説明
-   過剰なJavadoc/docstring
-   AIの説明文のようなコメント
-   実装変更後に古くなったコメント
-   不自然な日本語
-   プロジェクト内で使われていない造語

コメントは原則として以下に限定する。

-   WHY
-   非自明な仕様
-   制約
-   workaround
-   設計理由

### 過剰実装

-   不要なinterface
-   不要なfactory
-   不要なutility
-   不要なgeneric化
-   不要なfallback
-   不要なconfig
-   「将来のため」の未要求拡張
-   scope creep

既存repositoryに同種実装がある場合は、独自パターンより既存パターンへの整合を優先する。

## 9. Testレビュー

「実装に対してtestが通るか」だけで評価しない。

``` text
Requirement
   ↓
Expected behavior
   ↓
Test
```

の順で確認する。

AIが実装とtestを同じ誤解に基づいて生成している可能性を前提とする。

確認例:

-   正常系
-   validation
-   boundary
-   null/empty
-   permission/auth
-   error status
-   DB state
-   transaction
-   regression

## 10. 修正ループ

BLOCKER/MAJOR発生時:

1.  ReviewerがFindingsを返す
2.  Main Agentが修正
3.  test/lint
4.  diffを更新
5.  fresh Reviewerで再レビュー

レビュー結果はcommit SHAと関連付ける（20.8参照）。working treeの状態をレビュー対象にしない。

PASS後に実装が変更された場合、以前のPASSを無効として再レビューする。

無限修正ループを避けるため、一定回数で人間へ判断を返す仕組みを持たせてもよい。

## 11. ローカルAPI実行確認

レビューPASS後、可能な範囲でローカルAPIを実際に実行する。

目的:

-   Swagger等で人間が手動実行していた確認の自動化
-   実際のRequest/ResponseをPR本文へ載せる

取得:

``` text
Request
- method
- URL/path
- headers（機密情報除外）
- body/query

Response
- status
- body
```

認証情報、token、cookie、secret等をPR本文へ出力しない。

自動実行できない場合は捏造しない。テンプレートの既存欄を維持し、人間が追記できる状態で止める。

## 12. PR本文生成

既存SkillのRedmine→PR本文生成ロジックを再利用する。

入力:

``` text
Redmine情報
Specification Bundleの必要情報
Review PASS
実行確認Request/Response
既存PR template
```

ただし、Specification
Bundleを理由にテンプレートへ新規セクションを勝手に追加しない。

### IFリンク

現行運用を維持。

既存テンプレートの「IFリンクを貼る場所」をそのまま残す。

ClaudeはSharePoint URLを検索・生成・推測しない。

### Request/Response

既存テンプレートにRequest/Response欄がある場合、その欄へ実測値を投入する。

欄がなければ勝手にテンプレートを追加変更しない。

## 13. STOP条件

PR本文ドラフト生成後、必ず停止する。

出力には人間が確認すべき未完項目があれば明示する。

例:

``` text
- IF link: manual
- Request/Response: captured
- Adversarial Review: PASS
- PR creation: NOT EXECUTED
```

Skill内からPR作成へ進まない。

## 14. 権限制御

`/reference`はRead Onlyとして扱う。

禁止:

``` text
write
edit
delete
rename
move
format conversionによる元ファイル更新
Excel workbookへの保存
```

必要な中間生成物がある場合は`/reference`ではなく、コンテナの一時ディレクトリ（`/tmp`配下等）を使用する。

実装repo内には置かない。repo内に置くとMutagenでWindows側へ同期され、`git status`にも現れるため。

重要:

> この禁止規則は補助防御であり、実際のRead Only保証はPodman/OS側で行う。

## 15. モデル構成

初期構成:

``` text
Main Agent        Opus
Context Collector Sonnet
Reviewer          Sonnet
```

Main AgentのモデルはClaude Code起動時の設定（`--model`または`settings.json`の`model`）で決まる。Skill側からMain Agentのモデルを指定する手段はない。Opusで運用したい場合は、コンテナ内Claude Codeの起動側設定で担保する。

Reviewerを最初からOpusにする必要はない。

ただし、Reviewerが見る情報はすべてContext Collectorが抽出したSpecification Bundleに限られる。Collectorが仕様（特にRedmine Notesの後発変更やIFのremarks）を落とすと、Reviewerがどれだけ優秀でも検出できない。

したがってモデル強化を検討する順序は以下とする。

``` text
1. Context Collector
2. Reviewer
```

実運用で見逃しが確認された場合、まずSpecification Bundleに必要仕様が含まれていたかを確認する。含まれていなければCollector側の問題であり、Reviewerを変更しても解決しない。

## 16. Token方針

資料全体を毎回Reviewerへ投入しない。

目安:

``` text
Specification Bundle: 10k〜20k
Reviewer input:        15k〜40k
```

初回レビュー＋再レビューを含めた総入力処理はケースにより30k〜120k程度になり得る。

これは設計上の概算であり、課金・利用上限を保証する数字ではない。

## 17. Fail Closed

以下の場合、PASSやPR本文完成扱いにしない。

-   3.1の起動時環境チェックに失敗した
-   Ticketを取得できない
-   Team Backend API Skillを実行できない（21.10参照）
-   対象IFを特定できない
-   必須設計資料が不足
-   IFの解析に失敗
-   Reviewerが正常終了しない
-   BLOCKER/MAJORが残る
-   diff stateがレビュー後に変化
-   必須テストが実行不能で、人間確認が必要

「情報がないので問題なし」と判定しない。

## 18. 完了条件

Skill実装完了条件:

``` text
[ ] 明示起動時のみ実行される（disable-model-invocation: true）
[ ] 起動時環境チェックが正規環境以外で停止する
[ ] Collector / Reviewer の tools に Edit / Write が含まれない
[ ] branchからRedmine Ticketを取得できる
[ ] Redmine Notesを含めて仕様化できる
[ ] /referenceから対象IFを取得できる
[ ] design/ERをSpecification Bundleへ含められる
[ ] Context CollectorがSonnetで動作する
[ ] Reviewerがfresh Sonnet contextで動作する
[ ] BLOCKER/MAJOR時にPASSしない
[ ] Main Agent修正後に再レビューできる
[ ] AI生成コード特有の問題をレビューする
[ ] testをRequirement基準でレビューする
[ ] Request/Responseを可能なら実測取得する
[ ] 既存PRテンプレートを一切変更しない
[ ] IFリンク欄を現行どおり手動のまま残す
[ ] PR本文ドラフト生成後に停止する
[ ] gh pr create等を実行しない
[ ] /referenceを書き換えない
[ ] Read Only保証をSkillだけに依存しない
```

## 19. 導入順序

1.  「Claude Code 安全実行環境構築仕様書」に従ってPodman環境を完成
2.  `/workspace/repo-backend` R/Wを確認
3.  `/reference` ROを実書き込み試験で確認
4.  既存PR本文生成Skillをバックアップ
5.  Context Collector追加
6.  Adversarial Reviewer追加
7.  修正・再レビュー制御追加
8.  ローカルAPI実行確認追加
9.  既存PR本文生成処理へ接続
10. PR作成を行わずSTOPすることをテスト

## 20. Claude Codeへ渡す実装指示

この仕様書を既存Skillの改修担当Claude
Codeへ渡した場合、以下の工程を守る。

## 20.1 既存実装を最初に解析する

新規Skillをゼロから作る前に、現在使用しているPR本文生成Skillと関連設定を特定する。

最低限確認する:

-   Skill本体
-   Skillから呼ばれるscripts
-   agents/subagents
-   commands
-   CLAUDE.md
-   project-specific instructions
-   PR template
-   Redmine取得処理
-   branch名からticket IDを取得する処理
-   `gh`利用箇所
-   Hook設定
-   既存のPR本文生成例があればその出力
-   test/validation方法

既存Skillの現在の正常動作を先に理解する。

## 20.2 既存挙動の保護

以下をregression禁止項目とする。

``` text
Redmine取得
branch → Ticket ID
既存PR template
既存Markdown構造
既存のIF link手動欄
既存の利用者操作
```

今回の要件に不要なrefactorを行わない。

既存Skillを全面的に書き直すより、責務を分離した最小差分の拡張を優先する。

## 20.3 変更前に実装計画を作る

既存実装解析後、以下を提示する。

``` text
Existing Flow
Proposed Flow
Files to Change
Files to Add
Existing Behavior Preserved
Context Collector Design
Reviewer Design
Runtime Evidence Design
PR Body Integration
STOP Boundary
Test Plan
```

PR templateを変更する計画が含まれていたら、その計画は誤り。

## 20.4 推奨責務分離

実際の既存構造に適合させることを優先するが、概念上は次を分離する。

``` text
Orchestrator
├─ Ticket Resolver
├─ Redmine Collector
├─ Reference Locator
├─ Context Collector
├─ Specification Bundle Builder
├─ Adversarial Reviewer
├─ Verification Loop
├─ Runtime Evidence Collector
└─ Existing PR Body Generator
```

一つの巨大promptへ全責務を押し込まない。

### Claude Code上の実体への対応

概念上の責務は、Claude Codeでは次の3種類の実体に分けて実装する。

``` text
責務                          実体
----------------------------  ------------------------------------------
Orchestrator                  SKILL.md（手順のみ。判定ロジックは持たない）
Ticket Resolver               scripts/ （branch名→Ticket ID。既存Skillの処理を呼ぶ）
Redmine Collector             既存Skillの取得処理をそのまま使用
Reference Locator             Context Collector agent 内の手順
Context Collector             .claude/agents/context-collector.md
Specification Bundle Builder  Context Collector agent の出力形式
Adversarial Reviewer          .claude/agents/spec-reviewer.md
Verification Loop             SKILL.md の手順 + scripts/（diff state記録、cycle数管理）
Runtime Evidence Collector    scripts/（request実行とredact）
Existing PR Body Generator    既存Skillの処理をそのまま使用
環境チェック                   scripts/（3.1）
Reviewer出力検証               scripts/（20.7）
```

方針:

-   判定・記録・検証のように決定的であるべき処理はscriptsに置く。Claudeの読解に依存させない
-   agentsは`model:`と`tools:`をfrontmatterで固定する
-   SKILL.mdはどのscript/agentをどの順に呼ぶかだけを書く

### 配置場所

本Skill、agents、scriptsは個人用として`~/.claude/`配下（環境構築仕様書5.3で永続化する領域）に置く。

実装repo内の`.claude/`には置かない。repo内に置くとMutagenでWindows側へ同期され、チームのリポジトリへコミットされる対象になるため。チームで共有する段階になったら、その時点でrepo内への移動をチームで判断する。

ただし既存のPR本文生成SkillやTeam Backend API Skillがrepo内`.claude/`にある場合、それらは動かさない。

## 20.5 Reference探索

`/reference`はRead Only。

対象IFや設計資料を探すとき:

1.  Ticket情報、endpoint、機能名等から候補を検索
2.  候補ファイルを絞る
3.  必要sheet/section/rangeだけ抽出
4.  出典情報をSpecification Bundleに保持
5.  複数候補で確定不能なら停止

IFファイル名だけで仕様を推測しない。

xlsxの読み取りは環境構築仕様書6.1のExcel解析ツールを`sheet`/`range`指定で呼び出して行う。`/reference`はRO mountなので元ファイルへの保存は失敗するが、Skill側でも保存を試みる処理を書かない。一時ファイルが必要ならコンテナの一時領域を使う。

## 20.6 Specification Bundleの証跡

Reviewerの各重大指摘について、可能な範囲で根拠sourceを追跡できるようにする。

例:

``` text
Source Type: API IF
File: /reference/if/xxx.xlsx
Sheet: API001
Range: B12:N35
Requirement: ...
```

設計資料についてもfile/section等を保持する。

これにより、人間がBLOCKER/MAJORの根拠を確認できるようにする。

## 20.7 Reviewer出力契約

Reviewerは構造化されたFindingsを返す。

出力の末尾に、以下のJSONをフェンス付きコードブロック（言語指定`json`）で1つだけ置く。自由文の説明はJSONの前に書いてよいが、ループ制御はJSONだけを読む。

``` json
{
  "review_state": "PASS | FAIL",
  "reviewed_commit": "<commit SHA>",
  "findings": [
    {
      "severity": "BLOCKER | MAJOR | MINOR | NIT",
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

BLOCKER/MAJORが1件でもあれば`review_state`は`FAIL`。

### 検証script

Reviewer出力はMain Agentが読解せず、scriptで検証する。

``` text
[ ] json ブロックがちょうど1つある
[ ] 必須キーがすべて存在する
[ ] severity が定義済みの値である
[ ] BLOCKER/MAJOR があるのに review_state が PASS になっていない
[ ] reviewed_commit が今回レビュー対象のSHAと一致する
```

1つでも失敗したら`REVIEWER_OUTPUT_INVALID`とし、PASS扱いにしない。修正を試みず、fresh Reviewerで再実行する。再実行でも不正なら人間へエスカレーションする。

Reviewerは修正コードを書かず、問題と期待動作を返す。

## 20.8 Diff State

レビュー対象はcommit SHAで固定する。working treeのdiff fingerprintは使わない。

理由: 実装repoはMutagenでWindows側と双方向同期されているため、レビュー中にWindows側で誰かがファイルを触るとworking treeが変わる。fingerprint方式ではPASSが理由なく無効化されるか、逆に変更後の状態を見ずにレビューを続ける。

手順:

1.  Skill起動時にworking treeが clean でなければ、レビュー用のWIP commitを作る（メッセージ例: `wip: review target`）
2.  そのHEAD SHAを`reviewed_commit`としてCollector・Reviewer・Team Skillへ渡す
3.  ReviewerとTeam Skillは`git diff <base>..<SHA>`と`git show`で対象を読む
4.  修正が入ったら再びcommitし、新しいSHAで再レビューする
5.  PR本文生成直前にHEADが最後にPASSしたSHAと一致することを確認する

WIP commitは人間がPR作成前にsquash等で整理する前提とし、Skillはpush・rebase・squashを行わない。

レビュー中はWindows側で対象repoを編集しないことを運用ルールとして人間へ提示する。

## 20.9 修正ループ制御

Main AgentがReviewer findingsを受けて修正する。

修正後:

``` text
test
lint
diff refresh
fresh reviewer
```

を実行する。

同一Reviewer contextを延々と継続使用しない。

一定回数以上FAILが続いた場合は自動で仕様を曲げず、人間へエスカレーションする。

初期値として最大3レビューサイクル程度を候補とし、既存運用に合わせて設定可能にする。

## 20.10 Runtime Evidence

Swagger UIの画面操作自体を必須方式にしない。

既存環境を調査し、同じローカルAPIへ再現可能にrequestできる方法を選ぶ。

例:

-   curl
-   project test client
-   API client script
-   Swagger/OpenAPIから得たendpoint情報

ただし認証や起動方法を推測しない。

取得したRequest/Responseから以下をredactする。

-   Authorization
-   Cookie
-   access token
-   refresh token
-   API key
-   session identifier
-   password/secret
-   その他project-defined secret

実行確認不能なら`NOT VERIFIED`として人間へ返し、架空のRequest/Responseを生成しない。

## 20.11 PR本文生成への統合

新しいレビュー結果やSpecification Bundleは、既存PR
templateを書き換える理由にしない。

PR本文に入れる情報は、既存templateの既存欄へだけ投入する。

既存欄へ対応しない内部情報はSkillの実行結果として別途表示してよいが、PR
templateへ新規セクションを追加しない。

IF URL欄は手動のまま。

## 20.12 PR作成境界

コード上でもPR本文生成とPR作成を分離する。

本Skillから以下を呼び出さない。

``` text
gh pr create
gh pr merge
GitHub REST/GraphQL PR create mutation
既存のPR作成script
```

既存SkillにPR作成処理が存在する場合は、今回のSkill実行経路からそこへ到達しないことを確認する。

`gh`は必要に応じてread操作や既存運用上のpushに使用してもよいが、PR作成は禁止。

## 20.13 Hook禁止

今回のReview workflowを以下へ登録しない。

-   global Hook
-   SessionEnd Hook
-   Stop Hook
-   unrelated command Hook

ユーザーがPR本文生成Skillを明示的に起動したときだけ実行する。SKILL.mdに`disable-model-invocation: true`を設定する（5章参照）。

既存Hookが存在しても、今回の目的に不要なら勝手に変更しない。

## 20.14 Model指定

初期設計:

``` text
Main Agent        Opus
Context Collector Sonnet
Reviewer          Sonnet
```

Main Agentは起動側設定に依存し、Skillからは指定できない（15章参照）。

CollectorとReviewerは`.claude/agents/*.md`のfrontmatterで`model:`と`tools:`を固定する。

``` yaml
model: sonnet
tools: Read, Grep, Glob, Bash
```

実際のClaude Code
Skill/Subagent設定でモデル指定可能な箇所を確認して実装する。

モデル名・設定キーを推測しない。

現行Claude Codeで指定方法が変わっている場合は公式仕様に従う。

## 20.15 実装時のテストfixture

本番ticketだけで検証しない。

可能なら安全なfixtureまたは既存の完了済みticket/diffを使い、以下を確認する。

### PASS case

``` text
[ ] 仕様と実装が一致
[ ] Reviewer PASS
[ ] PR本文生成まで到達
[ ] PRは作成されない
```

### Spec mismatch

``` text
[ ] IFと実装を意図的に不一致にしたfixtureでFAIL
[ ] BLOCKER/MAJORを検出
[ ] PR本文完成扱いにしない
```

### Missing reference

``` text
[ ] IFなしでFail Closed
[ ] 別IFを勝手に採用しない
```

### Reference protection

``` text
[ ] /referenceを変更しない
```

### Diff invalidation

``` text
[ ] PASS後にdiffを変える
[ ] 以前のPASSを無効化
[ ] 再レビュー要求
```

### Template regression

``` text
[ ] 改修前後のPR template構造が同一
[ ] IF link手動欄が残る
```

### PR safety

``` text
[ ] gh pr createが呼ばれない
[ ] GitHub APIでPRが作成されない
```

## 20.16 Fail Closed

以下の場合は処理を成功扱いにしない。

``` text
ENV_PRECHECK_FAILED
Ticket unresolved
Redmine unavailable
TEAM_SKILL_UNAVAILABLE
REVIEWER_OUTPUT_INVALID (再実行後も)
IF ambiguous
Required reference missing
Reference parse failed
Reviewer crashed
Reviewer output invalid
BLOCKER/MAJOR remains
Tests required by project failed
Diff changed after PASS
Runtime evidence required but unverifiable
Existing PR template cannot be identified
```

失敗を埋めるために仕様・証跡を生成しない。

## 20.17 実装後の最終報告

以下を報告する。

``` text
Changed Files
Added Files
Existing Skill Behavior Preserved
Context Collector
Specification Bundle
Reviewer
Review Loop
Runtime Evidence
PR Template Compatibility
Reference Access
Tests Executed
PASS/FAIL Test Results
Known Limitations
PR Creation Performed: NO
```

## 20.18 完了判定

「コードを書いた」だけでは完了ではない。

20.15の主要受入試験を実行し、既存PR本文生成機能のregressionがなく、PRが作成されないことを確認して初めて完了とする。

## 21. チーム既存Backend APIチェックSkillとの統合

## 21.1 基本方針

チームで既に用意されている「既存Backend
APIとして満たすべき実装・規約を確認するSkill」を、本Skillのレビュー工程で再利用する。

既存Backend APIチェックSkillは変更しない。

本Skill内へ既存Skillのチェックロジックをコピーして二重管理しない。

本Skillは以下を担当する。

1.  既存Backend APIチェックSkillを呼び出す
2.  そのFindingsを取得する
3.  既存Skillが確認している観点を把握する
4.  既存Skillで不足している観点だけを本Skill側で補完する
5.  仕様レビュー結果と既存Skillの結果を統合する
6.  修正が必要な場合はMain AgentへFindingsを返す

既存Backend APIチェックSkill自身にはコードを修正させない。

### 実行コンテキスト

Team Backend API SkillはMain Agentのcontextでは起動しない。

Claude CodeのSkillは起動したAgentのcontextへ読み込まれる。Main Agentがそのまま起動すると、コードを書いた本人が自分の実装を規約チェックすることになり、7章のfresh context原則と矛盾する。

したがって、Team Backend API Skillはfresh contextのSubagent（`.claude/agents/`配下、`tools:`にEdit/Writeを含めない）を起動し、そのSubagentの中で呼び出す。Subagentには`reviewed_commit`のSHAとdiff取得方法だけを渡し、Main Agentの実装経緯を渡さない。

Team Backend API Skill自体がすでにSubagentとして実装されている場合は、そのまま利用してよい。

## 21.2 実装前の既存Skill解析

本Skillを実装するClaude Codeは、最初にチーム既存Backend
APIチェックSkillを探索・解析する。

最低限以下を確認する。

``` text
Skill location
Invocation method
Input
Output format
Model/subagent configuration
Checked categories
Severity definition
Whether it modifies files
Dependencies
Related scripts
Related CLAUDE.md / rules
```

解析結果として、既存Skillが確認済みの観点を一覧化する。

例:

``` text
Controller structure
Service structure
Repository pattern
Exception handling
Response format
Validation
Naming
Project conventions
```

実際の項目は既存Skillの内容から取得し、推測で決めない。

### 出力契約の確認

既存Skillの出力が構造化されているとは限らない。解析時に以下を判定する。

``` text
STRUCTURED   : severity / location / problem を機械的に取り出せる
SEMI         : 一定の書式はあるが解釈が必要
FREE_TEXT    : 自由文
```

`SEMI`または`FREE_TEXT`の場合、本Skill側に既存Skill出力をConsolidated Findings形式へ変換するadapterを置く。adapterは既存Skillを変更せず、出力を読むだけとする。

変換時に元の指摘内容・severityを改変しない。変換できない指摘は`UNPARSED`として元文をそのまま保持し、人間へ提示する。`UNPARSED`をPASSの根拠にしない。

出力が不安定でadapterが成立しない場合は、21.3に従い人間へ理由を提示して停止する。

## 21.3 既存Skillの保護

禁止:

``` text
既存Backend APIチェックSkillの変更
既存Skillのprompt変更
既存Skillのseverity変更
既存Skillの自動修正機能追加
既存Skillのロジックを本Skillへコピー
既存Skillを本Skill専用にfork
```

どうしても既存Skill側の変更が必要と判断した場合、本Skillの実装範囲として勝手に変更せず、人間へ理由を提示して停止する。

## 21.4 チェック責務の分離

原則:

``` text
Team Backend API Skill
    ↓
Project Standard / Existing Backend Convention

Specification Reviewer
    ↓
Ticket / IF / Design / ER / Tests / Diff固有の検証
```

既存Skillが既に十分確認している項目は、本Skill側で同一ルールを再実装しない。

本Skill側で主に補完する候補:

``` text
Redmine requirement consistency
Redmine Notesによる仕様変更
API IF consistency
Design consistency
ER/DB consistency
Tests proving requirements
Diff-specific regression
AI-generated code hygiene
Unrequested scope expansion
```

ただし、この補完範囲も既存Skill解析後に確定する。

## 21.5 優先順位

判断根拠の基本優先順位:

``` text
1. Redmine / API IF等の明示仕様
2. Team Backend API Skill
3. Repository existing patterns
4. 本Skillの汎用レビュー規則
```

明示仕様とTeam Backend API
Skillの指摘が衝突した場合、Claudeが独断でどちらかを無効化しない。

以下として人間へエスカレーションする。

``` text
SPEC_CONFLICT
```

最低限以下を提示する。

``` text
Explicit Specification
Team Skill Finding
Relevant Code
Conflict Description
Human Decision Required
```

## 21.6 Review Pipeline

レビュー工程を以下に変更する。

``` text
Implementation (commit SHA)
      │
      ├──────────────────────┐
      │                      │
      ▼                      ▼
Team Backend API Skill   Specification Bundle
 (fresh subagent)            │
      │                      ▼
      │              Adversarial Reviewer
      │               (fresh subagent)
      └──────────┬───────────┘
                 ▼
        Consolidated Findings
                 │
          BLOCKER / MAJOR?
             │         │
            YES        NO
             │         │
             ▼         ▼
        Main Agent    PASS
          fixes
             │
        test / lint
             │
       diff refresh
             │
       両レビュー再実行
```

## 21.7 Consolidated Findings

本SkillはTeam Backend API SkillとAdversarial Reviewerの結果を統合する。

各Findingについて出所を保持する。

例:

``` text
Source: TEAM_BACKEND_SKILL | SPEC_REVIEWER
Severity: BLOCKER | MAJOR | MINOR | NIT
Category: ...
File/Location: ...
Specification Source: ...
Problem: ...
Expected Behavior: ...
Evidence: ...
```

Team Backend API Skillが独自severityを持つ場合、元severityを失わない。

必要なら統合表示用severityを別フィールドとして保持する。

既存SkillのFinding内容を書き換えて意味を変えない。

## 21.8 修正責務

Findingsを検出するAgentとコードを修正するAgentを分離する。

``` text
Team Backend API Skill
        ↓
      Findings
        │
Specification Reviewer
        ↓
      Findings
        │
        ▼
Consolidated Findings
        ↓
Main Agent (Opus)
        ↓
      修正
```

Team Backend API SkillおよびReviewerは原則コードを変更しない。

修正はMain Agentが行う。

## 21.9 再レビュー

Main Agentによる修正後は以下を再実行する。

``` text
test
lint
Team Backend API Skill
Specification Reviewer
Consolidation
```

以前のTeam Backend API Skill結果やReviewer
PASSを、そのまま新しいdiffへ流用しない。

diff stateが変化した場合は新しいレビューサイクルとして扱う。

## 21.10 既存Skill障害時

Team Backend API Skillが実行不能の場合、原則Fail Closed。

``` text
TEAM_SKILL_UNAVAILABLE
```

として本SkillをPASS扱いにしない。

ただし、チーム運用上「既存Skill障害時は人間確認で継続可能」という明示ルールが存在する場合のみ、そのルールに従う。

本Skillが既存Skillの代替チェックを即席で生成して成功扱いにしない。

## 21.11 受入試験追加

以下を本Skillの受入試験へ追加する。

``` text
[ ] Team Backend API Skillを特定できる
[ ] Team Backend API Skill自体を変更していない
[ ] Team Backend API SkillをMain Agentではなくfresh subagentで実行している
[ ] Team Backend API Skillをレビュー工程から実行できる
[ ] Team Skill Findingsを取得できる
[ ] Team Skillのチェック範囲を把握できる
[ ] 重複ルールを本Skillへコピーしていない
[ ] 不足観点をSpecification Reviewerが補完する
[ ] Team Skill Findingの出所を保持する
[ ] Specification Reviewer Findingの出所を保持する
[ ] BLOCKER/MAJORを統合判定できる
[ ] 明示仕様との衝突をSPEC_CONFLICTとして扱う
[ ] Reviewer/Team Skillがコードを直接修正しない
[ ] 修正後にTeam SkillとReviewerの両方を再実行する
[ ] Team Skill実行不能時に無条件PASSしない
```

## 21.12 実装時の重要原則

本Skillを「すべてのBackendルールを内包する巨大Skill」にしない。

チーム固有のBackend規約は既存Backend APIチェックSkillをSingle Source of
Truthとして扱う。

本Skillはオーケストレーション、明示仕様との照合、既存Skillで不足するレビュー観点、結果統合、PR本文生成までの制御を担当する。

これにより、チーム側のBackend規約変更時に本Skillへ同じ変更を二重反映する必要がない構造を維持する。

## 22. 仕様ソース状態・競合管理

## 22.1 Source Status

各仕様ソースについて「記載がない」と「取得・解析できなかった」を区別する。

最低限以下の状態を持つ。

``` text
FOUND
NOT_REQUIRED
NOT_FOUND
AMBIGUOUS
PARSE_FAILED
UNAVAILABLE
```

意味:

-   `FOUND`: 対象仕様を特定・解析できた
-   `NOT_REQUIRED`: 今回の変更では当該資料が不要と合理的に判断できる
-   `NOT_FOUND`: 必要と判断したが対象資料を発見できない
-   `AMBIGUOUS`: 候補が複数あり一意に確定できない
-   `PARSE_FAILED`: ファイルはあるが必要内容を解析できない
-   `UNAVAILABLE`: 外部サービス障害・権限等で取得不能

`NOT_FOUND`、`AMBIGUOUS`、`PARSE_FAILED`、必須ソースの`UNAVAILABLE`を「問題なし」と解釈しない。

## 22.2 Specification Bundleへの状態記録

例:

``` text
API IF
Status: FOUND
File: /reference/if/xxx.xlsx
Sheet: API001
Range: B12:N35

ER
Status: NOT_REQUIRED
Reason: Controller validation only; no persistence/DB change
```

`NOT_REQUIRED`には理由を必須とする。

## 22.3 仕様競合

以下のような明示仕様同士の矛盾を検出した場合:

``` text
Redmine Description ↔ Redmine Notes
Redmine ↔ API IF
API IF ↔ Design
Design ↔ ER
```

Claudeが独断で片方を採用して自動修正しない。

``` text
SPEC_CONFLICT
```

として人間判断へエスカレーションする。

ただし、Redmine
Notes等に明確な変更指示・日時・文脈があり、既存チーム運用で「後続Notesを正式仕様として扱う」等の明示ルールが存在する場合は、そのルールを適用してよい。

その運用ルール自体をClaudeが推測して作らない。

## 22.4 既存コードとの不一致

明示仕様が一貫しており、既存コードだけが異なる場合は原則として`SPEC_CONFLICT`ではなくImplementation
Findingとして扱う。

既存コードが存在することだけを理由に明示仕様を無効化しない。

## 23. Team Backend API Skill Coverage Matrix

## 23.1 目的

Team Backend API SkillをSingle Source of
Truthとして再利用するが、そのPASSをコード全体の正しさの保証とはみなさない。

既存Skill解析時にCoverage Matrixを作成する。

例:

``` text
Category                 Coverage
Controller structure     YES
Service structure        YES
Repository pattern       YES
Validation               YES
Exception handling       YES
Response format          YES
Naming                   YES
Security                 NO
DB transaction           NO
Explicit specification   NO
Test quality             PARTIAL
AI-generated hygiene     NO
```

値:

``` text
YES
PARTIAL
NO
UNKNOWN
```

実際のCoverageは既存Skillの実装から判断する。

## 23.2 Reviewerとの責務分担

-   `YES`: 原則として同一ルールをSpecification Reviewerへ重複実装しない
-   `PARTIAL`: 不足部分だけ補完
-   `NO`: 必要性がある場合はSpecification Reviewerが担当
-   `UNKNOWN`: 既存SkillのPASSを根拠に省略しない

Coverage MatrixはチームSkillの変更時に再評価できる構造にする。

## 24. Review Noise Control

## 24.1 Gate対象

自動修正・再レビューのGate対象は原則:

``` text
BLOCKER
MAJOR
```

のみ。

`MINOR`と`NIT`だけではReview StateをFAILにしない。

## 24.2 Non-blocking Findings

MINOR/NITは以下のように分離して提示する。

``` text
Blocking Findings
Non-blocking Observations
```

重大Findingを大量のstyle指摘で埋没させない。

## 24.3 NIT制御

NITを無制限に列挙しない。

初期方針:

-   同種NITはまとめる
-   代表例を提示する
-   最大5件程度を目安とする
-   重要度の低いformat指摘は既存lint/formatterへ委譲する

目的は粗探しではなく、重大な仕様違反・不具合の検出。

## 25. Production投入前評価

## 25.1 目的

「Skillが実行できた」ことと「Skillが有効である」ことを区別する。

Production運用へ定着させる前に、可能なら過去の実装/PRを用いて評価する。

## 25.2 評価対象

目安として5〜10件以上の過去API変更を候補とする。

可能なら以下を混ぜる。

``` text
実際にレビュー指摘・不具合があった変更
問題なくmergeされた変更
小規模変更
DBを含む変更
validation/error responseを含む変更
```

機密情報・履歴の利用は会社ルールに従う。

## 25.3 評価観点

最低限:

``` text
Known重大問題を検出できたか
誤BLOCKER/MAJORをどれだけ出したか
平均レビュー時間
平均レビューcycle数
人間が有用と判断したFinding
Team Skillとの重複Finding
```

## 25.4 Production投入判断

次のような状態では調整を行う。

-   重大問題を繰り返し見逃す
-   誤BLOCKER/MAJORが多すぎる
-   Team Skillとの重複が多い
-   レビュー時間が通常開発を阻害する
-   MINOR/NITが多く人間が重要Findingを読み飛ばす

モデルをOpusへ変更する前に、context不足、仕様抽出、責務分担、prompt、Coverage
Matrixを確認する。

見逃しが確認された場合は、まずSpecification Bundleに該当仕様が含まれていたかを確認し、含まれていなければContext Collector側を先に改善する（15章参照）。

## 26. 追加のProduction運用上の防御

## 26.1 Scope / Diff Boundary

Reviewerは原則として今回の変更diffを中心に評価する。

関連既存コードに問題を発見しても、今回の変更と無関係な既存問題を大量に修正対象へ含めない。

分類:

``` text
IN_SCOPE
RELATED_EXISTING_RISK
OUT_OF_SCOPE
```

`OUT_OF_SCOPE`は今回の自動修正対象にしない。

重大な既存security/data-loss問題等を偶然発見した場合は、人間へ別件として通知する。

## 26.2 Generated Artifacts / Secrets

Specification Bundle、Reviewer出力、一時Excel変換物、Request/Response
evidence等にsecretを残さない。

以下をGit管理対象へ追加しない。

``` text
access token
cookie
API key
password
secret
production data
private certificate/key
```

一時成果物は原則コンテナの一時領域へ置き、不要になったら破棄する。

## 26.3 Production Data禁止

Runtime Evidence取得のために本番DB・本番APIへ接続しない。

既存の開発/ローカル環境を利用する。

接続先が不明な場合は実行せず、人間へ確認する。

## 26.4 Destructive Command Boundary

レビュー・PR本文生成のために以下を実行しない。

``` text
database destructive migration
production deployment
force push
git reset --hard
git clean -fd
untracked user filesの削除
remote branch削除
```

必要性が発生した場合は本Skillの自動処理外として人間へ返す。

## 26.5 Timeout / Cost Control

Context探索やレビューが無制限に継続しないよう上限を設ける。

実装時に既存Claude Code機能に合わせて以下を設定・制御する。

``` text
最大review cycle
探索範囲
subagent呼び出し回数
不要資料の読み込み
```

初期運用では速度・token使用量を記録し、Production投入前評価に利用する。

## 26.6 Observability

各実行で最低限以下を確認可能にする。

``` text
Ticket ID
Diff State
Reference Status
Team Skill Result
Reviewer Result
Review Cycle Count
Tests/Lint Result
Runtime Evidence Status
Final State
```

chain-of-thoughtは保存・出力しない。

目的は、失敗時に「どこで止まったか」を人間が判断できること。

## 26.7 Version Drift

以下は将来変更され得る。

``` text
Team Backend API Skill
PR template
Claude Code
Model identifiers
Redmine conventions
IF format
Repository conventions
```

これらを本Skillへ不必要にハードコードしない。

既存Skill/テンプレートを実行時または適切なタイミングで参照し、変更後も二重管理にならない構造を優先する。

## 26.8 Rollout

最初から全案件の必須Gateにしない。

推奨:

``` text
Phase 1: Shadow / Trial
  人間の通常レビューと並行
  Findingsと所要時間を記録

Phase 2: Recommended
  実用性が確認できた範囲で通常利用

Phase 3: Gate
  BLOCKER/MAJOR判定を正式Gateにするかチームで判断
```

チーム正式Gate化は個人判断で行わない。

## 27. 追加受入試験

``` text
[ ] Source Statusを区別できる
[ ] NOT_FOUNDを問題なし扱いしない
[ ] NOT_REQUIREDに理由がある
[ ] 明示仕様同士の矛盾をSPEC_CONFLICTとして停止できる
[ ] Team Skill Coverage Matrixを作成できる
[ ] Team Skill PASSを万能な正しさ保証として扱わない
[ ] MINOR/NITだけでFAILしない
[ ] NITが重大Findingを埋没させない
[ ] OUT_OF_SCOPE既存問題を勝手に修正しない
[ ] secretを成果物/PR本文へ漏らさない
[ ] Runtime Evidenceでproductionへ接続しない
[ ] destructive commandを実行しない
[ ] review loopが無限継続しない
[ ] 実行結果の状態を人間が追跡できる
[ ] Production投入前に過去変更で評価できる
```

## 28. Claudeへの開始指示

この仕様書を受け取ったClaude
Codeは、最初に既存Skillと関連ファイルを探索・解析すること。

ユーザーへ既存ファイルの場所を聞く前に、現在のrepository、`.claude`、Skill、commands、agents、CLAUDE.md等から自力で特定を試みる。

不明点を推測して実装しない。

既存実装を理解した後、20.3の形式で最小差分の実装計画を提示し、環境の承認フローに従って実装すること。
