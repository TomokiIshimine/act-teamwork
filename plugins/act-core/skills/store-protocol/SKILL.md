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

型は `decision`、`task`、`knowledge`、`artifact_meta`、`request`、`adoption`、`project_registry`、`client`、`tag`、`membership`、`workflow_config`。

## 操作一覧

| 操作 | 入力（`args`） | 出力（`result`） |
| --- | --- | --- |
| `describe` | なし | `adapter`、`schema_version`（このアダプタが目標とする版）、`types`（対応する型）、`ops`（対応する操作）、`permission_check`（権限の確認方法） |
| `inspect` | `location`、`project_key`、`project_name`（下記） | 下記 |
| `plan` | `inspection`（`inspect` の結果）、`target_schema_version`、`mode`（`新規作成`／`取り込み`／`更新`）、`mapping`（任意）、`shared_choice`（共通 DB の候補が複数あったときに選んだもの） | `changes`、`mapping_proposal`、`manual_steps`（下記） |
| `apply` | `changes`（承認された変更だけ）、`mapping`（確定した対応づけ。取り込み・更新のとき）、`run_id`（実行の識別子。呼び出し元の作業フォルダ名など）、`project_key` | `results`（変更 ID ごとに `success`／`failed`／`already_applied`）、`schema_version`（適用後の版）、`store_ids`（型ごとの DB 識別子） |
| `clear_migration` | `run_id`（止まった実行の識別子） | なし。止まった移行の印を外す（下記「移行中の印」） |
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
  - `databases`: 見つかった DB の配列。各要素は `role`（推定した型。分からなければ `null`）、`name`、`id`、`properties`（名前と型）、`template_headings`、`scope`（`project`／`shared`／`unknown`）
  - `migration`: `{ "in_progress": 真偽, "since": 日時, "run_id": 識別子 }`（下記「移行中の印」）
  - `workspace`: 接続先の識別子（Notion ならワークスペース ID、localfs なら `root`）
  - `user`: 接続しているアカウントの `name`・`email`・`id`（分かる範囲で）

### `plan` の出力

- `changes`: 変更の配列。各要素は `change_id`、`target`（対象 DB）、`category`（`追加`／`変換`／`削除`／`手作業`）、`description`、`affected_pages`（影響するページ数）、`samples`（変換の見本。変換のときだけ）。
- `mapping_proposal`: 取り込みのときの対応づけ案。各要素は `canonical`（正準項目）、`existing`（既存プロパティ名。無ければ `null`）、`treatment`（`そのまま使う`／`名前を変えて使う`／`新規に追加する`／`ACT では使わない`）、`basis`（根拠）。
- `manual_steps`: API で設定できず人の作業が要る手順（行単位の権限ルールなど）。
- 削除は既定で `changes` に入れず、`旧_` を付けて残す変更にする。人が追加したプロパティ・見出しは定義に無くても残す。

### `apply` の約束

- 冪等: 同じ `changes` を 2 回渡しても結果は変わらない。適用済みの変更は `already_applied` で返す。
- 途中で失敗したら、そこまでの結果を `results` に返す。続きは同じ `changes` で再び呼べば進む。
- 適用を始めるときに、まずプロジェクトの記録の置き場所（版は空のまま）を作る。版は全部の変更が終わってから書く。これで、途中で止まった保存先は `inspect` で `partial: true` になり、どこからやり直しても不足分（`missing_types`）を作り直せる。
- 適用中であることの表示（移行中の印）、保存先への版の記録、移行ログはアダプタが行う。

### 移行中の印

- `apply` は始めに、変更する保存先（プロジェクト側、共通側の両方）に `{ in_progress: true, since: <日時>, run_id: <args.run_id> }` を記録し、終わったら（失敗で止まったときも）外す。
- 印が残っているときの扱い:
  - 同じ `run_id` の `apply` は、印があっても続きから進めてよい。
  - 別の `run_id` の書き込みは `conflict` を返す。
  - 処理が切れて印が残ったときは、呼び出し元が人に確認したうえで `clear_migration`（止まった実行の `run_id` を渡す）を呼んで外す。外した後は、`inspect` → `plan` → `apply` で続きから適用できる。

## 本文の更新ルール

本文は人も直接書き換える。`update` では担当するセクションだけを変える。

| セクション | 更新方法 |
| --- | --- |
| 状況サマリ | `replace`（同時に `status_updated_at` を更新） |
| 進捗ログ、経緯 | `append`（日付付き）。過去の行は変えない |
| 完了条件 | チェック状態の変更だけ。条件の追加・変更はマネージャーの操作 |
| その他のセクション | 作成時に書く。以降の変更は書き込み計画で差分を示して確認を取る |
| テンプレートに無い見出し・段落 | 人が足したものとして残す。消さない |

## 正準エラー

| `code` | 意味 | 呼び出し元の既定の扱い |
| --- | --- | --- |
| `permission_denied` | 保存先が操作を拒否した | 失敗にせず、その項目を飛ばして警告にする。書き込みなら「提案」や「判断依頼」への切り替えを検討する |
| `not_found` | 対象が無い | 失敗として扱う |
| `conflict` | 移行中・同時更新などで今は書けない | 失敗として扱い、時間をおいて再実行するよう案内する |
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
