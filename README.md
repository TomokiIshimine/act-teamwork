# act-teamwork

ACTモデル（Agentic, Context-driven Teamwork）を Claude 上で運用するためのプラグイン群。

## 収録プラグイン

| プラグイン | 概要 | バージョン |
| --- | --- | --- |
| `act-core` | コア。初期設定（`/act-core:setup`）、保存先アダプタの呼び出し規約、書き込み計画と確認フォームの規約 | 0.1.0 |
| `act-store-notion` | 保存先アダプタ（Notion）。決定事項・タスク・ナレッジなどを Notion の DB に置く | 0.1.0 |
| `act-store-localfs` | 保存先アダプタ（ローカルファイル）。アーティファクト本体を共有フォルダに置く | 0.1.0 |
