---
name: store-protocol
description: act-core のワークフローやサブエージェントが保存先アダプタ（act-store-notion、act-store-localfs など）を呼ぶときの規約。保存先の種類からアダプタを選ぶ規則、操作（describe・inspect・plan・apply・query・get・create・update・link・comment・check_permission・put_file・get_file）の呼び出し形式と入出力、正準モデルの JSON の形、正準エラーの扱い、書き込み前の接続先の照合を定める。act-core のスキル・エージェントが「act-core:store-protocol に従って」と指示されたときに読む。依頼例:「保存先のアダプタの呼び方は」「inspect の入出力は」「permission_denied が返ったらどうする」「/act-core:store-protocol」。
---

# store-protocol — 保存先アダプタの呼び出し規約

act-core（コア）は保存先を知らない。保存先への読み書きはすべて、ここで定める操作をアダプタに依頼して行う。アダプタは呼び出し元のワークフローを知らない。どの操作を、いつ、何のために呼ぶかは呼び出し元のスキル・エージェントが決める。

## アダプタの選び方

- 保存先の種類（`adapter`）ごとに、アダプタのプラグインとスキルが 1 つ決まる: `notion` → `act-store-notion:store`、`localfs` → `act-store-localfs:store`。一般に `<adapter>` → `act-store-<adapter>:store`。
- 型ごとの保存先は `<root>/.act/project.yaml` の `stores.<型>.adapter` で決まる。`project.yaml` がまだ無いとき（初期設定の新規作成）は、呼び出し元が渡す保存先の種類を使う。
- アダプタのスキルを `Skill` ツールで読み込み、その指示に従って操作を実行する。読み込めないとき（プラグインが入っていない）は、操作をせずに呼び出し元へ「`act-store-<adapter>` プラグインが無い」と返す。代わりの手段（コネクタのツールを直接呼ぶなど）で保存先に書かない。

## 操作の呼び出し形式

依頼は次の JSON 1 つで渡し、結果も JSON 1 つで受け取る。

```json
{ "op": "<操作名>", "adapter": "<adapter>", "args": { ... } }
```

```json
{ "ok": true, "result": { ... } }
{ "ok": false, "error": { "code": "<正準エラー>", "message": "<人が読める説明>", "item": "<失敗した対象の ID（分かれば）>" } }
```

## 正準モデルの JSON

エンティティは `attributes`（属性: 絞り込み・機械処理に使う値）と `narrative`（記述: 本文のセクション名 → 内容）に分ける。

```json
{
  "type": "task",
  "id": "FAQ-TSK-1",
  "attributes": { "status": "in_progress", "due": "2026-11-30", "assignee": "member:suzuki" },
  "narrative": {
    "status_summary": "…",
    "done_criteria": [ { "text": "…", "checked": false } ],
    "progress_log": [ { "date": "2026-10-22", "text": "…" } ]
  }
}
```

型は `decision`、`task`、`knowledge`、`artifact_meta`、`run_log`、`daily_summary`、`project_registry`、`client`、`tag`、`membership`、`workflow_config`。

判断依頼は `task` の `kind: request`（`kind` は `defined`／`auxiliary`／`request`）で表し、属性 `to_role`（宛先の役割: `manager`／`business_owner`）・`request_result`（判断の結果: 承認／却下）・`affected_tasks`（影響するタスクの ID）を持つ。

## 操作一覧

| 操作 | 入力（`args`） | 出力（`result`） |
| --- | --- | --- |
| `describe` | なし | `adapter`、`schema_version`（このアダプタが目標とする版）、`types`（対応する型）、`ops`（対応する操作）、`permission_check`（権限の確認方法） |
| `inspect` | `location`、`project_key`、`project_name`（下記） | 下記 |
| `plan` | `inspection`（`inspect` の結果）、`target_schema_version`、`mode`（`新規作成`／`取り込み`／`更新`）、`mapping`（任意）、`shared_choice`（共通 DB の候補が複数あったとき、または印の無い共通 DB の取り込みの候補があったときに選んだもの。候補の場所か、取り込まずに新しく作る `new`）、`project_profile`（任意。`name`・`client`。プロジェクトの保存先を新しく作るとき、トップページなどの記録の雛形に使う） | `changes`、`mapping_proposal`、`manual_steps`、`prefix_conflicts`、`prefix_unchecked`（下記） |
| `apply` | `changes`（承認された変更だけ）、`mapping`（確定した対応づけ。取り込み・更新のとき）、`run_id`（実行の識別子。呼び出し元の作業フォルダ名など）、`project_key` | `results`（変更 ID ごとに `success`／`failed`／`already_applied`）、`schema_version_before`（適用前の版）、`schema_version`（適用後の版）、`store_ids`（型ごとの DB 識別子） |
| `clear_migration` | `run_id`（止まった実行の識別子） | `{ "cleared": true, "run_id" }`。止まった移行の印を外す（下記「移行中の印」） |
| `query` | `type`、`filter`（属性の条件: 状態・担当・期間・キーワード・関連 ID・名前・別名） | エンティティの配列（`attributes` だけ。`narrative` は含めない） |
| `get` | `type`、`id` | エンティティ（`narrative` を含む） |
| `create` | `type`、`entity` | `id`、`url` |
| `update` | `type`、`id`、`attributes`（変更する属性）、`sections`（セクション名 → `{ "mode": "replace" \| "append", "value": … }`） | 更新後のエンティティ |
| `link` | `from`、`to`、`relation` | なし |
| `comment` | `type`、`id`、`body` | なし |
| `check_permission` | `type`、`op`、`id`（任意） | `allowed`（`true`／`false`／`unknown`）、`reason` |
| `put_file` / `get_file` | `path`、`content`／`version` | `location`、`hash` |

`query` は属性だけで絞り込み、属性だけを返す。記述が要るときだけ `get` で取る。

### 対応づけ（`mapping`）は読み替えに使う

取り込みで既存の DB を ACT の管理下に置いたとき、既存のプロパティ名は変えない。`mapping`（型ごとに、正準項目 → 既存プロパティ名）で読み替える。

- 呼び出し元は、`query`・`get`・`create`・`update`・`link` を含むすべての操作の `args` に、その型の `mapping` を渡す（`project.yaml` の `stores.<型>.mapping`。空なら省略してよい）。
- アダプタは、`mapping` にある正準項目はその既存プロパティ名で読み書きし、無い項目は自分のスキーマの名前を使う。
- 取り込みの `plan` では、「名前を変えて使う」項目はプロパティ名を変えずに `mapping` に載せるだけにする（`変換` にしない）。

### `inspect`

- 入力:
  - `location`: 次のどれでもよい。アダプタは値の形から判別する。
    - Notion: ACT のルートページ（共通 DB とプロジェクトのトップページを置く親ページ）か、プロジェクトのトップページの URL、または DB／データソースの識別子（`project.yaml` の `stores.<型>.database`）。
    - localfs: 共有フォルダのパス（`root`）。
  - `project_key`、`project_name`: 対象のプロジェクト。アダプタはこのキーのプロジェクトの保存先だけをプロジェクト側として扱い、同じ場所にある別のプロジェクトの保存先を取り違えない。
- 出力:
  - `project`: 対象プロジェクトの保存先。
    - `managed`: このプロジェクトの ACT の記録（スキーマ版などの記録の置き場所）があるか
    - `recorded_schema_version`: 記録された版。記録はあるが版が無い（前回の適用が途中で止まった）、または記録が無いときは `null`
    - `partial`: 記録はあるが版が無いとき `true`（適用途中）
    - `store_ids`: このプロジェクトの型ごとの DB 識別子（ACT の管理下にあるものだけ）
    - `missing_types`: 現在の版の構成にあってまだ無い型
  - `shared`: 全プロジェクト共通の保存先（Notion ならクライアント・プロジェクト一覧・タグ・ナレッジ）。
    - `store_ids`: 型ごとの識別子。見つからなければ空
    - `recorded_schema_version`: 共通の保存先に記録された版。無ければ `null`
    - `candidates`: 起点の近くで見つからず、離れた場所で 2 つ以上見つかったときの候補（それぞれの場所と `store_ids`）。1 つに決まったときは空
    - `adoptable`: ACT の印の無い既存の共通の保存先を、アダプタのスキーマと照合して取り込めそうなもの（取り込みの候補）。各要素は `location`、`store_ids`（型ごと）、`match`（型ごとの照合結果 `一致`／`近い` と、足りないもの・食い違うもの・根拠）。印のある共通の保存先が見つかったとき、または共通の保存先を持たないアダプタでは空
  - `databases`: 見つかった DB の配列。各要素は `role`（推定した型。分からなければ `null`）、`name`、`id`、`properties`（名前と型）、`template_headings`、`scope`（`project`／`shared`／`unknown`）
  - `migration`: `{ "in_progress": 真偽, "since": 日時, "run_id": 識別子 }`（下記「移行中の印」）
  - `workspace`: 接続先の識別子（Notion ならワークスペース ID、localfs なら `root`）
  - `user`: 接続しているアカウントの `name`・`email`・`id`（分かる範囲で）
  - `id_prefixes`（任意）: 保存先の中で一意でなければならない ID の接頭辞を持つアダプタ（Notion）が、使用済みの接頭辞を返す。各要素は `prefix`（読めなければ `null`）、`database`、`name`、`url`、`marked`（ACT の印があるか）、`type`、`source`（どこから読んだか）。あわせて `id_prefix_check`（`checked`・`unknown`・`complete`）を返す。接頭辞の一意性の制約が無いアダプタは省略する

### `plan` の出力

- `changes`: 変更の配列。各要素は `change_id`、`target`（対象 DB）、`category`（`追加`／`変換`／`削除`／`手作業`）、`description`、`affected_pages`（影響するページ数）、`samples`（変換の見本。変換のときだけ）、`adopt`（印の無い既存の共通の保存先を取り込む変更のときだけ。`{ "type", "database" }`）。
- `mapping_proposal`: 取り込みのときの対応づけ案。各要素は `type`（型）、`canonical`（正準項目）、`existing`（既存プロパティ名。無ければ `null`）、`treatment`（`そのまま使う`／`名前を変えて使う`／`新規に追加する`／`ACT では使わない`）、`basis`（根拠）。
- `manual_steps`: API で設定できず人の作業が要る手順（行単位の権限ルールなど）。
- `prefix_conflicts`: 作ろうとした保存先の ID の接頭辞が使用済みで、その作成を `changes` から外したもの。各要素は `prefix`、`type`、`used_by`（使っている保存先の `database`・`name`・`url`・`marked`・`type`）、`resolution`（`adopt`＝その保存先を取り込めば解消／`change_key`＝プロジェクトキーを変えれば解消／`manual`＝人が保存先側で直すまで解消しない）。無ければ空。
- `prefix_unchecked`: 接頭辞を読めなかった保存先（空の DB など）。作るときに初めて衝突が分かることがある。無ければ空。
- 削除は既定で `changes` に入れず、`旧_` を付けて残す変更にする。人が追加したプロパティ・見出しは定義に無くても残す。

### `apply` の約束

- 冪等: 同じ `changes` を 2 回渡しても結果は変わらない。適用済みの変更は `already_applied` で返す。
- 途中で失敗したら、そこまでの結果を `results` に返す。続きは同じ `changes` で再び呼べば進む。
- 適用を始めるときに、まずプロジェクトの記録の置き場所（版は空のまま）を作る。版は全部の変更が終わってから書く。これで、途中で止まった保存先は `inspect` で `partial: true` になり、どこからやり直しても不足分（`missing_types`）を作り直せる。
- 移行中の印と版の記録はアダプタが行う。実行の記録（移行を含む）は呼び出し元が実行ログに残す（下記「実行ログ」）。

### 移行中の印

- `apply` は始めに、変更する保存先（プロジェクト側、共通側の両方）に `{ in_progress: true, since: <日時>, run_id: <args.run_id> }` を記録し、終わったら（失敗で止まったときも）外す。
- 印が残っているときの扱い:
  - 同じ `run_id` の `apply` は、印があっても続きから進めてよい。
  - 別の `run_id` の書き込みは `conflict` を返す。
  - 処理が切れて印が残ったときは、呼び出し元が人に確認したうえで `clear_migration`（止まった実行の `run_id` を渡す）を呼んで外す。外した後は、`inspect` → `plan` → `apply` で続きから適用できる。

## 実行ログ

保存先に書いたワークフローの実行を、型 `run_log` で 1 実行 1 件残す。`run_id` は `apply` に渡した実行の識別子（呼び出し元の作業フォルダ名など）。

- いつ書くか: 実行が終わったとき（途中で止まったときも）に 1 件 `create` する。保存先に何も書かずに終えた実行（中止、採用 0 件）は書かない。同じ `run_id` の続き（続きからの適用）は、その件を `update` で置き換える。ほかの件は変えない。
- 形:
  - 属性: `id`（`run_id`）、`workflow`（ワークフロー ID: C0〜C6／E1〜E5／M1〜M7）、`run_by`（メンバー）、`started_at`・`finished_at`、`result`（成功／一部失敗／失敗）、`scheduled`（定期実行か）、`applied_count`・`held_count`・`failed_count`、`schema_change`（移行したときだけ。例 `notion/1→notion/2`、`共通: notion/1→notion/2`）
  - 件数の数え方: `applied_count` は `apply` の `results` の `success` と `already_applied` の数、`failed_count` は `failed` の数（権限エラーで適用できなかった項目を含む）、`held_count` は承認されず保留と確定した項目の数。
  - `result` の決め方: `applied_count` が 0 なら失敗、1 以上で `failed_count` が 0 なら成功、どちらも 1 以上なら一部失敗。
  - 記述: `summary`（概要）、`applied`（適用した変更）、`held`（保留した項目）、`issues`（失敗・警告）
- 書く役: 呼び出し元のワークフロー。アダプタは実行の記録を書かず、記録に要る値（`apply` の `results`、`schema_version_before`・`schema_version`）を返すだけにする。共通の保存先だけの移行も、その実行をしたプロジェクトの実行ログに残す。
- 書かないもの: 秘密情報、実行者の PC のパス。
- 書けないとき: 実行ログの置き場所が無い（DB が未作成、アダプタが `run_log` に未対応）ときは、作業フォルダだけに残して警告する。

作業フォルダとの分担:

| | 作業フォルダ | 実行ログ |
| --- | --- | --- |
| 置き場所 | 実行者の PC | 保存先（チームで共有） |
| 中身 | 中間生成物、項目ごとの適用記録 | 1 実行 1 件の要約 |
| 使い道 | 本人の詳細確認、続きからの適用の起点 | 誰がいつ何をしたかをチームで見る。保留した項目を次回に示す |

## 本文の更新ルール

本文は人も直接書き換える。`update` では担当するセクションだけを変える。

| セクション | 更新方法 |
| --- | --- |
| 状況サマリ | `replace`（同時に `status_updated_at` を更新） |
| 進捗ログ、経緯 | `append`（日付付き）。過去の行は変えない |
| 完了条件 | チェック状態の変更だけ。条件の追加・変更はマネージャーの操作 |
| 実行ログ | 作成時に書く。同じ `run_id` の続きだけが置き換える |
| その他のセクション | 作成時に書く。以降の変更は書き込み計画で差分を示して確認を取る |
| テンプレートに無い見出し・段落 | 人が足したものとして残す。消さない |

## 正準エラー

| `code` | 意味 | 呼び出し元の既定の扱い |
| --- | --- | --- |
| `permission_denied` | 保存先が操作を拒否した | 失敗にせず、その項目を飛ばして警告にする。書き込みなら「提案」や「判断依頼」への切り替えを検討する |
| `not_found` | 対象が無い | 失敗として扱う |
| `conflict` | 移行中・同時更新などで今は書けない。または ID の接頭辞が使用済みで作れない | 失敗として扱い、時間をおいて再実行するよう案内する。`message` が ID の接頭辞の使用済みを示すときは、再実行では解消しないので、`message` の使用元を直すよう案内する |
| `unsupported` | アダプタがその型・操作に対応していない | 失敗として扱い、対応するアダプタの版を案内する |
| `invalid_request` | 入力の形が規約に合わない | 呼び出し元の不具合。失敗として扱う |

呼び出し元に別の定めがあれば、そちらを優先する。

## 書き込み前の接続先の照合

保存先の接続（Notion コネクタなど）は 1 つのワークスペースにしかつながらないことがある。書き込み（`apply`・`create`・`update`・`link`・`comment`・`put_file`）の前に、`inspect` の `workspace` を、照合する値と比べる。違えば書かずに止め、接続先を切り替えるよう案内する。

- 照合する値は、ふだんは個人設定 `~/.act/config.yaml` の `projects.<キー>.accounts.<adapter>.workspace`。
- 初期設定のように個人設定にまだその値が無いときは、呼び出し元が持つ値（初期設定なら変更計画の `accounts.<adapter>.workspace`）を使う。この場合の照合は「計画を作った後に接続先が変わっていないか」の確認になる。
- どちらの値も無ければ、照合できない旨を警告して止める。

## してはいけないこと

- 認証情報（トークン・パスワード）を、依頼・結果・ログ・設定ファイルに書かない。保存先への認証は各コネクタの接続に任せる。
- アダプタを通さずに保存先へ書かない。
- 承認されていない `changes` を `apply` に渡さない。
