# Claude Code 安全実行環境構築仕様書

## 1. 目的

API実装・敵対的レビュー・PR本文生成をClaude Codeに委譲するため、Claude
Codeから参照資料を破壊できない実行環境を構築する。

要求する境界は以下。

-   実装対象リポジトリ: Claude Codeから読み書き可能
-   IF・設計書・ER図などの参照資料: Claude Codeから読み取り専用
-   Windows上のその他のファイル: 原則Claude Codeから不可視
-   Windows側の利用者: 参照資料を通常どおり読み書き可能
-   権限制御はプロンプトではなくPodmanのマウント境界で強制する

## 2. 現行環境

-   ホストOS: Windows
-   WSL2導入済み
-   PodmanはWindows側のpodman machine（WSL2 backend）経由で使用する。WSL2ディストリビューション内にPodmanを直接導入する構成ではない
-   Windows側のソースとコンテナ側の開発領域はMutagenで同期
-   Claude CodeはWindowsネイティブでもコンテナ内でも起動可能
-   現在の開発用PodmanイメージにはClaude Codeが標準では含まれない
-   過去にコンテナへClaude
    Codeを手動インストールしたが、イメージ更新時の再設定が運用負荷となった

## 3. 採用方針

会社・プロジェクト標準の開発イメージ自体は変更しない。

標準開発イメージを継承した「Claude
Code追加用の薄い派生イメージ」を作成し、Claude
Codeを使用する作業ではその派生イメージを利用する。

概念:

``` text
標準開発イメージ
      ↓ FROM
Claude用派生イメージ
  + Claude Code
  + Excel解析ツール
      ↓
Claude作業用コンテナ
```

標準イメージ更新後は派生イメージを再buildするだけでClaude
Codeを再導入できる状態にする。手動インストールを運用手順に含めない。

派生イメージへ追加するものはClaude Code本体と、`/reference/if`のExcelを読み取り専用で解析するためのツール（6.1参照）の2点に限定する。

## 4. ディレクトリ構成

Windows側の例:

``` text
work/
├─ repo-backend/          # 実装対象。既存Mutagen同期対象
├─ ai-reference/          # Claude用参照資料
│  ├─ if/                 # SharePointから手動DLしたAPI IF Excel
│  ├─ design/             # 必要な設計資料
│  └─ er/                 # 必要なER資料
└─ claude-container/      # Claude用派生イメージ定義
```

`ai-reference`は`repo-backend`のGit管理下に置かない。

プロジェクトの`.gitignore`変更を前提としない。

## 5. コンテナから見える領域

``` text
/workspace/repo-backend   R/W
/reference                Read Only
~/.claude                 永続化
```

それ以外のWindowsファイルシステムは必要がない限りマウントしない。

### 5.1 実装リポジトリ

既存のMutagen同期を維持する。

Claude Codeはコンテナ内の実装リポジトリに対して通常の開発作業を行える。

許可対象例:

-   ソース編集
-   テスト追加・変更
-   lint/format
-   build
-   git diff/status
-   必要に応じたcommit/push

PR作成は後述のSkillでは実行しない。

### 5.2 参照資料

Windows側:

``` text
C:\...\ai-reference
```

利用者は通常どおりR/W。

コンテナ側:

``` text
/reference
```

Read Only mountとする。

Claude Codeから以下を不可能にする。

-   上書き
-   削除
-   rename
-   新規ファイル作成
-   Excel等の直接更新

Read
OnlyはSkillやCLAUDE.mdの指示だけではなく、Podmanのマウント設定で強制する。

podman machine経由のため、Windows側パスはpodman machine VM内に自動マウントされたパス（例: `/mnt/c/...`）を介してコンテナへ渡る。

``` text
Windows  C:\...\ai-reference
   ↓ podman machine が VM 内へ自動マウント
VM       /mnt/c/.../ai-reference
   ↓ podman run -v ...:/reference:ro
Container /reference   (RO)
```

`podman run -v`にはWindowsパスをそのまま指定できるが、実際にどのVM内パスへ変換されるか、`:ro`が有効かは14.7の書き込み試験で必ず確認する。VM内の自動マウントはWindowsドライブ全体を対象とするため、コンテナへマウントするのは`ai-reference`ディレクトリだけに限定する。

### 5.3 Claude設定・Skill

Claude Codeの設定、Skill等はコンテナ再作成で失われないよう永続化する。

候補:

-   named volume
-   WSL側の専用ディレクトリbind mount

`~/.claude`全体を永続化する場合は、認証情報や会社ポリシーとの整合を確認する。

本題Skill（agents、scripts含む）はこの永続領域の`~/.claude/skills`および`~/.claude/agents`に置く。実装repo内の`.claude/`には置かない。repo内に置くとMutagenでWindows側へ同期され、チームリポジトリへのコミット対象になるため。

### 5.4 Redmineアクセス

Redmineの取得方法・認証は、既存のPR本文生成Skillが現在使用しているものをそのまま使う。本仕様で新しい認証方式や認証情報の保存場所を設計しない。

環境構築側で行うのは以下だけ。

-   既存Skillが認証情報をどこから読んでいるか（環境変数、設定ファイル、`~/.claude`配下など）を調査する
-   コンテナ内の既存Skillからも同じ方法で到達できるよう、必要な環境変数の引き渡しまたは設定ファイルの永続化を行う
-   コンテナからRedmineへのネットワーク到達性を確認する

認証情報を派生イメージへ焼き込まない。Git管理へ入れない。

## 6. Claude用派生イメージ

最小構成のDockerfile/Containerfileを作る。

概念例:

``` dockerfile
FROM <project-development-image>

# Claude Code: 公式のNative Install（14.2参照）。npm版を既定にしない。
RUN curl -fsSL https://claude.ai/install.sh | bash

# Excel解析ツール（6.1参照）
RUN pip install --no-cache-dir openpyxl
```

実際の導入方法は会社環境、ベースイメージ、Claude
Code公式の現行インストール方法に合わせる。

要求:

-   ベース開発環境を壊さない
-   Claude Code本体とExcel解析ツール以外の変更を入れない
-   rebuild可能
-   手作業によるClaude再インストールを不要にする
-   バージョン固定の要否を判断できる構成にする

### 6.1 Excel解析ツール

`/reference/if`のAPI IFはExcelで配置される。Claude Codeはxlsxを直接読めないため、派生イメージに解析手段を含める。

方針:

-   ベースイメージにPythonがあれば`openpyxl`を追加する
-   Pythonがなければ`python3`と`openpyxl`を追加する。Node側のxlsxライブラリを選んでもよいが、どちらか1つに固定する
-   解析は読み取り専用で行う。xlsxを開いて保存する処理を含めない
-   sheet名・range指定で必要部分だけをテキスト（TSVまたはMarkdown表）へ出力する小さなスクリプトを`claude-container/`配下に置き、Skillから呼べるようにする

スクリプトの入出力例:

``` text
input : /reference/if/xxx.xlsx, sheet=API001, range=B12:N35
output: 標準出力へTSV
```

一時ファイルが必要な場合はコンテナ内の一時領域を使い、`/reference`配下へは書かない。`/reference`はRO mountなので、書こうとしても失敗する。

## 7. 起動方式

Claude Codeを使用するAPI実装作業ではコンテナ側Claudeを利用する。

例:

``` bash
podman exec -it <development-container> claude
```

### 既存devコンテナとの関係

Claude用に別コンテナを並走させず、既存のdevコンテナ自体を派生イメージから起動する。

理由:

-   Mutagenの同期先は特定のコンテナ（またはそのvolume）に紐づく。別コンテナを並走させると、同期先volumeの共有や2つ目のsessionが必要になり、既存Mutagen構成の変更につながる
-   派生イメージは標準イメージにClaude CodeとExcel解析ツールを足しただけの上位互換なので、devコンテナとして使っても既存作業に影響しない

したがって既存の起動定義（compose / script等）でイメージ名を派生イメージへ差し替え、`/reference`のRO mountを追加するのが最小差分となる。

イメージ名の差し替えがチームの起動定義を書き換えることになる場合は、個人用の起動定義（override file等）で差し替え、チーム共有の定義は変更しない。

WindowsネイティブClaude Codeは廃止必須とはしない。ただし、本仕様のRead
Only保証が必要なSkillを実行するときはコンテナ版を正規経路とする。

## 8. Mutagenとの関係

既存ソース同期は変更しないことを基本とする。

``` text
Windows repo-backend
       ↕
     Mutagen
       ↕
Container /workspace/repo-backend
```

`ai-reference`は原則Mutagen双方向同期に含めない。

参照資料はRead Only
mount、または同等にコンテナ側の書き込みをOS/コンテナ境界で拒否できる方式を使用する。

## 9. 参照資料投入フロー

### SharePoint

``` text
SharePoint
   ↓ 人間が必要なIFをダウンロード
Windows ai-reference/if/
   ↓ Read Only
Container /reference/if/
   ↓
Claude
```

ClaudeにはSharePointへの直接アクセス権を与えない。

PR本文のSharePoint IFリンクは現行運用どおり人間が手動で貼る。

### doc repository

初期導入では必要資料を人間が`ai-reference/design`または`ai-reference/er`へ配置する。

将来、doc repository全体をClaudeに検索させる必要が生じた場合は、別途Read
Only mountとして追加する。書き込み権限は与えない。

## 10. 多層防御

優先順位:

1.  Podman/OSによる実アクセス権
2.  Claude Code Permission/Sandbox等の利用可能な制御
3.  CLAUDE.md / Skillによる行動規約

3だけをセキュリティ境界として扱わない。

Skillには参照資料変更禁止を明記するが、それは誤操作防止の補助であり、Read
Only保証そのものではない。

## 11. 動作確認

環境構築後、最低限以下を確認する。

``` text
[ ] Claudeからrepo-backendを読める
[ ] Claudeからrepo-backendを編集できる
[ ] Claudeから/referenceを読める
[ ] /referenceへのファイル作成が失敗する
[ ] /reference既存ファイルの上書きが失敗する
[ ] /reference既存ファイルの削除が失敗する
[ ] Windows側からai-referenceを編集できる
[ ] /reference/if のxlsxをsheet/range指定でテキスト化できる
[ ] xlsx解析後も元ファイルのタイムスタンプ・内容が変わっていない
[ ] コンテナ内から既存Skillの方法でRedmineを取得できる
[ ] コンテナ再作成後もClaude Codeを再手動インストールする必要がない
[ ] 派生イメージの追加パッケージがClaude CodeとExcel解析ツールだけである
[ ] 必要なClaude設定/Skillが再利用できる
[ ] Windows上の不要なディレクトリがClaudeコンテナから見えない
```

Read
Only確認はClaudeへの質問ではなく、実際のOSコマンドで書き込み失敗を確認する。

## 12. 完了条件

以下を満たした時点で環境構築完了とする。

-   Claude用派生イメージが再現可能
-   実装repoがR/W
-   referenceがRO
-   referenceがGit管理対象外
-   Windows側利用者はreferenceをR/W可能
-   Claude設定/Skillの永続化方法が確定
-   Excel解析ツールが派生イメージに含まれ、読み取り専用で動作
-   コンテナ内から既存Skillの方法でRedmineへ到達可能
-   イメージ更新後の再build手順が確定
-   Read Only境界の実動作試験がPASS

## 13. 本題Skillとの接続

環境構築後、本題Skillには固定パスとして次を渡せる状態を目標とする。

``` text
Implementation Root: /workspace/repo-backend
Reference Root:      /reference
```

本題Skillはこの境界を前提として、API実装検証・敵対的レビュー・PR本文生成を行う。

## 14. Claude Codeへ渡す実装指示

この文書をClaude Codeへ渡した場合、以下の順序で作業すること。

## 14.1 最初に現状調査する

変更を始める前に、現在の開発環境を調査する。

最低限確認するもの:

-   podman machineの構成（machine名、rootful/rootless、WSL2 backendのディストリビューション名）
-   podman machine VM内でWindowsドライブがどのパスへ自動マウントされているか
-   既存コンテナが使用しているvolume/bind mountのWindows側パスとVM内パスの対応
-   Containerfile / Dockerfileの有無と配置
-   compose / podman-compose / shell script等の起動定義
-   使用中のベースイメージ
-   コンテナのLinux distribution
-   コンテナ実行USER / UID / GID / HOME
-   repo-backendのコンテナ内実パス
-   Windows側repoとコンテナ側repoのMutagen同期設定
-   Mutagen session名、同期方向、ignore設定
-   volume / bind mount
-   現在のClaude Codeインストール有無
-   Claude設定・Skillの保存場所
-   既存PR本文生成SkillのRedmine取得方法と認証情報の読み込み元
-   ベースイメージのPython有無（Excel解析ツール選定のため）
-   Gitの認証方法
-   `gh`の利用方法
-   ローカルAPIの起動方法

不明点を推測して構成ファイルを書き換えない。

既存環境から判断できない必須事項だけをユーザーへ確認する。

## 14.2 公式仕様を確認する

Claude
Codeのインストール方法・sandbox要件など、変更される可能性がある事項は実装時点のAnthropic公式ドキュメントを確認する。

2026-09時点ではLinux/WSL向けNative Installが推奨されている。

``` bash
curl -fsSL https://claude.ai/install.sh | bash
```

npm版をClaude Code導入の既定手段にしない。

Linux/WSL2のClaude Code
sandboxを併用する場合は、公式要件に従って`bubblewrap`と`socat`等の必要パッケージを確認する。

## 14.3 変更計画を提示する

現状調査後、実ファイルを変更する前に次を提示する。

``` text
Current Architecture
Files to Change
Files to Add
Mount Changes
Mutagen Impact
Claude Installation Method
Claude Persistence Method
Security Boundary
Verification Plan
```

既存環境を大きく作り直す案より、現在のPodman/Mutagen構成に対する最小差分を優先する。

## 14.4 実装する

ユーザーの承認が必要な運用であれば承認後に実装する。

実装原則:

-   既存開発イメージを直接破壊しない
-   可能なら薄い派生イメージまたは既存構成に適した再現可能な追加レイヤーを使う
-   Claude Codeの手動インストールを残さない
-   WindowsネイティブClaude Codeを削除・変更しない
-   既存Mutagen sessionを勝手に削除・再作成しない
-   `ai-reference`を双方向同期へ追加しない
-   `/reference`の書き込み権限を与えない
-   必要のないWindowsディレクトリをmountしない
-   root常用を前提にしない
-   `--privileged`等で問題を回避しない
-   セキュリティ制約を緩めてbuild/run成功扱いにしない

## 14.5 Claude Codeの永続化

イメージ更新・コンテナ再作成後も以下を毎回手動再設定しなくてよい構成にする。

-   Claude Code本体: image buildで再現
-   Skill / agents / commands / settings: 会社ルールに適合する永続領域
-   認証:
    現在の認証方式を調査したうえで、安全に永続化可能な範囲だけを対象にする

認証情報をGit管理へ入れない。

`~/.claude`全体のmountが適切とは限らないため、現状を調査してから永続化対象を決める。

## 14.6 Sandbox

PodmanのRO mountを唯一の必須Read Only境界とする。

コンテナ内Claude Codeのnative sandboxは初期構成では使用しない。

理由:

-   native sandboxは`bubblewrap`によるユーザー名前空間のネストを必要とし、rootlessコンテナ内では`--privileged`やseccomp/capabilityの緩和なしに動作しないことが多い
-   14.4の「セキュリティ制約を緩めてbuild/run成功扱いにしない」と衝突する
-   `sandbox.failIfUnavailable = true`をコンテナ内で設定すると、sandbox不成立によりClaude Code自体が起動しなくなる

したがってコンテナ内では`sandbox`設定を有効化せず、`failIfUnavailable`も設定しない。

Claude Code側の第2層としては、`settings.json`の`permissions.deny`で`/reference`配下へのWrite/Editを拒否する設定を派生イメージまたは永続領域に含める。これは誤操作防止の補助であり、Podman RO mountの代替ではない。

将来、コンテナ側でnative sandboxが権限緩和なしに動作すると確認できた場合のみ併用を再検討する。その場合もPodmanの`/reference:ro`をsandbox設定で書き込み可能に戻さない。

## 14.7 受入試験

実装後、説明だけで終わらず実際に試験する。

### Build / 起動

``` text
[ ] 派生イメージをclean buildできる
[ ] コンテナを起動できる
[ ] claude --version が成功する
[ ] claude doctor を実行可能な範囲で確認する
```

### repository

テスト用ファイルを使用して:

``` text
[ ] /workspace/repo-backendをreadできる
[ ] /workspace/repo-backendへcreateできる
[ ] 作成したテストファイルをdeleteできる
[ ] Windows側との既存Mutagen同期が正常
```

既存ソースを破壊する試験は行わない。

### reference

テスト用ファイルを`ai-reference`に用意して:

``` text
[ ] /referenceからreadできる
[ ] /referenceへのcreateが失敗する
[ ] /reference既存ファイルへのappend/overwriteが失敗する
[ ] /reference既存ファイルのdeleteが失敗する
[ ] /reference既存ファイルのrenameが失敗する
[ ] Windows側では同じ資料を編集可能
```

書き込み試験が1つでも成功した場合はFAIL。

書き込み試験はコンテナ内のシェルから`touch`、`echo >>`、`rm`、`mv`等を実行して行う。podman machine経由のマウントはWindows側パス→VM内パス→コンテナの2段になるため、`:ro`が最終段まで効いていることをこの試験で確認する。

### Excel解析

テスト用xlsxを`ai-reference/if`に用意して:

``` text
[ ] sheet/range指定でテキスト化できる
[ ] 解析後に元xlsxのハッシュが変わっていない
[ ] 解析スクリプトが/reference配下へ一時ファイルを作らない
```

### Redmine

``` text
[ ] コンテナ内から既存Skillと同じ方法でRedmine Ticketを取得できる
[ ] 認証情報が派生イメージ・Git管理対象に含まれていない
```

### Isolation

``` text
[ ] 不要なWindowsディレクトリがコンテナへmountされていない
[ ] Windows側Claude Code環境が変更されていない
[ ] ai-referenceがGit statusに出ない
```

## 14.8 Fail Closed

以下の場合、環境構築完了と報告しない。

-   `/reference`へ書き込める
-   Claude Codeがイメージ再buildで再現できない
-   Excel解析ツールがイメージ再buildで再現できない
-   コンテナ内からRedmineへ到達できない
-   Mutagen同期を破壊した
-   Windows側環境を壊した
-   security制約を緩めないと起動できない
-   実行ユーザーやmountの実態を確認できていない
-   必須受入試験を実行していない

回避策として権限を広げる前に停止し、原因と必要な判断をユーザーへ提示する。

## 14.9 最終報告形式

作業完了時は以下を報告する。

``` text
Changed Files
Added Files
Container/Image
Claude Version / Install Method
Excel Tool
Redmine Access
Mounts
Mutagen Status
Persistence
Acceptance Test Results
Remaining Manual Steps
Security Exceptions
```

`Security Exceptions`がない場合も`None`と明記する。

## 15. Claudeへの開始指示

この仕様書を受け取ったClaude
Codeは、いきなり実装を開始せず、まず14.1の現状調査を行うこと。

現状調査だけで解決できる事項についてユーザーへ質問しない。

調査後に最小差分の実装計画を提示し、その環境の既存ルール・承認フローに従って実装へ進むこと。
