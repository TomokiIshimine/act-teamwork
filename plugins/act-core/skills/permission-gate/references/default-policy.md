# 既定ポリシー

権限ゲートの既定の表。役割は `business_owner`（ビジネスオーナー）・`manager`（マネージャー）・`worker`（ワーカー）。読み方は `SKILL.md` の「判定の手順」。

- 「許す役割」に入る役割は「許すとき」の書き方、入らない役割は「許さないとき」の書き方になる。
- `<root>/.act/project.yaml` の `policy.<キー>: [役割…]` で、その行の「許す役割」を置き換えられる。「上書き」が `不可` の行は置き換えない。
- 「確認つき」の行は、結果の `confirm` を `true` にする。

## 決定事項（`decision`）

| キー | 対象・操作 | 許す役割 | 許すとき | 許さないとき | 上書き | 備考 |
| --- | --- | --- | --- | --- | --- | --- |
| `decision_confirm_business` | 決定事項（`layer: business`）を確定 | business_owner | 確定で書く | 提案として書く → `decision_propose` で判定し直す | 可 | |
| `decision_confirm_management` | 決定事項（`layer: management`）を確定 | business_owner, manager | 確定で書く | 提案として書く → `decision_propose` で判定し直す | 可 | |
| `decision_propose` | 決定事項を提案 | business_owner, manager | 提案として書く | 判断依頼にする（決定の案を書く） | 可 | 宛先は層で決める: `business` → `business_owner`、`management` → `manager`。ワーカーは決定事項の保存先を編集できない前提のため判断依頼にする |

入力から読み取っただけの決定は、確定の行で許されても `提案として書く` を上限にする（`SKILL.md`「入力から読み取った決定・状態」）。

## タスク（`task`）

| キー | 対象・操作 | 許す役割 | 許すとき | 許さないとき | 上書き | 備考 |
| --- | --- | --- | --- | --- | --- | --- |
| `task_define` | 定義済みタスク（`kind: defined`）の作成、対象範囲・完了条件・期限の変更 | business_owner, manager | 確定で書く | 判断依頼にする（宛先 `manager`） | 可 | |
| `task_auxiliary_add` | 補助タスク（`kind: auxiliary`）の追加 | business_owner, manager | 確定で書く | 判断依頼にする（宛先 `manager`） | 可 | ワーカーは本来「影響チェックを通れば確定」。影響チェックはこの規約にまだ無いため、判定できないものとして判断依頼に倒す |
| `task_status_own` | タスクの状態・進捗メモの更新（自分の担当） | business_owner, manager, worker | 確定で書く | 書かない（コメントのみ） | 可 | 条件: `assignee` が操作者。ワーカーが `done` にするときは `adjusted: { status: in_review }`（レビュー待ちまで） |

自分の担当でないタスクの状態の変更は表に無い（`未定義`）。

## ナレッジ（`knowledge`）

| キー | 対象・操作 | 許す役割 | 許すとき | 許さないとき | 上書き | 備考 |
| --- | --- | --- | --- | --- | --- | --- |
| `knowledge_own` | 自分のナレッジの作成・更新 | business_owner, manager, worker | 確定で書く | 書かない（コメントのみ） | 可 | 条件: 所有者が操作者 |
| `knowledge_other_update` | 他者のナレッジの更新 | （なし） | — | 書かない（コメントのみ） | 不可 | ナレッジは本人の理解。他者は内容を書き換えない |
| `knowledge_widen` | 自分のナレッジを `汎用` に変える、閲覧の制限を外す | business_owner, manager, worker | 確定で書く（確認つき） | 書かない（コメントのみ） | 可 | 条件: 所有者が操作者 |
| `knowledge_restrict_other` | 他者のナレッジの閲覧を絞る（漏れの是正） | business_owner, manager | 確定で書く | 書かない（コメントのみ） | 可 | `manager` は、自分がマネージャーであるプロジェクトのタグが付いたものだけ |

## タグ（`tag`）

| キー | 対象・操作 | 許す役割 | 許すとき | 許さないとき | 上書き | 備考 |
| --- | --- | --- | --- | --- | --- | --- |
| `tag_attach` | タグを付ける（一覧から選ぶ） | business_owner, manager, worker | 確定で書く | 書かない（コメントのみ） | 可 | |
| `tag_create` | タグの作成（軸 `話題:` だけ、作成ルールに沿う）・別名の追加 | business_owner, manager, worker | 確定で書く | 書かない（コメントのみ） | 可 | `話題:` 以外の軸のタグの作成は表に無い（`未定義`） |
| `tag_rename` | タグの改名・非推奨（統合）・軸の変更 | business_owner, manager | 確定で書く | 書かない（コメントで提案） | 可 | |
| `tag_delete` | タグの削除 | （なし） | — | 書かない（コメントのみ） | 不可 | 使われなくなったタグは非推奨にする |

## アーティファクト（`artifact_meta`、本体）

| キー | 対象・操作 | 許す役割 | 許すとき | 許さないとき | 上書き | 備考 |
| --- | --- | --- | --- | --- | --- | --- |
| `artifact_register` | アーティファクトの登録・新しい版 | business_owner, manager, worker | 確定で書く | 書かない（コメントのみ） | 可 | |
| `artifact_approve` | アーティファクトを「承認済み」にする | business_owner, manager | 確定で書く | 判断依頼にする（宛先 `manager`） | 可 | `policy.artifact_approve` で置き換える |

## 置き場所（アーティファクト本体の置き場所と索引）

| キー | 対象・操作 | 許す役割 | 許すとき | 許さないとき | 上書き | 備考 |
| --- | --- | --- | --- | --- | --- | --- |
| `placement_structure` | 置き場所の構成の設定・変更（`project.yaml` の `stores.artifact_body` の `folder_root`・`layout`） | business_owner, manager | 確定で書く | 判断依頼にする（構成案を書く。宛先 `manager`） | 可 | |
| `placement_folder_add` | 既存の構成への置き場所の追加 | business_owner, manager, worker | 確定で書く | 書かない（コメントのみ） | 可 | |
| `placement_index_update` | 置き場所の索引の更新 | business_owner, manager, worker | 確定で書く | 書かない（コメントのみ） | 可 | |
| `placement_folder_modify` | 置き場所の改名・移動・削除 | （なし） | — | 書かない（プラグインでは行わない。人の作業として案内する） | 不可 | |

## 判断依頼（`task` の `kind: request`）

| キー | 対象・操作 | 許す役割 | 許すとき | 許さないとき | 上書き | 備考 |
| --- | --- | --- | --- | --- | --- | --- |
| `request_create` | 判断依頼の起票 | business_owner, manager, worker | 確定で書く | 書かない（コメントのみ） | 可 | |
| `request_answer` | 判断依頼への回答（`request_result` の記入） | business_owner, manager | 確定で書く | 書かない（コメントのみ） | 可 | `manager` は `to_role: manager` の依頼だけ |

## ワークフロー設定（`workflow_config`）

| キー | 対象・操作 | 許す役割 | 許すとき | 許さないとき | 上書き | 備考 |
| --- | --- | --- | --- | --- | --- | --- |
| `config_dictionary_append` | 辞書への追記 | business_owner, manager, worker | 確定で書く | 書かない（コメントのみ） | 可 | |
| `config_rule_change` | ルール・書式の変更、有効／無効の切り替え | business_owner, manager | 確定で書く | 書かない（コメントで提案） | 可 | |
