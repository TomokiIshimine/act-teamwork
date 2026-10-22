---
name: store
description: ACT の保存先アダプタ（Notion）。act-core のスキル・エージェントが act-core:store-protocol に従って保存先の種類 notion の操作（describe・inspect・plan・apply・query・get・create・update・link・comment・check_permission）を依頼したときに読み、本人の Notion コネクタで実行して正準モデルの JSON で返す。依頼例:「act-store-notion で inspect して」「Notion の保存先に task を create して」「Notion の ACT スキーマを最新にする計画を作って」「/act-store-notion:store」。
---

# store — Notion アダプタ

`act-core:store-protocol` の操作を Notion で実行する。呼び出し形式（`{ "op", "adapter", "args" }` → `{ "ok", "result" | "error" }`）、正準モデルの JSON、正準エラーはその規約に従う。ここでは Notion での実現だけを定める。呼び出し元のワークフローの事情（なぜその操作を呼ぶか）は扱わない。

Notion の構成（DB・プロパティ・本文テンプレート・ACT の印・正準モデルとの対応）は `references/schema-v1.md` に定める。`describe`・`inspect`・`plan`・`apply` を行うとき、または型とプロパティの対応が要るときに読む。

## 前提

- 本人の Notion コネクタで読み書きする。コネクタのツールは `ToolSearch` で `notion` を検索して読み込む（名前は `mcp__Notion__notion-fetch` や `mcp__notion__notion-fetch` のように表記が揺れる。見つかった方を使う）。主に使うもの: 検索（`notion-search`）、取得（`notion-fetch`）、DB の作成（`notion-create-database`）、データソースの変更（`notion-update-data-source`）、ページの作成（`notion-create-pages`）、ページの更新（`notion-update-page`）、DB の問い合わせ（`notion-query-data-sources`）、コメント（`notion-create-comment`）、自分の情報（`notion-get-self`）。各ツールの引数はツールの説明に従う。
- コネクタが無い、または未接続なら、何もせずに `{ "ok": false, "error": { "code": "unsupported", "message": "Notion コネクタが接続されていない" } }` を返す。
- 現在の版は `1`（`references/schema-v1.md`）。
- 認証情報を扱わない。返す JSON・ページ本文・ログに書かない。

## 操作

### `describe`

`{ "adapter": "notion", "schema_version": 1, "types": [<schema-v1.md の型と DB の対応の型すべて>], "ops": ["describe","inspect","plan","apply","clear_migration","query","get","create","update","link","comment","check_permission"], "permission_check": "Notion のページ権限は API で読めないため、check_permission は試験的な読み取りで判定し、書き込みは unknown を返す" }` を返す。`put_file`・`get_file` は扱わない（`unsupported`）。

### `inspect`

何も書かない。

1. `location` を判別する。`https://` で始まればページの URL、それ以外は DB／データソースの識別子として `notion-fetch` で取得する。取得できなければ `not_found`。
   - DB の識別子なら、その DB の親（プロジェクトのトップページ）を起点にする。
   - 起点がプロジェクトのトップページ（本文に `ACT 設定` がある）なら、その親を ACT のルートとする。それ以外は起点を ACT のルートとする。
2. **プロジェクト側**: ルートの子ページのうち、`ACT 設定` の `project_key` が `args.project_key` と一致するページを、このプロジェクトのトップページとする。一致するものが無ければ、タイトルが `ACT: <project_name>` で `ACT 設定` を持たないページを候補にする（取り込みの対象）。他の `project_key` のトップページとその DB は、プロジェクト側として扱わない。
   - `managed`: トップページに `ACT 設定` があれば `true`
   - `recorded_schema_version`: `ACT 設定` の `schema` の版。`schema` が空なら `null` で `partial: true`。DB の説明文の印の版はここに使わない
   - `store_ids`: トップページの子の DB のうち `[ACT]` 印のあるものを型ごとに
   - `missing_types`: `schema-v1.md` のプロジェクト DB のうち `store_ids` に無い型
3. **共通側**: 共通 DB（🏢・📁・🏷・🧠）を次の順で探す。
   1. ルートの子の DB で `[ACT]` 印の `type` が共通の型のもの。
   2. 見つからなければ `notion-search` で `[ACT] type=` を手がかりに同じワークスペースを探す。1 組だけ見つかればそれを使う。2 組以上（別の ACT ルート）なら、選ばずに `candidates` に並べる。
   - `recorded_schema_version`: 共通 DB の説明文の印の版（4 つで違えば最小）。無ければ `null`
4. 印の無い DB は、名前とプロパティを `schema-v1.md` と比べて `role` を推定し、`scope` を `unknown` として `databases` に載せる（取り込みの材料）。
5. `migration`: トップページの `ACT 設定` の `migrating`・`migrating_since`・`run_id`。共通 DB だけが移行中なら、説明文の 1 行目末尾の ` migrating <run_id> <日時>` から読む。
6. `workspace`・`user`: `notion-get-self` から。

### `plan`

`inspection`・`target_schema_version`・`mode` から変更を組み立てる。何も書かない。プロジェクト側と共通側は別々に考える。

- **プロジェクト側**
  - 新規作成（`project.managed` が偽）、または適用途中（`partial`）: トップページ（無ければ）、`ACT 設定`（無ければ）、`missing_types` の DB の作成を `追加` として並べる。最後に `ACT 設定` の `schema` を書く変更を入れる。
  - 取り込み: 既存の DB を `schema-v1.md` の DB に対応づける。`mapping` が渡されればそれを使い、無ければ `mapping_proposal` を作る（プロパティ名・型・値の傾向から推定し、各項目に `basis` を書く。人が足したプロパティは `ACT では使わない`）。「そのまま使う」「名前を変えて使う」項目はプロパティ名を変えない（読み替えは `mapping` で行う）。足りないプロパティ・DB の作成を `追加` とし、最後に `[ACT]` 印と `ACT 設定` を書く変更を入れる。
  - 更新: `recorded_schema_version` から `target_schema_version` まで 1 版ずつ、`references/schema-v<n>.md` の移行手順を並べる。渡された `mapping` で既存のプロパティ名に読み替える。
- **共通側**
  - `shared.store_ids` に揃っていれば作らない（つなぐだけ）。足りない共通 DB の作成と、版 1 の初期タグ（`種類:` 7 件と `汎用`）を `追加` にする。
  - `shared.candidates` があれば、`shared_choice` が渡されるまで共通側の変更を作らず、`changes` に `category: 手作業`・`description: 共通 DB の候補が複数ある。どれを使うか選んで再計画する` を 1 件だけ入れる。
  - 共通側の版が `target_schema_version` より古ければ、共通 DB の移行を 1 版ずつ並べる。
- 変更 ID は `<分類の頭文字>-<連番>`（例: `A-1`、`C-2`）。`変換` には `samples`（実際のページ 3 件程度の変換前後）と `affected_pages` を付ける。
- 削除は `changes` に入れない。不要なプロパティは `旧_` を付けて名前を変える `変換` にする。
- `manual_steps` に `schema-v1.md` の「手作業が必要な設定」のうち、この計画で作る DB に関わるものを入れる。

### `apply`

渡された `changes` だけを、渡された順に適用する。

1. 移行中の印を付ける。プロジェクトのトップページ（無ければ最初に作る）の `ACT 設定`（無ければ `schema` を空にして作る）に `migrating: true`、`migrating_since`、`run_id`、`project_key` を書く。共通 DB を変えるときは、その DB の説明文の 1 行目末尾に ` migrating <run_id> <日時>` を付ける。既に別の `run_id` の印があれば何もせず `conflict` を返す（同じ `run_id` なら続ける）。
2. 各変更を適用し、`results` に `success`／`failed`／`already_applied` を記録する。適用前に現状を読み、既に目的の状態なら書かずに `already_applied` とする（冪等）。
   - DB の作成: `schema-v1.md` のプロパティで作り、説明文の 1 行目に `[ACT] type=<型> schema=notion/<版>` を書く。ID のプロパティは接頭辞を付ける。
   - プロパティ名の変更と、同じ名前のプロパティの追加は、1 回の更新に混ぜない（元の値が失われるため）。変換は、旧名に `旧_` を付けて残す → 新しいプロパティを作る → 値を写す → 写した値を照合する、の順に別々の更新で行う。
   - タグの作成は 🏷 タグ に行を足す。既に同じ名前があれば `already_applied`。
3. 1 つでも `failed` があれば、そこで止める（後の変更は適用しない）。
4. 終わったら（止まったときも）、`ACT 設定` の `store_ids` を今ある DB で更新し、`migrating` を `false` に戻し、共通 DB の説明文の印を外す。全部成功したときだけ `schema` に `notion/<版>` を書く。`ACT 移行ログ` に 1 行追記する（日時、`run_id`、実行者、前後の版、適用した変更 ID、失敗）。
5. `{ "results", "schema_version", "store_ids" }` を返す。`store_ids` はプロジェクト側と共通側を合わせたもの（`project_registry` は 3 つの識別子の形）。

### `clear_migration`

`run_id` の印が残っている `ACT 設定` と共通 DB の説明文から印を外し、`ACT 移行ログ` に「印を外した（実行 `run_id` が途中で止まっていた）」を追記する。印の `run_id` が違えば何もせず `conflict` を返す。保存先のほかの内容は変えない。

### 対応づけ（`mapping`）

`query`・`get`・`create`・`update`・`link` は、`args.mapping`（正準項目 → 既存プロパティ名）があれば、その項目を既存プロパティ名で読み書きする。`mapping` に無い項目は `schema-v1.md` のプロパティ名を使う。

### `query`

`type` に対応する DB を `store_ids`（呼び出し元が `project.yaml` から渡す識別子、無ければ `inspect` で探す）で特定し、`notion-query-data-sources` で `filter` をプロパティの条件に変換して問い合わせる。名前・別名・キーワードはタイトルと `別名`・`適用範囲` の部分一致で探す。結果は `attributes` だけの正準エンティティの配列で返す（本文は読まない）。🏷 タグ の `非推奨` の行は、呼び出し元が求めない限り `merged_into` を付けて返す。

### `get`

`id`（ACT の ID か Notion のページ識別子）でページを取得し、プロパティを `attributes` に、本文の固定見出しごとの中身を `narrative` に入れて返す。テンプレートに無い見出しは `narrative.other` にまとめる。

### `create`

`type` の DB にページを作る。`attributes` をプロパティに、`narrative` を本文の固定見出しの下に書く（見出しは `schema-v1.md` の順）。リレーション先は ACT の ID からページを引いて結ぶ。ID のプロパティは Notion が採番するので、返す `id` は採番後の値。`{ "id", "url" }` を返す。

### `update`

`attributes` のプロパティを書き換え、`sections` を `mode` に従って書く（`replace` は見出しの下を置き換え、`append` は末尾に日付付きで足す）。テンプレートに無い見出し・段落は消さない。完了条件はチェック状態の変更だけを受け付け、条件の追加・変更が来たら `invalid_request` を返す。更新後のエンティティを返す。

### `link`

`relation` に当たるリレーションのプロパティに、`to` のページを足す。既にあれば何もしない。

### `comment`

対象ページにコメントを付ける。他者のナレッジへの気づきはこの操作で渡す（ページ本文は書き換えない）。

### `check_permission`

Notion の権限は API で読めない。読み取りは対象を `notion-fetch` で取得できるかで `true`／`false` を返す。書き込みは試しに書くことをせず `unknown` を返し、手がかり（メンバー DB の自分の役割、DB の説明文の権限の記載）を `reason` に書く。書き込みの可否は、実際の書き込みが成功するか `permission_denied` になるかで分かる。

## エラーの変換

| Notion の状況 | 正準エラー |
| --- | --- |
| 権限が無い・共有されていない（403、restricted） | `permission_denied` |
| ページ・DB が見つからない（404） | `not_found` |
| 移行中の印が別の `run_id` で付いている | `conflict` |
| 同時更新・レート制限で失敗（409・429） | `conflict` |
| `put_file`・`get_file`、未対応の型 | `unsupported` |
| 入力の形が合わない | `invalid_request` |
