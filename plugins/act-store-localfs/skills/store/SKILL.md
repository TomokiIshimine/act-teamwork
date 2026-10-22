---
name: store
description: ACT の保存先アダプタ（ローカルファイル）。プロジェクトの共有フォルダ（Google ドライブ等の同期先）にアーティファクト本体を置く。act-core のスキル・エージェントが act-core:store-protocol に従って保存先の種類 localfs の操作（describe・inspect・plan・apply・put_file・get_file・check_permission）を依頼したときに読み、共有フォルダで実行して JSON で返す。依頼例:「act-store-localfs で共有フォルダを inspect して」「議事録の本体を共有フォルダに put_file して」「/act-store-localfs:store」。
---

# store — ローカルファイルアダプタ

`act-core:store-protocol` の操作を、プロジェクトの共有フォルダ（`root`）で実行する。扱う型はアーティファクト本体（`artifact_body`）だけ。メタ情報は別のアダプタ（Notion など）が持つ。

## 前提

- `root` は共有フォルダの絶対パス。呼び出し元が渡す（`inspect` の `location`、それ以外は `args.root`）。
- ファイルの読み書きの手段: このセッションから直接読み書きできるパスは `Read`／`Write`／`Bash` を使う。Cowork で `root` がメンバーの PC の接続フォルダにあるときは `mcp__remote-devices__device_bash` を使う（接続フォルダは `$HOME/mnt/<フォルダ名>` に見えるので、渡されたパスを読み替える。`device_bash` の `~` はセッション側のホーム）。どちらでも読めなければ `not_found`、書けなければ `permission_denied` を返す。
- 現在の版は `1`。
- 共有フォルダの外には書かない。ファイルを削除しない。既存のファイルを上書きしない（下記の版の規則で別名にする）。

## 版 1 の構成

```
<root>/
├── .act/
│   ├── project.yaml          （act-core が書く。このアダプタは読まない・書かない）
│   ├── store-localfs.json    このアダプタの記録
│   └── store-localfs.log     移行ログ（1 行 1 回の追記）
├── minutes/                  議事録
├── inputs/                   入力（要望書・データなど。種類ごとのサブフォルダ）
├── work/                     作業途中
└── deliverables/             成果物（版付き）
```

`store-localfs.json`: `{ "schema": "localfs/1", "project_key": "FAQ", "migrating": false, "migrating_since": null, "run_id": null, "layout": { "minutes": "minutes/{date}_{title}.md", "input": "inputs/{kind}/{title}{ext}", "working": "work/{title}{ext}", "deliverable": "deliverables/{title}_{version}{ext}" } }`

配置規則（`layout`）は `project.yaml` の `stores.artifact_body.layout` が正。呼び出し元が `args.layout` で渡せばそれを使い、無ければ上の既定を使う。

## 操作

### `describe`

`{ "adapter": "localfs", "schema_version": 1, "types": ["artifact_body"], "ops": ["describe","inspect","plan","apply","clear_migration","put_file","get_file","check_permission"], "permission_check": "共有フォルダへの書き込みを、このアダプタ専用の固定ファイルで試す" }` を返す。

### `inspect`

`location`（`root`）を読み、次を返す。何も書かない。共有フォルダはプロジェクトごとに別なので、`project_key` は照合だけに使う（`<root>/.act/store-localfs.json` の `project_key` が違えば、`project` に `mismatch: true` を付ける。呼び出し元はこれを警告にする）。

- `project`:
  - `managed`: `<root>/.act/store-localfs.json` があれば `true`
  - `recorded_schema_version`: その `schema` の版。ファイルはあるが `schema` が空なら `null`
  - `partial`: ファイルはあるが `schema` が空なら `true`
  - `store_ids`: `{ "artifact_body": "<root>" }`（管理下のときだけ）
  - `missing_types`: 版 1 の構成のフォルダのうち無いものがあれば `["artifact_body"]`、無ければ空
- `shared`: `{ "store_ids": {}, "recorded_schema_version": null, "candidates": [] }`（共通の保存先は持たない）
- `databases`: 版 1 の構成のフォルダごとに `{ "role": "artifact_body", "name": <フォルダ名>, "id": <root からの相対パス>, "properties": [], "template_headings": [], "scope": "project" }`（あるものだけ）。版 1 の構成に無い直下のフォルダは `scope: unknown` で載せる（取り込みの材料）
- `migration`: `store-localfs.json` の `migrating`・`migrating_since`・`run_id`
- `workspace`: `root`（照合用。共有フォルダのパス）
- `user`: `null`

`root` が無ければ `not_found`。

### `plan`

- **新規作成・適用途中**: 版 1 の構成の無いフォルダと `.act/store-localfs.json` の作成を `追加` として並べる。既にあるフォルダは載せない。
- **取り込み**: 既存のフォルダを `layout` の各分類に対応づける案を `mapping_proposal` にする（例: 既存の `議事録/` を `minutes` に使う）。対応づけたフォルダは作らず、`layout` をその名前で書く変更にする。既存のファイルは移動しない。
- **更新**: 1 版ずつ移行手順を並べる（版 1 は最初の版なので、版 1 への更新は無い）。
- 削除・移動は載せない。`manual_steps` は空。

### `apply`

1. `.act/store-localfs.json` を（無ければ `schema` を空にして）用意し、`migrating: true`、`migrating_since`、`run_id`、`project_key` を書く。既に別の `run_id` で `migrating: true` なら何もせず `conflict` を返す（同じ `run_id` なら続ける）。
2. 各変更を順に適用する。フォルダは既にあれば `already_applied`。
3. 失敗した変更があれば、そこで止める。
4. 終わったら（止まったときも）`migrating` を `false` に戻す。全部成功したときだけ `schema` に `localfs/1` と確定した `layout` を書く。`store-localfs.log` に 1 行追記する（日時、`run_id`、前後の版、適用した変更 ID、失敗）。
5. `{ "results", "schema_version", "store_ids": { "artifact_body": "<root>" } }` を返す。

### `clear_migration`

`store-localfs.json` の `run_id` が渡された `run_id` と同じなら `migrating` を `false` にし、ログに「印を外した」を追記する。違えば何もせず `conflict` を返す。

### `put_file`

`args`: `root`、`category`（`minutes`／`input`／`working`／`deliverable`）、`fields`（`date`・`title`・`kind`・`ext`・`version` のうち `layout` が使うもの）、`content`（テキスト）または `source_path`（コピー元のファイル）。

1. `layout` から相対パスを作る。ファイル名に使えない文字（`/ \ : * ? " < > |`）は全角に置き換える。
2. 同じパスにファイルがあれば、内容のハッシュが同じなら書かずにそのパスを返す（二重取り込みの防止）。違えば上書きせず、`deliverable` は `version` を上げるよう `conflict` を返し、それ以外は末尾に `_2`、`_3`… を付けた名前で書く。
3. 書いた内容の SHA-256 を計算する。
4. `{ "location": <root からの相対パス>, "hash": "sha256:<16 進>" }` を返す。

### `get_file`

`args`: `root`、`location`（相対パス）。ファイルを読み、テキストなら `content` を、そうでなければ `size` と `hash` を返す。無ければ `not_found`。

### `check_permission`

`<root>/.act/.write-test`（このアダプタ専用の固定名。毎回同じファイルに日時を書き直す。上書きしてよいのはこのファイルだけ）を書けるかで書き込みの可否を決め、`{ "allowed": true|false, "reason": ... }` を返す。`.act/` が無ければ `root` を書き込めるかを、ファイルを作らずに判定できる手段（`test -w` など）で確かめる。

`query`・`get`・`create`・`update`・`link`・`comment` は扱わない（`unsupported`）。アーティファクトのメタ情報は別のアダプタが持つ。

## エラーの変換

| 状況 | 正準エラー |
| --- | --- |
| 書き込みが拒否された（読み取り専用、権限なし、接続フォルダの外） | `permission_denied` |
| `root` やファイルが無い | `not_found` |
| 移行中の印が別の `run_id` で付いている、同名の成果物が別の内容で既にある | `conflict` |
| 扱わない操作・型 | `unsupported` |
| 入力の形が合わない | `invalid_request` |
