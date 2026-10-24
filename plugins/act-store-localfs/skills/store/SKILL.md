---
name: store
description: ACT の保存先アダプタ（ローカルファイル）。プロジェクトの共有フォルダ（Google ドライブ等の同期先）にアーティファクト本体を置く。act-core のスキル・エージェントが act-core:store-protocol に従って保存先の種類 localfs の操作（describe・inspect・plan・apply・put_file・get_file・check_permission・list_folders・create_folder・update_index）を依頼したときに読み、共有フォルダで実行して JSON で返す。依頼例:「act-store-localfs で共有フォルダを inspect して」「議事録の本体を共有フォルダに put_file して」「/act-store-localfs:store」。
---

# store — ローカルファイルアダプタ

`act-core:store-protocol` の操作を、プロジェクトの共有フォルダ（`root`）で実行する。扱う型はアーティファクト本体（`artifact_body`）だけ。メタ情報は別のアダプタ（Notion など）が持つ。置き場所（テーマ別の階層）と索引の実体は、このアダプタではディレクトリと各階層の `README.md` にする（書式は下記「置き場所の索引（`README.md`）」）。

## 前提

- `root` は共有フォルダの絶対パス。呼び出し元が渡す（`inspect` の `location`、それ以外は `args.root`）。
- ファイルの読み書きの手段: このセッションから直接読み書きできるパスは `Read`／`Write`／`Bash` を使う。Cowork で `root` がメンバーの PC の接続フォルダにあるときは `mcp__remote-devices__device_bash` を使う（接続フォルダは `$HOME/mnt/<フォルダ名>` に見えるので、渡されたパスを読み替える。`device_bash` の `~` はセッション側のホーム）。どちらでも読めなければ `not_found`、書けなければ `permission_denied` を返す。
- 現在の版は `1`。
- `folder_root`（テーマ別の置き場所の根。`root` からの相対パス）と `layout` の正は `project.yaml` の `stores.artifact_body`。呼び出し元が `args.folder_root`・`args.layout` で渡す。`folder_root` が渡されなければ `library` とみなす。
- 共有フォルダの外には書かない。ファイルを削除しない。フォルダの改名・移動・削除をしない。既存のファイルを上書きしてよいのは作業途中（`working`）と、置き場所の `README.md` の `## 一覧` の表だけ。それ以外は下記の版の規則で別名にする（要件 §4.6「版の残し方」）。

## 版 1 の構成

```
<root>/
├── .act/
│   ├── project.yaml          （act-core が書く。このアダプタは読まない・書かない）
│   └── store-localfs.json    このアダプタの記録
├── minutes/                  議事録
├── inputs/                   入力（要望書・データなど。種類ごとのサブフォルダ）
├── work/                     作業途中
├── deliverables/             成果物（版付き）
└── <folder_root>/            テーマ別の置き場所の根（既定 library。構成の設定後だけ）
    ├── README.md             索引（目的と直下の一覧）
    └── <テーマ>/              第 1 階層（根から 3 階層まで）
        ├── README.md
        └── <テーマ>/
            └── README.md
```

`<folder_root>/` は `plan`／`apply` では作らない。呼び出し元が最初に `create_folder` を呼んだときに作る（下記 `create_folder`）。根を作る前に `inputs/`・`work/`・`deliverables/` に置いたファイルは動かさない。

以前の版で作った `.act/store-localfs.log` は消さない（読まない・書かない）。実行の記録は呼び出し元が実行ログに残す。

`store-localfs.json`: `{ "schema": "localfs/1", "project_key": "FAQ", "migrating": false, "migrating_since": null, "run_id": null, "layout": { "minutes": "minutes/{date}_{title}.md", "input": "inputs/{kind}/{title}{ext}", "working": "work/{title}{ext}", "deliverable": "deliverables/{title}_{version}{ext}" } }`

配置規則（`layout`）は `project.yaml` の `stores.artifact_body.layout` が正。呼び出し元が `args.layout` で渡せばそれを使い、無ければ上の既定を使う。`layout` の変数は `{date}`・`{title}`・`{kind}`・`{ext}`・`{version}` と `{folder}`（`folder_root` からの `/` 区切りの相対パス。例: `library/{folder}/{title}{ext}`）。スキーマ版は `1` のまま（`store-localfs.json` の形は変えない）。

## 操作

### `describe`

`{ "adapter": "localfs", "schema_version": 1, "types": ["artifact_body"], "ops": ["describe","inspect","plan","apply","clear_migration","put_file","get_file","check_permission","list_folders","create_folder","update_index"], "permission_check": "共有フォルダへの書き込みを、このアダプタ専用の固定ファイルで試す" }` を返す。

### `inspect`

`location`（`root`）を読み、次を返す。何も書かない。共有フォルダはプロジェクトごとに別なので、`project_key` は照合だけに使う（`<root>/.act/store-localfs.json` の `project_key` が違えば、`project` に `mismatch: true` を付ける。呼び出し元はこれを警告にする）。

- `project`:
  - `managed`: `<root>/.act/store-localfs.json` があれば `true`
  - `recorded_schema_version`: その `schema` の版。ファイルはあるが `schema` が空なら `null`
  - `partial`: ファイルはあるが `schema` が空なら `true`
  - `store_ids`: `{ "artifact_body": "<root>" }`（管理下のときだけ）
  - `missing_types`: 版 1 の構成のフォルダのうち無いものがあれば `["artifact_body"]`、無ければ空
- `shared`: `{ "store_ids": {}, "recorded_schema_version": null, "candidates": [], "adoptable": [] }`（共通の保存先は持たない）
- `databases`: 版 1 の構成のフォルダごとに `{ "role": "artifact_body", "name": <フォルダ名>, "id": <root からの相対パス>, "properties": [], "template_headings": [], "scope": "project" }`（あるものだけ）。`args.folder_root` の置き場所の根があれば、`{ "role": "artifact_body", "name": <folder_root>, "id": <folder_root>, "properties": [], "template_headings": [], "scope": "project" }` で載せる（根が無くても `missing_types` には含めない。根は `create_folder` が作るため）。それ以外で版 1 の構成に無い直下のフォルダは `scope: unknown` で載せる（取り込みの材料）
- `migration`: `store-localfs.json` の `migrating`・`migrating_since`・`run_id`
- `workspace`: `root`（照合用。共有フォルダのパス）
- `user`: `null`

`root` が無ければ `not_found`。

### `plan`

- **新規作成・適用途中**: 版 1 の構成の無いフォルダと `.act/store-localfs.json` の作成を `追加` として並べる。既にあるフォルダは載せない。
- **取り込み**: 既存のフォルダを `layout` の各分類に対応づける案を `mapping_proposal` にする（例: 既存の `議事録/` を `minutes` に使う）。対応づけたフォルダは作らず、`layout` をその名前で書く変更にする。既存のファイルは移動しない。
- **更新**: 1 版ずつ移行手順を並べる（版 1 は最初の版なので、版 1 への更新は無い）。
- 削除・移動は載せない。`manual_steps`・`prefix_conflicts`・`prefix_unchecked` は空（ID の接頭辞を持たない）。

### `apply`

1. `.act/store-localfs.json` を（無ければ `schema` を空にして）用意し、`migrating: true`、`migrating_since`、`run_id`、`project_key` を書く。既に別の `run_id` で `migrating: true` なら何もせず `conflict` を返す（同じ `run_id` なら続ける）。
2. 各変更を順に適用する。フォルダは既にあれば `already_applied`。
3. 失敗した変更があれば、そこで止める。
4. 終わったら（止まったときも）`migrating` を `false` に戻す。全部成功したときだけ `schema` に `localfs/1` と確定した `layout` を書く。
5. `{ "results", "schema_version_before", "schema_version", "store_ids": { "artifact_body": "<root>" } }` を返す。

### `clear_migration`

`store-localfs.json` の `run_id` が渡された `run_id` と同じなら `migrating` を `false` にし、`{ "cleared": true, "run_id" }` を返す。違えば何もせず `conflict` を返す。

### `put_file`

`args`: `root`、`category`（`minutes`／`input`／`working`／`deliverable`）、`fields`（`date`・`title`・`kind`・`ext`・`version`・`folder` のうち `layout` が使うもの。`folder` は `layout` が `{folder}` を使うときの置き場所の `folder_root` からの相対パス）、`content`（テキスト）または `source_path`（コピー元のファイル）、`expected_hash`（任意。`working` を上書きするとき、呼び出し元が前に読んだ・書いたときのハッシュ）、`layout`・`folder_root`（上記「前提」）。

レビューに出す・承認・提出の節目で版を固定するときは、`category: deliverable`、`fields.version` に新しい版、`source_path` に作業途中のファイルを渡して版付きの写しを作る。

1. `layout` から相対パスを作る。ファイル名に使えない文字（`/ \ : * ? " < > |`）は全角に置き換える（`{folder}` の区切りの `/` はそのまま使い、各階層の名前の中だけを置き換える）。`{folder}` の置き場所が無ければ書かずに `not_found` を返す（置き場所は先に `create_folder` で作る）。
2. 同じパスにファイルがあれば、内容のハッシュが同じなら書かずにそのパスを返す（同じ内容を二重に置かない）。違えば分類ごとに次のようにする。
   - `working`: `expected_hash` が渡されていて今のファイルのハッシュと同じときだけ上書きする。`expected_hash` と違えば人が後から編集したので、書かずに `conflict` を返す（`current_hash` を添える）。`expected_hash` が無いときも上書きせず `conflict` を返す（同じフォルダに用途の違う同名ファイルが並ぶため、別のファイルを上書きしないようにする）
   - `deliverable`: 上書きせず、`version` を上げるよう `conflict` を返す（版付きの写しは以後変えない）
   - `minutes`・`input`: 上書きせず、末尾に `_2`、`_3`… を付けた名前で書く（清書した議事録と入力の原本は確定扱い）
3. 書いた内容の SHA-256 を計算する。
4. `{ "location": <root からの相対パス>, "hash": "sha256:<16 進>" }` を返す。`working` を上書きしたときは `previous_hash`（上書き前のハッシュ）も返す。

### `get_file`

`args`: `root`、`location`（`root` からの相対パス）。ファイルを読み、テキストなら `content` を、そうでなければ `size` と `hash`（`put_file` と同じ形）を返す。無ければ `not_found`。

### `check_permission`

`<root>/.act/.write-test`（このアダプタ専用の固定名。毎回同じファイルに日時を書き直す。上書きしてよいのはこのファイルだけ）を書けるかで書き込みの可否を決め、`{ "allowed": true|false, "reason": ... }` を返す。`.act/` が無ければ `root` を書き込めるかを、ファイルを作らずに判定できる手段（`test -w` など）で確かめる。

### `list_folders`

`args`: `root`、`folder_root`。`<root>/<folder_root>/` の下の置き場所（ディレクトリ）を根から 3 階層までたどり、`{ "folders": [{ "path", "purpose", "files", "folders" }] }` を返す。何も書かない。

- `path`: `folder_root` からの `/` 区切りの相対パス（根そのものは含めない）
- `purpose`: そのフォルダの `README.md` の `## 目的` の 1 行目。`README.md` か `## 目的` が無ければ `null`
- `files`・`folders`: 直下のファイル数（`README.md` を除く）とフォルダ数

`.` で始まる名前のフォルダはたどらない。`<root>/<folder_root>/` が無ければ `not_found`。

### `create_folder`

`args`: `root`、`folder_root`、`path`（`folder_root` からの相対パス）、`purpose`（目的の 1 行）。

1. 名前のファイル名に使えない文字は `put_file` と同じく全角に置き換える。
2. 根（`<root>/<folder_root>/`）が無ければ、先に根と根の `README.md` を作る（目的は「このプロジェクトのアーティファクト本体をテーマ別に置く」）。
3. 親のフォルダが無ければ書かずに `not_found` を返す（呼び出し元は親から順に作る）。
4. フォルダが既にあれば何も作らず `already_applied` を返す（`README.md` が無ければ作る）。無ければディレクトリと、`purpose` を目的にした `README.md`（下記の書式、`## 一覧` は空の表）を作る。
5. 親の `README.md` の `## 一覧` に、このフォルダの行（`type: folder`、`summary_line` は `purpose`）を `update_index` と同じ規則で足す。
6. `{ "path": <folder_root からの相対パス> }` を返す。

### `update_index`

`args`: `root`、`folder_root`、`folder`（`folder_root` からの相対パス。根は `""`）、`upsert`（索引の行の配列。行の形は `act-core:store-protocol` の「索引の行」）、`purpose`（`README.md` が無いときだけ使う）。

1. `<folder>/README.md` を読む。無ければ `purpose` を目的にして下記の書式で作る（`purpose` も無ければ `invalid_request`）。
2. `## 一覧` の表だけを書き換える。`upsert` の行を `name` で探し、あれば置き換え、無ければ足す。
3. 表と直下を突き合わせる: 表に無い直下のファイル・フォルダ（`README.md` と `.` で始まる名前を除く）は、内容を空、状態を「未登録」にした行を足す。表にあって直下に無いものは、その行の状態を「見つからない」にする（行は消さない）。
4. 他の見出し（`# <フォルダ名>`・`## 目的`・`## メモ` など）と、人の書いた文は変えない。
5. `{ "folder", "hash": "sha256:<16 進>" }`（書いた `README.md` のハッシュ）を返す。表が変わらなければ書かずに同じ形で返す。

フォルダが無ければ `not_found`。

`query`・`get`・`create`・`update`・`link`・`comment` は扱わない（`unsupported`）。アーティファクトのメタ情報は別のアダプタが持つ。

## 置き場所の索引（`README.md`）

置き場所の各階層（根を含む）に置く。書式は次のとおり。

```markdown
# <フォルダ名>

## 目的

<目的の 1 行>（2 行目以降に人が詳しい説明を書いてよい）

## 一覧

> この表は ACT が更新します。メモは表の外（`## メモ`）に書いてください。

| 名前 | 内容 | 種類 | 状態 | 版 | 更新日 | メタ情報 |
| --- | --- | --- | --- | --- | --- | --- |
| 要件定義書.docx | 顧客要件の一覧と優先度 | 要望書 | 下書き | v1 | 2026-10-24 | [ページ](<meta_url>) |
| 画面設計/ | 画面の設計とワイヤーフレーム | | | | | |

## メモ
```

- 列と索引の行（`act-core:store-protocol` の「索引の行」）の対応: 名前 = `name`（`type` が `folder` なら末尾に `/` を付ける）、内容 = `summary_line`、種類 = `kind`、状態 = `status`、版 = `version`、更新日 = `updated`、メタ情報 = `meta_url` へのリンク。値が `null` の列は空にする。
- 表の中の `|` は全角に置き換える。行は名前の順（フォルダを先）に並べる。
- 根の `README.md` の見出しは `# <folder_root>` にする。

## エラーの変換

| 状況 | 正準エラー |
| --- | --- |
| 書き込みが拒否された（読み取り専用、権限なし、接続フォルダの外） | `permission_denied` |
| `root` やファイル、置き場所（フォルダ）・その親が無い | `not_found` |
| 移行中の印が別の `run_id` で付いている、同名の成果物が別の内容で既にある、作業途中のファイルが `expected_hash` の後に編集されている、作業途中の同じパスに別内容があり `expected_hash` が無い | `conflict` |
| 扱わない操作・型 | `unsupported` |
| 入力の形が合わない | `invalid_request` |
