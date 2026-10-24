---
name: ingest-file
description: ACT のプロジェクトにファイル（PDF・Office・画像・Markdown・コード等）を取り込むファイル取り込みワークフロー（C2）。共有フォルダのテーマ別の置き場所に命名規則どおり配置し（初回は置き場所の構成を作り、各階層の README を更新する）、メタ情報を登録し、内容からナレッジ・決定の案・関連リンクを抽出する。同じ内容は二重に取り込まず、書く前に確認フォームで承認を取る。資料・仕様書・成果物などのファイルを渡されてプロジェクトに入れたいとき、タスクの入力や最終成果として登録したいときに使う。依頼例:「この資料を取り込んで」「この PDF をプロジェクトに入れて」「仕様書をタスク FAQ-TSK-12 の入力として登録して」「この設計書を最終成果として置いて」「/act-core:ingest-file ~/Downloads/要望書.pdf」。
argument-hint: "<ファイルのパス…> [用途] [関連タスク] [プロジェクト]"
---

# ingest-file — ファイル取り込み（配置・メタ情報・抽出）

## 目的

1 回の実行で、渡されたファイルについて次を揃える。保存先と設定ファイルへの書き込みは、すべて確認フォームでユーザーが採用した項目に限る。

- 本体: 対象プロジェクトの共有フォルダの、内容に合うテーマ別の置き場所（`folder_root`（既定 `library`）の下の階層）に、`project.yaml` の `stores.artifact_body.layout` どおりに配置する。同じ内容（ハッシュ）は二重に置かない。
- 置き場所: 構成が未設定のプロジェクトでは、初回の構成（第 1 階層と今回要る第 2 階層、各フォルダの目的）を作って設定する。以後は既存の構成から選び、合う場所が無ければフォルダを足す。ファイルを置いた置き場所と、新しく作った置き場所の親の索引（直下の一覧と状態）を更新する。
- メタ情報: 📄 アーティファクト（名称・種類・用途・状態・版・要約・ハッシュ・所在、出典に使ったワークフロー設定の ID と版）を登録・更新する。
- 抽出: 本人の理解としてのナレッジ、資料に書かれた決定からの決定の案（`act-core:permission-gate` の `decision_propose` の判定どおり、提案か判断依頼にする）、関連する既存のタスク・決定・ナレッジへのリンク、辞書への追加候補。
- 記録: 保存先に書いた実行をプロジェクトの実行ログに 1 行残し、詳細を作業フォルダに残す。

資料に「決定した」「承認済み」と書かれていても、それだけを根拠に確定や承認済みにはしない。取り込み時の状態の既定は `下書き` で、`承認済み` は利用者が求め、かつ `act-core:permission-gate` の `artifact_approve` で許されるときだけ計画に載せる。

## 実行前提

- 起動: `/act-core:ingest-file <ファイルのパス…> [用途] [関連タスク] [プロジェクト]`。「この資料を取り込んで」「この PDF をプロジェクトに入れて」のように、ファイルを渡されてプロジェクトへの取り込みを頼まれたときは依頼文から自動でも起動する（`disable-model-invocation` は指定しない。誤って起動しても、Step 5 で中止を選べば何も書かれない）。
- 引数 `$ARGUMENTS` はすべて任意で、形から見分ける（Step 1）。チャットに添付されたファイルも入力にする。1 回で受けるファイルは 1〜10 件。
- 前提: `/act-core:setup` 済みであること（個人設定にプロジェクトがあり、`<root>/.act/project.yaml` に `stores.artifact_body` と `stores.artifact_meta` がある）。足りなければ Step 3 で `/act-core:setup` へ案内して止める。
- 保存先には本人の接続（Notion コネクタなど）で書く。認証情報はファイルに書かない。

## メインの作法

- ユーザーへの質問・確認はメインだけが行う。サブエージェントは質問せず、問うべき論点を出力に書く。
- サブエージェントには入力と出力先のパスを引数で渡す。サブエージェントの戻り値は、出力ファイルのパス 1 行（失敗時は `ERROR:` で始まる 1 行以上）とする。
- メインは、サブエージェントの出力ファイルを、下の「メインが読む見出し」で指定した見出し・キーだけ読む。本文を書いたり直したりしない。取り込むファイルの中身もメインは読まない（内容の読み取りは `act-core:ingest-file-planner` の担当）。
- 戻り値が `ERROR:` で始まったときの扱いは「失敗時の対応」に従う。
- 戻せない操作（本体の配置、置き場所の作成・構成の設定、索引の更新、メタ情報・抽出の登録、`project.yaml` の変更）の前に、内容を示してユーザーの承認を取る。本ワークフローでは Step 5 の書き込み計画フォームがその承認にあたり、Step 6 は承認された項目だけを適用する。
- 作業開始前に 1 回だけ `ToolSearch` を `query: "select:AskUserQuestion"`、`max_results: 1` で呼び、`AskUserQuestion` を使えるようにする。読み込めないときの尋ね方は `act-core:write-plan` の「部品の選び方」の段の下げ方に従う（Step 1・3 の質問も同じ）。
- 作業開始前に `Skill` ツールで `act-core:local-files` を読む。メインが読み書きする設定ファイルと作業フォルダ（Step 1 の個人設定と `01_request.yaml`、Step 3 の前回の実行の確認、Step 5 の `04_form-answers.json`・`04_approved-plan.json` など）は、これに従う。
- Step 3 の前に `Skill` ツールで `act-core:write-plan` を読み込む。
- Step 3・5 の確認フォームの組み立て（部品の選び方と、使えない環境での段の下げ方）と確信度の示し方は `act-core:write-plan` に従う。
- 時間のかかるステップ（Step 2・4・6）の前に、何をしているかを 1 行でユーザーに伝える。

### メインが読む見出し

| 用途 | 対象ファイル | 読むところ |
| --- | --- | --- |
| Step 1 共有フォルダの接続（Cowork） | `<config-path>` | `projects` の各キーの `name`・`root`・`status` |
| Step 3 前回の実行の確認 | 前回の作業フォルダの `02_context.json` / `04_approved-plan.json` / `05_apply-log.jsonl` / `06_apply-result.md` | `project.key`・`files[].hash` / 各要素の `item_id`・`decision` / 各行の `item_id`・`status` / `## 結果の要約` の先頭行（`状態: 完了` か `状態: 停止（段 <N>）`）と要点 |
| Step 3 前提の確認 | `<workdir>/02_context.md` / `<workdir>/02_context.json` | `## 判定`、`## 対象プロジェクト`、`## 警告` / `project.key`・`project.root`、`files[].hash` |
| Step 5 書き込み計画フォーム | `<workdir>/03_ingest-plan.md` / `<workdir>/03_ingest-plan.json` | `## 概要`、`## 確認項目`、`## 警告` / `items` の各要素の `item_id`・`replan_key` |
| Step 7 結果の報告 | `<workdir>/06_apply-result.md` | `## 結果の要約`、`## 配置したファイル`、`## 置き場所の変更`、`## 保留した項目`、`## 警告` |

`03_ingest-plan.md` の `## 採用済みでまとめる項目` はメインは読まない。その項目 ID は `03_ingest-plan.json` の `items` から取る（Step 5）。

## ファイルの置き場所

- 作業フォルダ `<workdir>` は `~/.act/runs/<YYYY-MM-DD_HHMMSS>_ingest-file/`（実行ごとに新しく作る。前回の続きから適用するときだけ前回のものを使う）。run_id は作業フォルダ名。作業フォルダは、実行者の手元の詳細な記録と、続きから適用するときの起点として削除しない。チームで共有する実行の記録は、Step 6 で `act-core:ingest-file-applier` が保存先の実行ログに 1 行残す（run_id で作業フォルダと対応する）。秘密情報と認証情報は書かない。
- メインとサブエージェントの間で受け渡すファイル（`01_request.yaml`、`02_context.json`、`03_ingest-plan.json`、`04_approved-plan.json`）のキーと、実行ログの `held` の形は、本スキルの定めを正とする。

| ファイル | 書く役 | 内容 |
| --- | --- | --- |
| `00_inputs/` | メイン | Cowork のチャット添付を PC 側に書き出したもの（添付が無ければ作らない） |
| `01_request.yaml` | メイン | 取り込みの依頼（キーは下の `### 01_request.yaml のキー`。Step 3・5 で `project`・`overrides` を上書き） |
| `02_context.md` / `02_context.json` | `act-core:ingest-file-inspector` | 判定・対象プロジェクト・入力ファイル・置き場所の構成・使うワークフロー設定・警告 / 機械が読む前提の値（形は下の `### 02_context.json の形`） |
| `02_workflow-config.md` | `act-core:ingest-file-inspector` | `act-core:workflow-config` の書き出しの形（重ねた C2 の設定と「使った設定」の表） |
| `03_ingest-plan.md` / `03_ingest-plan.json` | `act-core:ingest-file-planner` | 人が読む書き込み計画と確認項目 / 項目 ID ごとの適用内容（形は下の `### 03_ingest-plan.json の形`） |
| `04_form-answers.json` | メイン | 計画を作り直す前のフォームの答え（Step 5。作り直した計画のフォームの初期値に使う） |
| `04_approved-plan.json` | メイン | 項目 ID ごとの採用・保留・除外と修正値（キーは Step 5） |
| `05_apply-log.jsonl` / `06_apply-result.md` | `act-core:ingest-file-applier` | 項目ごとの適用記録 / 適用結果・配置したファイル・置き場所の変更・保留・警告 |

- 個人設定の実体パスを `<config-path>`（絶対パス）とし、Step 2 に渡す。
- Claude Code では `~/.act/`、取り込むファイル、共有フォルダ `<root>` をそのまま読み書きする。`<config-path>` は `~/.act/config.yaml` を絶対パスに展開したもの。
- Cowork では `~/.act/`、取り込むファイルを含むフォルダ、`<root>` はメンバーの PC 上にあり、接続フォルダにして扱う（Step 1・2）。メインとサブエージェントに渡すパスは PC 上の絶対パス（`/Users/...` など）とする。読み書きの手段（接続フォルダの読み替えを含む）は、メインもサブエージェントも `act-core:local-files` に従う。

### 01_request.yaml のキー

メインが書き、全サブエージェントが読む。

| キー | 値 |
| --- | --- |
| `files` | 取り込むファイルの配列。各要素は `file_id`（`f1`、`f2`… の順番）と `path`（PC 上の絶対パス）。ハッシュは書かない（正は `02_context.json` の `files[].hash`） |
| `project` | 明示されたプロジェクト名かキー（無ければ空）。Step 3・5 でプロジェクトを選び直したときは選ばれたキーで上書き |
| `session_project` | このセッションで選択中のプロジェクトのキー（分かるときだけ。無ければ空） |
| `usage` | 引数で指定された用途（`入力`／`作業途中`／`検証・記録`／`最終成果`。無ければ空） |
| `related_task` | 引数で指定された関連タスク（`<KEY>-TSK-<n>` か Notion の URL。無ければ空） |
| `status_request` | `承認済み` が求められたときだけ `承認済み`（無ければ空） |
| `overrides` | Step 3・5 の修正。初めは空。キーは下の表 |

| `overrides` のキー | 値 | 計画での扱い（`act-core:ingest-file-planner`） |
| --- | --- | --- |
| `project` | 選び直したプロジェクトのキー | このプロジェクトで計画する |
| `structure` | 初回の構成案への自由記述の修正（そのまま）か、`none`（構成を設定しない） | 自由記述は構成案に反映する。`none` なら構成の項目（`layout_set`・`folder_create`）を載せず、全ファイルを従来の固定パス（`02_context.json` の `stores.artifact_body` の `layout`）で計画する |
| `skip_folders` | 作らないことになった置き場所のパスの配列 | そのフォルダを作る項目を載せず、そこに置く予定だったファイルは既存の置き場所から選び直す（無ければ従来の固定パス） |
| `usage.<file_id>` | そのファイルの用途 | この用途で計画する |
| `kind.<file_id>` | そのファイルの種類 | この種類で計画する |
| `folder.<file_id>` | そのファイルの置き場所のパス | この置き場所に置く |

### 02_context.json の形

`act-core:ingest-file-inspector` が書き、メイン（Step 3）と `act-core:ingest-file-planner`（Step 4）・`act-core:ingest-file-applier`（Step 6）が読む。

| 最上位のキー | 値 |
| --- | --- |
| `project` | `key`・`name`・`client`・`root`・`roles`（このプロジェクトでの役割）・`rule`（決めた規則の番号 1〜5）・`confidence`・`requires_confirmation` |
| `operator` | 操作者。個人設定の `me` の `name`・`email` を写したもの（秘密情報は写さない）。planner は権限の判定とナレッジの所有者に、applier は実行ログの `run_by` に使う。保存先ごとのアカウントは `accounts` の `user` |
| `stores` | `project.yaml` の `stores`（`artifact_body` の `layout`・`folder_root` と `mapping` を含む）。Step 2 の時点の値で、Step 6 の段 1 で変わっても書き換えない |
| `folders` | `state`（`configured`／`unconfigured`／`unsupported`）と `tree`（`[{path, purpose, files, folders}]`。`configured` のときだけ） |
| `policy` | `project.yaml` の `policy` |
| `accounts` | 書き込み前の照合に使う値（アダプタごとの `workspace` と `user`） |
| `schema` | アダプタごとの記録値と目標値（`describe` の `schema_version`） |
| `files` | 各ファイルの `file_id`・`path`・`ext`・`size`・`hash`（`sha256:<hex>`）・`readable` |
| `run_id` | `<workdir>` のフォルダ名 |

### 03_ingest-plan.json の形

`act-core:ingest-file-planner` が書き、メイン（Step 5）と `act-core:ingest-file-applier`（Step 6）が読む適用用の計画。

| 最上位のキー | 値 |
| --- | --- |
| `run_id` | `<workdir>` のフォルダ名 |
| `project_key` | 対象プロジェクトのキー |
| `structure` | 初回の構成案（`folders.state` が `unconfigured` で、`overrides.structure` が `none` でないときだけ）。`act-core:artifact-placement` の構成案の形 `{folder_root, layout, folders: [{path, purpose, create}]}` |
| `files` | ファイルごとの配置予定。各要素は `file_id`・`category`（`input`／`working`／`deliverable`）・`folder`（置き場所。従来の固定パスに置くときは無し）・`fields`（`date`・`title`・`kind`・`ext`・`version`・`folder`）・予定パス・判定（`new`／`overwrite`／`version_up`／`duplicate`） |
| `items` | 項目の配列（中止の計画では空） |
| `warnings` | 計画時の警告の配列。各要素は `text`（`03_ingest-plan.md` の `## 警告` と同じ文）と `item_id`（関わる項目。無ければ `null`） |

| `items` の要素のキー | 値 |
| --- | --- |
| `item_id` | `03_ingest-plan.md` の `## 確認項目` または `## 採用済みでまとめる項目` の項目 ID と同じ。計画を作り直しても、同じ対象・同じ内容の項目には同じ ID を振る |
| `file_id` | 対象のファイル（ファイルに属さない項目は `null`） |
| `stage` | 適用の段（1〜8。下の段の表）。`kind` が `confirm` のときは `null` |
| `kind` | 項目の種類。`confirm`（確認だけで適用しない。対象プロジェクト、ファイルごとの用途・種類・置き場所）／`layout_set`／`folder_create`／`put_file`／`artifact_create`／`artifact_update`（`承認済み` への状態の変更は `payload.attributes.status` で表す）／`tag_create`／`knowledge_create`／`decision_propose`／`request_create`／`link`／`comment`（`act-core:permission-gate` の結果が `書かない（コメントのみ）` のときのコメント）／`dictionary_append`／`index_update` |
| `depends_on` | 前提になる項目の `item_id` の配列（無ければ空） |
| `requires_confirmation` | 確認が要るか（真偽） |
| `payload` | ストア操作の `args`（`confirm` は確認する値と候補）。`layout_set` は `project.yaml` の `stores.artifact_body` に書く `folder_root` と `layout`。`folder_create`・`put_file`・`index_update` には必ず `folder_root` と `layout` を入れ、値は今回の計画が前提にする構成のもの（`layout_set` があればその `payload` の値、無ければ `02_context.json` の `stores.artifact_body` の値）にする。適用して初めて決まる値（段 3 の所在とハッシュ、段 8 の行の値など）は空にし、`depends_on` に元の項目を入れる |
| `replan_key` | この項目への答えで計画を作り直すときの、`01_request.yaml` の `overrides` の書き先（Step 5 手順 5）。1 項目に 1 つだけ。`confirm` の項目は `project`／`usage:<file_id>`／`kind:<file_id>`／`folder:<file_id>`、`layout_set` は `structure`、`folder_create` は `skip_folder:<フォルダのパス>`。それ以外（`put_file` を含む）は `null` |

適用の段（`act-core:ingest-file-applier` はこの順で適用し、`05_apply-log.jsonl` の `stage` にこの番号を書く。段 0 は項目を持たない。`confirm` の項目は適用せず、`05_apply-log.jsonl` にも行を書かない）。

| 段 | 内容 |
| --- | --- |
| 0 | 接続先の照合（`act-core:store-protocol` の「書き込み前の接続先の照合」。保存先には何も書かない） |
| 1 | 置き場所の準備（初回だけ `project.yaml` に `folder_root` と `{folder}` を使う `layout` を書く `layout_set`、置き場所の作成 `folder_create`） |
| 2 | 本体の配置（`put_file`） |
| 3 | メタ情報（📄 アーティファクトの `create`／`update`。所在とハッシュは段 2 の結果を使う） |
| 4 | 新しいタグ |
| 5 | ナレッジ・決定の提案・判断依頼 |
| 6 | リンク（関連するタスク・決定・ナレッジとのリンクと、アーティファクト本文への抽出したコンテキストの追記）とコメント（`comment`） |
| 7 | 辞書への追記 |
| 8 | 索引の更新（`index_update`。行の値は段 2・3 の結果で埋める） |

### 実行ログの held の形

`act-core:ingest-file-applier` が、実行ログ（`act-core:store-protocol` の `run_log`）の記述 `held` に書き、次回以降の `act-core:ingest-file-planner` が読む。

- 書く: `04_approved-plan.json` で `held` になった項目（`kind` が `confirm`・`layout_set`・`folder_create` のものを除く）を 1 項目 1 要素で入れる。要素は `{item_id, kind, file_hash, title, payload}`。`file_hash` は対象ファイルの `02_context.json` の `files[].hash`（ファイルに属さない項目は `null`）、`title` はフォームに示した項目の見出し、`payload` は `03_ingest-plan.json` のその項目の `payload`。
- 読む: `run_log` を `query`（`workflow: C2`、対象プロジェクト）し、当たった件を `get` して `held[].file_hash` を今回の `files[].hash` と照らす。一致した要素は、その `kind` と `payload` を元に計画に再び載せる（今回の内容で作り直してよい）。

## 手順

### Step 1 準備（メイン）

1. `$ARGUMENTS` を形で見分ける。
   - `/` か `~` で始まる → 取り込むファイルのパス（複数可）
   - `<KEY>-TSK-<n>` の形か Notion の URL → 関連タスク
   - `入力`／`作業途中`／`検証・記録`／`最終成果` → 用途の指定
   - `承認済み` → 状態の希望
   - 残り → プロジェクト名かキー
2. 引数のパスとチャットの添付を合わせたファイルが 0 件なら、「取り込むファイルをチャットに添付するか、PC 上の絶対パスを 1 行に 1 件で答えてほしい（1 回に 10 件まで）」と尋ねる。11 件以上なら、何もせずに「1 回に取り込めるのは 10 件まで。分けて実行してほしい」と案内して終える。
3. Cowork なら、Step 2 で使うフォルダを接続フォルダにする。接続済みかの確認は `mcp__remote-devices__get_device_info`、フォルダの有無は `mcp__remote-devices__device_list_dir`、未接続のものの接続は `mcp__remote-devices__device_request_folder_access` で、理由を添えて求める。
   1. `~/.act/` と、各ファイルを含むフォルダ（理由: ACT の設定と実行記録を読む・取り込むファイルを読むため）。未接続のものを 1 回でまとめて求める。PC のホームに `.act` が無ければ、`/act-core:setup` を先に実行するよう 1 行で案内して終える。
   2. 共有フォルダ `<root>`（理由: プロジェクト設定を読み、共有フォルダにファイルを置くため）。`<config-path>` の `projects` から、明示されたプロジェクトが `projects` のキーか `name` と一致すればその 1 件、一致しなければ `status: archived` を除く全件の `root` を対象にし、未接続のものを 1 回でまとめて求める。
   - 接続を断られたら、何が要るかと理由を示して終える。
4. `~/.act/runs/<YYYY-MM-DD_HHMMSS>_ingest-file/` を作り、絶対パスを `<workdir>` とする。
5. Cowork でチャットに添付されたファイルがあれば、`mcp__remote-devices__device_commit_files` で `<workdir>/00_inputs/` に書き出し、その PC 上の絶対パスを取り込むファイルのパスとして使う。
6. `<workdir>/01_request.yaml` を書く（キーは `### 01_request.yaml のキー`。`overrides` は空）。前回の未完了の実行の確認は、今回のファイルのハッシュが Step 2 で分かってから Step 3 の冒頭で行う。

### Step 2 前提の点検・プロジェクト解決・設定の読み込み（`act-core:ingest-file-inspector`）

- Cowork では、起動の前に、`01_request.yaml` の `project` が指すプロジェクトの `root` が未接続なら（Step 3・5 でプロジェクトを選び直したときなど）、Step 1 手順 3 の 2 と同じ理由で接続を求める。断られたら、何が要るかと理由を示して終える。
- 起動前にユーザーへ「対象プロジェクトと取り込みの前提を点検しています（まだ何も書き込みません）」と伝える。
- 渡す引数: `request-path`: `<workdir>/01_request.yaml`、`workdir`: `<workdir>`、`config-path`: `<config-path>`
- 戻り値: `<workdir>/02_context.md` の絶対パス 1 行（同時に `<workdir>/02_context.json` と `<workdir>/02_workflow-config.md` が書かれる）
- このサブエージェントは何も書き換えない。前提が揃わないときは `## 判定` の先頭行が `判定: 中止` になる。

### Step 3 前提の確認（メイン）

1. `02_context.md` の `## 判定`・`## 対象プロジェクト`・`## 警告` と、`02_context.json` の `project.key`・`project.root`・`files[].hash` だけを読む。
2. `判定: 中止` なら、理由と次にすべきこと（多くは `/act-core:setup` の実行。アダプタを入れる、コネクタをつなぎ直す、など）と `## 警告` を示して終える。
3. `判定: 要確認`（対象プロジェクトの候補が複数で決められない）なら、`## 対象プロジェクト` の候補を選択肢にして `AskUserQuestion` で 1 問尋ねる。背景として各候補の根拠を添え、選択肢に「中止する（何もせずに終える）」を含める。選ばれたキーで `01_request.yaml` の `project` を上書きして Step 2 に戻る。中止なら終える。
4. `判定: 続行` なら、前回の未完了の実行を探す。`~/.act/runs/*_ingest-file/`（今回の `<workdir>` を除く）のうち、`02_context.json` の `project.key` が今回と同じで、`files[].hash` の集合が今回と同じで、`04_approved-plan.json` があるものから、最も新しい 1 件だけを見る（それより古い実行は見ない）。その 1 件が次の両方に当てはまるときだけ未完了とする。
   - `04_approved-plan.json` に `decision` が `adopted` の項目が 1 件以上ある（採用 0 件で終えた実行は未完了としない）。
   - `06_apply-result.md` が無いか、その `## 結果の要約` の先頭行が `状態: 完了` でない。

   未完了なら、残りの件数（前回 `adopted` にした項目のうち、前回の `05_apply-log.jsonl` に `status` が `success` か `already_applied` の行が無いもの）を数え、`AskUserQuestion` で 1 問尋ねる。背景: 前回の日時、どこで止まったか（`## 結果の要約` の要点）、残りの件数。選択肢:「続きから適用する（前回承認した計画のうち、残りの N 件を適用する。成功済みの項目は飛ばす）」「最初からやり直す（あらためて点検して計画を作り直す）」「中止する（何も変えずに終える）」。
   - 続きからなら、前回の作業フォルダを `<workdir>` として Step 6 に進む。中止なら終える。最初からやり直すなら手順 5 に進む。
   - 未完了でなければ、何も尋ねずに手順 5 に進む。
5. Step 4 に進む。プロジェクトが規則 ④⑤（内容からの推定・既定のプロジェクト）で決まったときも続行し、Step 5 の計画フォームで明示して確認する（計画の先頭の「対象プロジェクト」の項目）。

### Step 4 取り込み計画の作成（`act-core:ingest-file-planner`）

- 起動前にユーザーへ「ファイルの内容を読んで取り込み計画を作っています（まだ何も書き込みません）」と伝える。
- 渡す引数: `request-path`: `<workdir>/01_request.yaml`、`context-json-path`: `<workdir>/02_context.json`、`workflow-config-path`: `<workdir>/02_workflow-config.md`、`workdir`: `<workdir>`
- 戻り値: `<workdir>/03_ingest-plan.md` の絶対パス 1 行（同時に `<workdir>/03_ingest-plan.json` が書かれる）
- このサブエージェントは保存先を読むだけで、何も変更しない。初回（構成が未設定）は置き場所の構成案も計画に載せる。計画を作れないときは `## 概要` の先頭行が `判定: 中止` になる。

### Step 5 書き込み計画フォーム（メイン）

1. `03_ingest-plan.md` の `## 概要`、`## 確認項目`、`## 警告` と、`03_ingest-plan.json` の `items` の各要素の `item_id`・`replan_key` だけを読む。
2. `## 概要` の先頭行が `判定: 中止` なら、理由と次にすべきことと `## 警告` を示して終える。
3. `## 確認項目` が 0 件で、`items` も 0 件（すべて取り込み済みで重複など、書くものが無い）なら、フォームを出さずに `## 概要` と `## 警告` を示して終える。
4. それ以外は、次のとおりに示して確認フォームを 1 回出す。
   - 示すもの: `## 概要`（初回は置き場所の構成案の木（フォルダと目的）、ファイルごとの配置予定、更新する索引、件数、適用後に実行ログへ今回の実行を 1 行残すこと）と `## 警告`。フォームと同じメッセージで示す。
   - フォームの組み立て: `## 確認項目` の並び・各項目の確認要否・推奨の採否をそのまま使う（組み立て・中止の選択肢は `act-core:write-plan` の「確認フォームの組み立て」に従う）。`<workdir>/04_form-answers.json` があれば（計画を作り直したあと）、同じ `item_id` の項目はその答えを初期値にする。
   - 受け付ける答え: 各項目の採用・保留・除外と短い修正。初回の構成案の項目では自由記述での修正（例:「顧客要望と評価を分けて」）も受け付ける。確認だけの項目（対象プロジェクト、ファイルごとの用途・種類・置き場所）は値を選ぶだけで、採否は問わない。
5. 答えに応じて次のどれか 1 つを行う（上から順に当てはまるものを選ぶ）。
   - 中止が選ばれたら、何も書かずに終える。
   - `replan_key` を持つ項目への答えが次のどれかに当たるときは、計画を作り直す。推奨（初期値）のまま答えただけなら作り直さず、次の項目に進む。
     - `project`・`usage:<file_id>`・`kind:<file_id>`・`folder:<file_id>`: 初期値と違う値が選ばれた、または修正欄に書かれた。
     - `structure`: 自由記述で修正が書かれた、または保留・除外になった。
     - `skip_folder:<フォルダのパス>`: 保留・除外になった。

     作り直すときは次の順に行う。
     1. 「この修正で計画を作り直します（ほかの項目の答えは、作り直した計画の初期値に引き継ぎます）」と 1 行伝える。
     2. `01_request.yaml` の `overrides` に書く。`replan_key` ごとの書き先は次のとおり。
        - `project` → `overrides.project` と、最上位の `project` の両方に選ばれたキー
        - `structure` → `overrides.structure` に自由記述をそのまま。保留・除外なら `none`
        - `skip_folder:<フォルダのパス>` → `overrides.skip_folders` の配列にそのパスを足す
        - `usage:<file_id>`／`kind:<file_id>`／`folder:<file_id>` → `overrides.usage.<file_id>`／`overrides.kind.<file_id>`／`overrides.folder.<file_id>` に選ばれた値
     3. `replan_key` が `null` の項目への答えを、`04_approved-plan.json` と同じ形（手順 7 の表）で `<workdir>/04_form-answers.json` に書く（前回の内容に同じ `item_id` があれば今回の答えで置き換える）。
     4. `project` の書き先があれば Step 2 に、それ以外なら Step 4 に戻る。この往復は作り直す答えが無くなるまで続ける（上限は設けない。各回のフォームで中止を選べる）。
   - それ以外は、`act-core:write-plan` の「答えの反映」に従って採否と修正値を確定する。確定の確認で中止が選ばれたら、何も書かずに終える。
6. 確定した計画で分岐する。
   - 採用が 0 件なら、全項目を `held` か `excluded` として `<workdir>/04_approved-plan.json` に書き、「何も書いていない。保留した項目は同じファイルで `/act-core:ingest-file` を再実行すると計画に再び載る」と伝えて終える（この実行は次回の Step 3 手順 4 で未完了とは見なされない）。
   - それ以外は `<workdir>/04_approved-plan.json` を書いて Step 6 に進む。
7. `04_approved-plan.json` は `{"items": [...]}` とし、`03_ingest-plan.json` の `items` の全項目（`## 確認項目` の項目と、採用済みでまとめた項目）について次のキーを持つ要素を 1 つずつ入れる。採用済みでまとめた項目は、`act-core:write-plan` の「答えの反映」でまとめて保留・除外が選ばれない限り `adopted` とする。`confirm` の項目は `adopted` とする（適用はされない）。

| キー | 値 |
| --- | --- |
| `item_id` | 項目 ID（`03_ingest-plan.json` の `item_id` と同じ） |
| `decision` | `adopted`（採用）／`held`（保留）／`excluded`（除外） |
| `override` | 短い修正の値。その項目の `payload` に浅くマージするオブジェクト（キーは `payload` のキー。例: `{"title": "…"}`）。修正が無ければ `null`。`replan_key` を持つ項目の修正は手順 5 で `overrides` に書いて計画を作り直すため、ここには入れない |

### Step 6 適用（`act-core:ingest-file-applier`）

- 起動前にユーザーへ「承認された N 件を共有フォルダと保存先に適用しています」と伝える（続きからの適用なら、その旨も添える）。
- 渡す引数: `plan-json-path`: `<workdir>/03_ingest-plan.json`、`approved-plan-path`: `<workdir>/04_approved-plan.json`、`context-json-path`: `<workdir>/02_context.json`、`workdir`: `<workdir>`
- 戻り値: `<workdir>/06_apply-result.md` の絶対パス 1 行（同時に `<workdir>/05_apply-log.jsonl` が書かれる）
- 採用された項目だけを段の順に適用する。同じ `<workdir>` を渡すと、成功済みの項目は飛ばされる（続きからの適用はこれによる）。
- 項目の失敗では止まらない。失敗した項目に依存する項目は書かずに失敗とし、他のファイルの項目は続ける。段 0 で接続先が食い違ったときと、続けられない失敗（アダプタが応答しないなど）のときだけ、適用済みの範囲を記録してその段で止まる。
- `06_apply-result.md` の `## 結果の要約` の先頭行は次のとおり。
  - 段 0 で止まった → `状態: 停止（段 0）`
  - 失敗した項目が 1 件以上ある、または途中で止まった → `状態: 停止（段 <最初に失敗した項目の段、または止まった段>）`
  - それ以外 → `状態: 完了`
- `03_ingest-plan.json` の `warnings` を `06_apply-result.md` の `## 警告` に計画時の警告として引き継ぐ。`item_id` のある警告は、その項目を適用したか、保留・除外で適用しなかったかを書き分ける（例: 構成の設定の判断依頼を作ったか、保留して作っていないか）。
- 最後に実行ログへ今回の実行を記録する（途中で止まったときも）。保留した項目は `### 実行ログの held の形` で `held` に入り、次回同じファイルを取り込んだときに再び計画に載る。実行ログへの記録は確認項目にしない（実行そのものの記録で、承認された適用の結果だけを書くため）。

### Step 7 結果の報告（メイン）

1. `06_apply-result.md` の `## 結果の要約`、`## 配置したファイル`、`## 置き場所の変更`、`## 保留した項目`、`## 警告` だけを読む。
2. ユーザーに次を示す。
   - 配置したファイルのパス（共有フォルダからの相対パス）と、メタ情報ページの URL
   - 置き場所の変更（作った置き場所と更新した索引。初回は設定した構成の木。無ければ「なし」）
   - 適用結果（成功・失敗・飛ばした件数）
   - 保留した項目と、採用するには同じファイルで `/act-core:ingest-file` を再実行して計画フォームで採用すること（再実行すると、保留した項目は計画に再び載る）
   - 警告（計画時の警告を含む。議事録らしいファイルの清書と抽出は C1 の担当であること、構成の設定を判断依頼にしたか保留したか、人の編集との衝突で止めたファイル、保存先のスキーマ版が古い、など。`## 警告` に書かれたとおりに示す）
   - `## 結果の要約` の先頭行が `状態: 完了` でなければ、原因を取り除いてから同じファイルで `/act-core:ingest-file` を再実行すると、Step 3 で「続きから適用する」を選べること
   - 実行ログの行の URL（記録できなかったときはその旨。`## 結果の要約`・`## 警告` にある）
   - 作業フォルダ `<workdir>` の絶対パス

## 失敗時の対応

- Step 2・4 の戻り値が `ERROR:` で始まったら、次のステップに進まず、内容をユーザーにそのまま示して終える。まだ何も書いていないことを伝え、入力（ファイルのパス、プロジェクトの指定、用途・関連タスクの指定）の見直しを促し、直したうえで `/act-core:ingest-file` を再実行するよう案内する。
- Step 6 の戻り値が `ERROR:` で始まったら（適用を始められなかったとき）、次を示して終える。
  - 共有フォルダにも保存先にも何も書いていないこと
  - `ERROR:` の内容と、取り除くべき原因（プラグインの導入・更新、作業フォルダのファイルの破損など）
  - 原因を取り除いて同じファイルで `/act-core:ingest-file` を再実行すると、Step 3 で前回承認した計画を続きから適用できること
  - 作業フォルダ `<workdir>` の絶対パス
- Step 6 が途中で止まった場合（`06_apply-result.md` の `## 結果の要約` の先頭行が `状態: 停止（段 <N>）` の場合）は Step 7 で報告する。同じファイルで再実行すると Step 3 の手順 4 がその作業フォルダを見つけ、続きから適用するかを尋ねる。
- Step 3 の前に `act-core:write-plan` を読み込めなければ、何も書かずに、足りないものを示して終える。前提（個人設定・`project.yaml`・アダプタ・`act-core:store-protocol`）の不足はサブエージェントが `判定: 中止`（Step 3 手順 2・Step 5 手順 2）か `ERROR:`（上の項目）で返す。
