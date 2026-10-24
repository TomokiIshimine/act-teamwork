---
name: store
description: ACT の保存先アダプタ（Notion）。act-core のスキル・エージェントが act-core:store-protocol に従って保存先の種類 notion の操作（describe・inspect・plan・apply・query・get・create・update・link・comment・check_permission）を依頼したときに読み、本人の Notion コネクタで実行して正準モデルの JSON で返す。依頼例:「act-store-notion で inspect して」「Notion の保存先に task を create して」「Notion の ACT スキーマを最新にする計画を作って」「/act-store-notion:store」。
---

# store — Notion アダプタ

`act-core:store-protocol` の操作を Notion で実行する。呼び出し形式（`{ "op", "adapter", "args" }` → `{ "ok", "result" | "error" }`）、正準モデルの JSON、正準エラーはその規約に従う。ここでは Notion での実現だけを定める。呼び出し元のワークフローの事情（なぜその操作を呼ぶか）は扱わない。

Notion の構成（DB・プロパティ・本文テンプレート・ACT の印・正準モデルとの対応）は `references/schema-v3.md` に定める。`describe`・`inspect`・`plan`・`apply` を行うとき、または型とプロパティの対応が要るときに読む。版 2 の保存先を読むとき・版 2 から移すときは `references/schema-v2.md` も、版 1 の保存先を読むとき・版 1 から移すときはさらに `references/schema-v1.md` も読む（移行は 1 版ずつ）。

## 前提

- 本人の Notion コネクタで読み書きする。コネクタのツールは `ToolSearch` で `notion` を検索して読み込む（名前は `mcp__Notion__notion-fetch` や `mcp__notion__notion-fetch` のように表記が揺れる。見つかった方を使う）。主に使うもの: 検索（`notion-search`）、取得（`notion-fetch`）、DB の作成（`notion-create-database`）、データソースの変更（`notion-update-data-source`）、ページの作成（`notion-create-pages`）、ページの更新（`notion-update-page`）、DB の問い合わせ（`notion-query-data-sources`）、コメント（`notion-create-comment`）、自分の情報（`notion-get-self`）。各ツールの引数はツールの説明に従う。
- コネクタが無い、または未接続なら、何もせずに `{ "ok": false, "error": { "code": "unsupported", "message": "Notion コネクタが接続されていない" } }` を返す。
- 現在の版は `3`（`references/schema-v3.md`）。
- 以下の「ACT 設定」は、プロジェクトのトップページにある ACT の設定 JSON を指す（置き場所は `schema-v3.md`「ACT の印」）。版 2 以降では子ページ 🔧 ACT 設定、版 1 の保存先（移行の C-1 より前）では本文の見出し `ACT 設定`。両方あるときは `schema-v2.md`「ACT 設定の正」に従い、本文の `ACT 設定` を読む（C-1 が終わっていない）。
- 認証情報を扱わない。返す JSON・ページ本文・ログに書かない。

## 操作

### `describe`

`{ "adapter": "notion", "schema_version": 3, "types": [<schema-v3.md の型と DB の対応の型すべて（minutes を含む）>], "ops": ["describe","inspect","plan","apply","clear_migration","query","get","create","update","link","comment","check_permission"], "permission_check": "Notion のページ権限は API で読めないため、check_permission は試験的な読み取りで判定し、書き込みは unknown を返す" }` を返す。`put_file`・`get_file` は扱わない（`unsupported`）。

### `inspect`

何も書かない。

1. `location` を判別する。`https://` で始まればページの URL、それ以外は DB／データソースの識別子として `notion-fetch` で取得する。取得できなければ `not_found`。
   - DB の識別子なら、その DB の親（プロジェクトのトップページ）を起点にする。
   - 起点が 🔧 ACT 設定のページ → その親ページをトップページとする。
   - 起点が子ページ 🔧 ACT 設定（版 1 では本文の見出し `ACT 設定`）を持つページ → 起点をトップページとする。
   - 上の 2 つでは、トップページの親を ACT のルートとする。それ以外は起点を ACT のルートとする。
2. **プロジェクト側**: ルートの子ページのうち、ACT 設定（子ページは絵文字を除いた名前「ACT 設定」で照合。版 1 の保存先は本文の見出し `ACT 設定` も読む）を持ち、その JSON の `project_key` が `args.project_key` と一致するページを、このプロジェクトのトップページとする。子ページと本文の両方に ACT 設定があれば、`## 前提` のとおり本文を読む。一致するものが無ければ、タイトルが `ACT: <project_name>` で ACT 設定（子ページ・本文の見出しとも）を持たないページを候補にする（取り込みの対象）。他の `project_key` のトップページとその DB は、プロジェクト側として扱わない。
   - `managed`: トップページに ACT 設定があれば `true`
   - `recorded_schema_version`: ACT 設定の `schema` の版。`schema` が空なら `null` で `partial: true`。DB の説明文の印の版はここに使わない（例: C-1 の前で止まった版 1 の保存先は、本文の `schema` が `notion/1` なので `1`・`partial: false` となり、更新として続きから計画できる）
   - `store_ids`: トップページの子の DB のうち `[ACT]` 印のあるものを型ごとに。退役の印（`[ACT-retired]`）の DB は入れない
   - `missing_types`: `schema-v3.md` のプロジェクト DB のうち `store_ids` に無い型
3. **共通側**: 共通 DB（🏢・📁・🏷・🧠）を次の順で探す。
   1. ルートの子の DB で `[ACT]` 印の `type` が共通の型のもの。
   2. 見つからなければ `notion-search` で `[ACT] type=` を手がかりに同じワークスペースを探す。1 組だけ見つかればそれを使う。2 組以上（別の ACT ルート）なら、選ばずに `candidates` に並べる。
   3. 印のある共通 DB が 1 つも無ければ、印の無い既存の共通 DB を探し、`adoptable` に載せる（取り込みの候補。`store_ids` には入れない）。探す範囲は、ルートの子の DB と、`notion-search` で共通 DB の名前（絵文字を除いた「クライアント」「プロジェクト」「タグ」「ナレッジ」）を検索して見つかった DB。見つかった DB を親ページごとにまとめ、各 DB を下の「共通 DB の照合」で照合する。共通の型が 1 つ以上「一致」か「近い」になった親ページを 1 組の候補とする。
   - `recorded_schema_version`: 共通 DB の説明文の印の版（4 つで違えば最小）。無ければ `null`
   - `adoptable`: 候補の配列。各要素は `location`（親ページの URL と名前）、`store_ids`（型ごと。「一致」「近い」の DB だけ）、`match`（型ごとに `result`（`一致`／`近い`／`不一致`）、`missing_properties`（足りないプロパティ）、`differing_properties`（名前は同じで型が違うもの）、`renamed_properties`（名前は違うが対応づけられるもの: 既存名 → 正準項目）、`id_prefix`（下記 7 で読んだ接頭辞）、`basis`（判断の根拠を 1 文））。ルートの子の候補を先頭にし、「一致」の多い順に並べる。印のある共通 DB が見つかったときは空。
4. 印の無い DB は、名前とプロパティを `schema-v3.md` と比べて `role` を推定し、`scope` を `unknown` として `databases` に載せる（取り込みの材料）。共通の型と推定した DB も載せる（`adoptable` の DB と同じ `id`）。退役の印（`[ACT-retired]`）の DB は載せない。
5. `migration`: ACT 設定の `migrating`・`migrating_since`・`run_id`。共通 DB だけが移行中なら、説明文の 1 行目末尾の ` migrating <run_id> <日時>`（取り込み途中の印の無い DB では先頭の ` migrating` だけの行）から読む。
6. `workspace`・`user`: `notion-get-self` から。
7. `id_prefixes`: ワークスペースで使われている ID の接頭辞。Notion の ID（一意 ID）の接頭辞はワークスペース内で一意でなければならず、使用済みの接頭辞で DB を作ると Notion が拒否する（`apply` が途中で止まる）ので、`plan` で衝突を見つけるために集める。
   - 対象の DB: この `inspect` で見つけた DB すべて（ルートの子、プロジェクトのトップページの子、`[ACT]` の検索結果、`adoptable` の材料）と、`notion-search` で `schema-v3.md` の DB 名（絵文字を除く）を検索して見つかった DB。
   - 各 DB の ID のプロパティ（一意 ID）の接頭辞を次の順で読む。コネクタは DB のスキーマに接頭辞を出さないことがあるため、出ていなければ行の値から読む。
     1. `notion-fetch` で取れる DB のスキーマに接頭辞が出ていれば、それを使う（`source: schema`）。
     2. 出ていなければ、`notion-query-data-sources` で行を 3 件まで取り、ID の値（例 `CLI-12`）の末尾の `-<数字>` を除いた部分を接頭辞とする（`source: row`）。値が数字だけなら接頭辞なし（`prefix: null`）。3 件で接頭辞が食い違えば、多い方を使い `basis` に書く。
     3. 行が無い DB は接頭辞を読めない（`prefix: null`、`source: unknown`）。
   - 各要素は `prefix`、`database`（DB の識別子）、`name`、`url`、`marked`（`[ACT]` 印があるか）、`type`（印の型、または推定した型。分からなければ `null`）、`source`（`schema`／`row`／`unknown`）。
   - 検索で見つからない DB（共有されていない、ゴミ箱にある、名前が違う）は照合できない。`id_prefix_check` に `{ "checked": <照合した DB の数>, "unknown": <接頭辞を読めなかった DB の数>, "complete": false }` を返す（Notion では常に `complete: false`）。

#### 共通 DB の照合

印の無い DB を、`schema-v3.md` の共通 DB の定義と比べて `result` を決める。

- **一致**: DB 名（絵文字と空白を除く）が定義の DB 名と同じか、タイトルのプロパティ名が定義と同じで、定義のプロパティがすべて同じ名前・同じ型である（人が足したプロパティがあってもよい）。
- **近い**: 名前かタイトルのプロパティ名が合い、定義のプロパティの半分以上が名前（または `renamed_properties` で対応づけられる名前）と型で合う。足りないプロパティは取り込みで足せる。
- **不一致**: それ以外。`adoptable` には載せない。
- 型が違うプロパティ（`differing_properties`）は、足しても直せない（同じ名前で作れない）ので、1 つでもあれば最大で「近い」にする。
- ID のプロパティの接頭辞が定義（`CLI`／`TAG`／`KNW`）と違うか無いときは、照合の結果は変えず、`basis` にその旨を書く（接頭辞は変えない。取り込み後も既存の ID の値をそのまま使う）。

### `plan`

`inspection`・`target_schema_version`・`mode` から変更を組み立てる。何も書かない。プロジェクト側と共通側は別々に考える。

- **プロジェクト側**
  - 新規作成（`project.managed` が偽）、または適用途中（`partial`）: トップページ（無ければ。本文は `args.project_profile` から `schema-v3.md` の「トップページの雛形」（📌 プロジェクト概要・📍 現在の状況）で作る）、🔧 ACT 設定（無ければ）、`missing_types` の DB の作成を `追加` として並べる。最後に 🔧 ACT 設定の `schema` を書く変更を入れる。
  - 取り込み: 既存の DB を `schema-v3.md` の DB に対応づける。`mapping` が渡されればそれを使い、無ければ `mapping_proposal` を作る（プロパティ名・型・値の傾向から推定し、各項目に `basis` を書く。人が足したプロパティは `ACT では使わない`）。「そのまま使う」「名前を変えて使う」項目はプロパティ名を変えない（読み替えは `mapping` で行う）。足りないプロパティ・DB の作成を `追加` とし、最後に `[ACT]` 印と 🔧 ACT 設定を書く変更を入れる。
  - 更新: `recorded_schema_version` から `target_schema_version` まで 1 版ずつ、`references/schema-v<n>.md` の移行手順を並べる。渡された `mapping` で既存のプロパティ名に読み替える。
- **共通側**
  - `shared.store_ids` に揃っていれば作らない（つなぐだけ）。足りない共通 DB の作成と、版 1 の初期タグ（`種類:` 7 件と `汎用`）を `追加` にする。
  - `shared.candidates` があれば、`shared_choice` が渡されるまで共通側の変更を作らず、`changes` に `category: 手作業`・`description: 共通 DB の候補が複数ある。どれを使うか選んで再計画する` を 1 件だけ入れる。
  - `shared.store_ids` が空で `shared.adoptable` があるときは、`shared_choice` で次のように決める。
    - `shared_choice` が無い: 候補が 1 組なら、その組を取り込む計画を作る（下記）。2 組以上なら、共通側の変更を作らず、`changes` に `category: 手作業`・`description: 印の無い共通 DB の候補が複数ある。取り込む候補か新規作成を選んで再計画する` を 1 件だけ入れる。
    - `shared_choice` が候補の `location`: その組を取り込む計画を作る。
    - `shared_choice` が `new`: 取り込まずに新規作成する（上の 1 つ目の規則）。この場合も ID の接頭辞の衝突は下記で調べる。
  - **取り込み（共通側）**: 候補の組の「一致」「近い」の DB を、型ごとに次の変更にする。どれも `adopt: { "type": <型>, "database": <既存の DB の識別子> }` を付ける（呼び出し元はこれで取り込みの変更と分かり、確認が要る項目にする）。
    - 足りないプロパティ（`missing_properties`）の作成を `追加` にする。`renamed_properties` は名前を変えずに `mapping_proposal` に載せる（プロジェクト側の取り込みと同じ扱い。`type` に共通の型を書く）。
    - 型が違うプロパティ（`differing_properties`）は変えず、`手作業` にして `description` に食い違いを書く（その DB を ACT で使うと、そのプロパティは読み書きできない）。
    - 最後に、説明文の 1 行目に `[ACT] type=<型> schema=notion/<版>` を書く変更を `追加` にする。既存の説明文は 2 行目以降に残す。
    - 候補の組に無い共通の型は新規作成にする（下記の接頭辞の照合を通す）。
    - 版 1 の初期タグは、🏷 タグ を取り込むときは既存の行を名前と別名で探し、無いものだけを `追加` にする。
- **ID の接頭辞の照合**: DB を作る変更（共通側・プロジェクト側とも）ごとに、作る ID のプロパティの接頭辞（`schema-v3.md` の型と DB の対応。プロジェクト側は `<KEY>` を `project_key` に置き換える）を `inspection.id_prefixes` と照らす。同じ `prefix` を持つ DB があれば、その DB の作成を `changes` に入れず、`prefix_conflicts` に `{ "prefix", "type", "used_by": { "database", "name", "url", "marked", "type" }, "resolution" }` を 1 件足す。`resolution` は次のどれか。
  - `adopt`: 使っている DB が、同じ型の取り込みの候補（`adoptable`）にある。`shared_choice: new` をやめて取り込めば衝突しない。
  - `change_key`: プロジェクト側の型（`<KEY>-*`）。プロジェクトキーを変えれば衝突しない。
  - `manual`: それ以外。使っている DB の ID の接頭辞を Notion で変えるか、その DB を消す（ゴミ箱も空にする）まで作れない。
  - 衝突としないもの: 接頭辞を使っている DB が、作ろうとする型と同じ `type` の `[ACT]` 印を持ち、`inspection.project.store_ids`（共通側は `shared.store_ids`）にある（このプロジェクトのトップページの下にある自分の DB。例: 版 2 → 3 の適用が A-1 の後で止まり、作り直した計画に A-1 がまた並ぶとき）。この作成の変更は `changes` に残し、`apply` で `already_applied` になる。
  - 衝突した DB の作成だけを外し、ほかの変更は通常どおり返す。計画を止めるかどうかは呼び出し元が決める。
  - `id_prefix_check.unknown` が 1 以上なら、`prefix_unchecked` に接頭辞を読めなかった DB（名前と URL）を並べる（行が無い DB と同じ接頭辞は、作るときに初めて衝突が分かる）。
- 共通側の版が `target_schema_version` より古ければ、共通 DB の移行を 1 版ずつ並べる。
- 変更 ID は `<分類の頭文字>-<連番>`（例: `A-1`、`C-2`）。`変換` には `samples`（実際のページ 3 件程度の変換前後）と `affected_pages` を付け、行ごとに写す変換では対象の全件を `targets` に付ける（形は `act-core:store-protocol` の `plan` の出力に従う）。
- 途中で止まった保存先の再計画では、行ごとに写す変換の `targets` に写し済みの行も残し、`note` を `写し済み（飛ばす）` にする（`schema-v3.md` の C-1 と同じ文字列）（版 2 → 3 の C-1 では、元の 📄 の行を関連アーティファクトに持つ 📝 がある行）。
- 削除は `changes` に入れない。不要なプロパティは `旧_` を付けて名前を変える `変換` にする。
- `manual_steps` に `schema-v3.md` の「手作業が必要な設定」のうち、この計画で作る DB・トップページに関わるものを入れる。

### `apply`

渡された `changes` だけを、渡された順に適用する。

1. 移行中の印を付ける。置き場所は保存先の版で分ける（`schema-v3.md`「ACT の印」）。
   - トップページの本文に `ACT 設定` がある（版 1 の保存先で C-1 の前）: 子ページは作らず、本文の `ACT 設定` に `migrating: true`、`migrating_since`、`run_id` を書く。子ページ 🔧 ACT 設定は C-1 で作る。
   - それ以外: プロジェクトのトップページ（無ければ雛形の本文で最初に作る）の子ページ 🔧 ACT 設定（無ければ `schema` を空にして作る。トップページの本文の後に置く）に `migrating: true`、`migrating_since`、`run_id`、`project_key` を書く。

   共通 DB を変えるときは、その DB の説明文の 1 行目末尾に ` migrating <run_id> <日時>` を付ける（取り込む DB でまだ `[ACT]` 印が無いものは、` migrating <run_id> <日時>` だけの行を説明文の先頭に足す）。ACT 設定・共通 DB のどちらかに既に別の `run_id` の印があれば何もせず `conflict` を返す（同じ `run_id` なら続ける）。
2. 各変更を適用し、`results` に `success`／`failed`／`already_applied` を記録する。適用前に現状を読み、既に目的の状態なら書かずに `already_applied` とする（冪等）。
   - DB の作成: 同じ型の `[ACT]` 印を持つ DB がプロジェクトのトップページの下（共通側は共通 DB の置き場所）に既にあれば、作らず、接頭辞も照らさずに `already_applied` とする（前回の適用で作った分）。無ければ `schema-v3.md` のプロパティで作り、説明文の 1 行目に `[ACT] type=<型> schema=notion/<版>` を書く。ID のプロパティは接頭辞を付ける。作る直前に、その接頭辞を `inspect` の 7 と同じ方法で（このとき見つかる DB について）もう一度照らし、使用済みなら作らずに `failed` とし、`conflict` を返す（`message` は `ID の接頭辞 <接頭辞> は <DB 名>（<URL>）で使用済み`）。Notion が接頭辞の使用済みで作成を拒否したときも同じく `failed`・`conflict` とし、`message` に接頭辞と「使用元は検索で見つからなかった（共有されていない DB かゴミ箱の DB の可能性）」を書く。どちらも再実行では解消しない。
   - 共通 DB の取り込み（`adopt` の付いた変更）: 既存の DB の識別子に対して適用する。印を書く変更は、説明文の 1 行目に `[ACT] type=<型> schema=notion/<版>` を入れ、既存の説明文は 2 行目以降に残す（取り込みの途中で足した ` migrating` の行はこのとき外す）。既にその印があれば `already_applied`。
   - プロパティ名の変更と、同じ名前のプロパティの追加は、1 回の更新に混ぜない（元の値が失われるため）。変換は、旧名に `旧_` を付けて残す → 新しいプロパティを作る → 値を写す → 写した値を照合する、の順に別々の更新で行う。
   - タグの作成は 🏷 タグ に行を足す。既に同じ名前があれば `already_applied`。
   - 版 1 → 2 の C-1: `schema-v2.md` の移行手順のとおり、移行中の印ごと子ページ 🔧 ACT 設定に写してから本文の `ACT 設定` を外す。以後の印と 4. の書き込みは子ページに行う。
   - 版 2 → 3 の C-1〜C-3: `schema-v3.md` の移行手順と「冪等・再開」のとおり。C-1 は、照合キー（📝 の関連アーティファクトにある元の 📄 の行）で写し済みと分かった行を書かずに飛ばす。
   - 行ごとに写す変換（版 2 → 3 の C-1〜C-3 など）も、`results` には変更 ID ごとに 1 件だけ記録する。対象の全行が写し済み（飛ばした）なら `already_applied`、1 行でも失敗すれば `failed`（失敗した行の ID と理由をエラーの `message` に書く）、それ以外は `success`。
3. 1 つでも `failed` があれば、そこで止める（後の変更は適用しない）。
4. 終わったら（止まったときも）、ACT 設定の `store_ids` を今ある DB で更新し、`migrating` を `false` に戻し、共通 DB の説明文の印を外す。全部成功したときだけ `schema` に `notion/<版>` を書く。
5. `{ "results", "schema_version_before", "schema_version", "store_ids" }` を返す。`schema_version_before` は適用前の版（プロジェクト側。新規作成なら `null`）。`store_ids` はプロジェクト側と共通側を合わせたもの（`project_registry` は 3 つの識別子の形）。

### `clear_migration`

`run_id` の印が残っている ACT 設定と共通 DB の説明文から印を外し（先頭の ` migrating` だけの行は行ごと消す）、`{ "cleared": true, "run_id" }` を返す。印の `run_id` が違えば何もせず `conflict` を返す。保存先のほかの内容は変えない。

### 対応づけ（`mapping`）

`query`・`get`・`create`・`update`・`link` は、`args.mapping`（正準項目 → 既存プロパティ名）があれば、その項目を既存プロパティ名で読み書きする。`mapping` に無い項目は `schema-v3.md` のプロパティ名を使う。

### `query`

`type` に対応する DB を `store_ids`（呼び出し元が `project.yaml` から渡す識別子、無ければ `inspect` で探す）で特定し、`notion-query-data-sources` で `filter` をプロパティの条件に変換して問い合わせる。名前・別名・キーワードはタイトルと `別名`・`適用範囲` の部分一致で探す。結果は `attributes` だけの正準エンティティの配列で返す（本文は読まない）。🏷 タグ の `非推奨` の行は、呼び出し元が求めない限り `merged_into` を付けて返す。

### `get`

`id`（ACT の ID か Notion のページ識別子）でページを取得し、プロパティを `attributes` に、本文の固定見出しごとの中身を `narrative` に入れて返す。テンプレートに無い見出しは `narrative.other` にまとめる。

型が `run_log` のときは ID のプロパティが無いので、`id`（run_id）を 🧾 実行ログのタイトル（実行）と照合して行を特定する（`notion-query-data-sources` でタイトルが一致する行）。無ければ `not_found`。

### `create`

`type` の DB にページを作る。`attributes` をプロパティに、`narrative` を本文の固定見出しの下に書く（見出しは `schema-v3.md` の順）。リレーション先は ACT の ID からページを引いて結ぶ。ID のプロパティは Notion が採番するので、返す `id` は採番後の値。型が `run_log` のときは、タイトルを `id`（run_id）にする（ID のプロパティは無い）。`{ "id", "url" }` を返す。

型が `daily_summary` で、その日付がその DB の中で最新なら、トップページの 📍 現在の状況を、作ったページの本文の先頭セクション（一言で）とそのページへのリンクで置き換える。

### `update`

`attributes` のプロパティを書き換え、`sections` を `mode` に従って書く（`replace` は見出しの下を置き換え、`append` は末尾に日付付きで足す）。テンプレートに無い見出し・段落は消さない。完了条件はチェック状態の変更だけを受け付け、条件の追加・変更が来たら `invalid_request` を返す。型が `minutes` のときは、`act-core:store-protocol` の本文の更新ルールに合わない更新を `invalid_request` で返す（見出しの対応は `schema-v3.md`「議事録の本文の更新」）。型が `run_log` のときは、`get` と同じく `id`（run_id）をタイトルと照合して行を特定し（無ければ `not_found`）、その 1 件を丸ごと置き換える更新だけを受け付ける: `attributes` に無い属性は空にし、`sections` はすべて `replace` で書く。`mode: append` と、`attributes` の `id` を別の値に変える更新は `invalid_request` を返す。更新後のエンティティを返す。

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
| ID の接頭辞が使用済みで DB を作れない | `conflict`（`message` に接頭辞と使用元。時間をおいても解消しないことを書く） |
| 同時更新・レート制限で失敗（409・429） | `conflict` |
| `put_file`・`get_file`、未対応の型 | `unsupported` |
| 入力の形が合わない | `invalid_request` |
