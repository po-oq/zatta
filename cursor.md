# Cursor IDE チャット管理ダッシュボード — 調査ブリーフ

作成日: 2026-08-31
用途: 社内PC（Cursor インストール環境）での実地調査のための引き継ぎ資料

---

## 0. ゴール

Cursor IDE のチャット（Composer / Agent）セッションを一覧・監視するデスクトップダッシュボードを作る。
既存の Claude Code 向け同種アプリと同等の情報粒度を目指す。

**主目的は「セッションの状態が今どうなっているか」の可視化。**

想定パネル:
- セッション一覧（状態 / 経過 / ターン数 / 入力）
- ターン詳細（種別 / 内容 / 経過 / Thinking）
- 履歴（完了したプロセス）
- 利用状況の推移

---

## 1. 前提・制約

| 項目 | 内容 |
|---|---|
| 対象 | **Cursor IDE のみ**。CLI（`cursor-agent`）は社内申請で未許可のため対象外 |
| DB | **globalStorage の SQLite をコピーしない**（GB級のため）。原則、直接読みも避ける |
| 実装 | C# / .NET（デスクトップアプリ） |
| 追加要件 | アプリ導入**前**に発生したセッションを取り込めるかを判断したい |
| 安全性 | Cursor 本体の動作を絶対に阻害しないこと（業務停止は許容不可） |

---

## 2. Cursor のデータ保存構造（調査済み）

### 2.1 SQLite（今回は原則触らない層）

Windows のベースパスは `%APPDATA%\Cursor\User\`。

```
%APPDATA%\Cursor\User\
├── globalStorage\state.vscdb     # 会話の実体（全プロジェクト共通・巨大）
└── workspaceStorage\<hash>\
    ├── workspace.json            # hash → 実プロジェクトパス
    └── state.vscdb               # 選択中タブ等の小物
```

両DBともテーブルは2つだけ。値は BLOB だが実体は UTF-8 の JSON 文字列。

```sql
CREATE TABLE ItemTable   (key TEXT UNIQUE, value BLOB);
CREATE TABLE cursorDiskKV(key TEXT UNIQUE, value BLOB);
```

**主要キー（globalStorage / cursorDiskKV）**

| キー | 内容 |
|---|---|
| `composerData:{composerId}` | 会話メタ。`fullConversationHeadersOnly` が順序、`unifiedMode` / `modelConfig` / `createdAt` / `status` |
| `bubbleId:{composerId}:{bubbleId}` | 1メッセージ1行。`text` / `type`(1=user,2=assistant) / `codeBlocks` / `toolResults` / `allThinkingBlocks` / `createdAt` |
| `checkpointId:{composerId}:{checkpointId}` | ターンごとのファイル差分スナップショット |
| `messageRequestContext:{composerId}:{messageId}` | モデルに送った完全なコンテキスト |
| `composer.content.{hash}` | 大きいテキストの content-addressed キャッシュ |

**セッション一覧のインデックス（バージョンで分岐）**

- Cursor 3.0+（2026-04-02リリース・一方向の破壊的マイグレーション）: globalStorage の `ItemTable` の `composer.composerHeaders` に集約。各エントリに `workspaceIdentifier`（`id` = workspaceStorage のハッシュ、`uri.fsPath` = 実パス）
- Cursor 2.x 以前: workspaceStorage 側 `ItemTable` の `composer.composerData` 内 `allComposers[]`
- 判定: workspace 側に `allComposers` が無ければ移行済み
- **注意**: `composer.composerHeaders` はマイグレーション後に開いた会話しか載らない

**WAL 注意**: 両DBとも WAL モード。`.vscdb` 本体だけ読むと最新が反映されていない。整合スナップショットが要る場合は `.vscdb` / `-wal` / `-shm` の3点セットが必要（＝今回コピーしない方針の根拠）。

### 2.2 agent-transcripts（過去セッション取り込みの本命）

```
%USERPROFILE%\.cursor\projects\<フォルダ>\agent-transcripts\<ID>\<ID>.jsonl
```

- Cursor が書き出す読み取り専用ログ。Cursor 自身はここから会話をロードしない（SQLite の補助）
- フォルダ名はプロジェクトパスの区切り文字を `-` に置換したもの
- **JSONL であることを実機確認済み**（＝構造化データ。テキストログより再現度が高い）

**`~/.cursor/projects` 配下で観測されたフォルダ3種**

| 種類 | 推定 | 扱い |
|---|---|---|
| 13桁の数値 | ワークスペース非紐付けの会話。Cursor 3.0 でタイムスタンプ形式の識別子が振られる仕様と一致（13桁＝epoch ms） | 一覧には出すが「プロジェクトなし」扱い |
| `〜-AppData-Local-Temp-<hash>` | Temp配下で開いた一時ワークスペース | 除外候補 |
| プロジェクト名 | 本命。`agent-transcripts` を持つ | 対象 |

`canvases` / `terminals` / `mcps` サブフォルダは本用途では無視。

---

## 3. Hooks（リアルタイム層の主役）

公式機能。`hooks.json` に登録したコマンドが stdio で JSON をやり取りする。
**CLI の実行は不要**（IDE の Agent ループに対する機能）。

### 3.1 設定場所と優先度

優先度（高→低）: Enterprise → Team → Project → User

- User: `~/.cursor/hooks.json`（スクリプトの相対パス基準は `~/.cursor/`）
- Project: `<project-root>/.cursor/hooks.json`（基準はプロジェクトルート）
- Enterprise(Windows): `C:\ProgramData\Cursor\hooks.json`

Cursor は hooks.json を監視して自動リロードする。**Customize 内に Hooks タブと出力チャネル**があり、設定済み/実行済みフックとエラーを確認できる。

### 3.2 全フック共通の入力（stdin JSON）

```json
{
  "conversation_id": "string",   // 多数のターンをまたいで安定した会話ID
  "generation_id": "string",     // ユーザーメッセージごとに変わる
  "model": "string",
  "model_id": "string",
  "model_params": [{ "id": "thinking", "value": "true" }],
  "hook_event_name": "string",
  "cursor_version": "string",
  "workspace_roots": ["<path>"],
  "user_email": "string | null",
  "transcript_path": "string | null"  // トランスクリプト無効なら null
}
```

`conversation_id` をセッションキーにできる。`workspace_roots` があるので workspace.json のハッシュ解決は不要。

### 3.3 使うフックと取得できる情報

| フック | 取れるもの |
|---|---|
| `sessionStart` | `session_id`（`conversation_id` と同一と明記）/ `composer_mode`(agent,ask,edit) / `is_background_agent` |
| `beforeSubmitPrompt` | `prompt`（ユーザー入力本文）/ `attachments` |
| `preToolUse` | `tool_name` / `tool_input` / `tool_use_id` / `cwd` / `agent_message` |
| `postToolUse` | 上記 + `tool_output` / `duration`(ms) |
| `postToolUseFailure` | `error_message` / `failure_type`(error,timeout,permission_denied) / `is_interrupt` |
| `afterAgentThought` | 思考テキスト / `duration_ms` |
| `afterAgentResponse` | アシスタント最終テキスト |
| `afterFileEdit` | `file_path` / `edits[]`(old_string,new_string) |
| `beforeShellExecution` / `afterShellExecution` | `command` / `output` / `duration` / `sandbox` |
| `preCompact` | `context_usage_percent` / `context_tokens` / `context_window_size` / `message_count` / `messages_to_compact` / `trigger` |
| `stop` | `status`(completed,aborted,error) / `loop_count` |
| `sessionEnd` | `reason`(completed,aborted,error,window_close,user_close) / `duration_ms` / `final_status` |
| `subagentStart` / `subagentStop` | `subagent_type` / `task` / `message_count` / `tool_call_count` / `modified_files` / `duration_ms` |

環境変数として `CURSOR_PROJECT_DIR` / `CURSOR_VERSION` / `CURSOR_TRANSCRIPT_PATH` なども渡る。

### 3.4 Hooks の限界（重要）

- **トークン消費数は取得できない**。フックのペイロードに token 数のフィールドは無い（`duration` はある）
- **コンテキスト使用率は `preCompact` 時のみ**。常時ゲージにはならない → 「直近の圧縮時点」表記にする
- **遡って発火しない**。導入前セッションはフックでは一切取れない
- Tab（インライン補完）は別系統（`beforeTabFileRead` / `afterTabFileEdit`）。ノイズになるなら登録しない

### 3.5 実装上の必須ルール

- **`failClosed: true` を絶対に付けない**。デフォルトは fail-open（クラッシュ・タイムアウト・不正JSONでもアクションは通る）。付けると監視アプリの障害で Cursor が止まる
- フックは毎回プロセスを spawn する。`postToolUse` 等は高頻度なので、**NativeAOT の単一 exe** で「stdin 読む → localhost へ投げる → `exit 0`」だけにする。集計は本体側
- 終了コード `2` はアクションをブロックする意味なので、監視用途では絶対に返さない

---

## 4. 現時点の設計方針

**3層構成 / SQLite は触らない**

| 層 | ソース | 役割 |
|---|---|---|
| リアルタイム層 | Hooks | 状態遷移・ターン・ツール実行・エラー。導入後の主データ |
| ヒストリカル層 | `agent-transcripts/*.jsonl` | 導入前セッションの取り込み |
| （不使用） | globalStorage の SQLite | コピーも直読みもしない |

**状態遷移の素案**

```
sessionStart          → 待機中
beforeSubmitPrompt    → 処理中
preToolUse/postToolUse→ 処理中（ツール実行中）
afterAgentResponse    → 応答完了
stop                  → 完了（待機中） ※status で completed/aborted/error を分岐
sessionEnd            → 終了 ※reason を保持
```

**パネルとデータソースの対応**

| 表示項目 | 取得元 | 備考 |
|---|---|---|
| セッション状態 | Hooks | ◎ |
| ターン数・種別・内容 | Hooks | ◎ |
| 経過時間 | Hooks(`duration`, `duration_ms`) | ◎ |
| Thinking時間 | `afterAgentThought.duration_ms` | ◎ |
| ツール実行履歴 | `preToolUse`/`postToolUse` | ◎ |
| モデル名 | 共通ペイロード | ◎ |
| コンテキスト使用率 | `preCompact` のみ | △ 圧縮時点のみ |
| 消費トークン | **なし** | ✕ 断念 or 自前推定 |
| 導入前セッション | transcripts jsonl | 調査次第 |

---

## 5. 社内PCで確認すること（未確定事項）

### Q1. transcript の JSONL スキーマ ★最優先

- 各行の `type` / `event` / `role` 等のイベント種別にどんな値が出るか
- ツール呼び出し・思考ブロック・アシスタント応答が区別できるか
- **タイムスタンプ（epoch ms らしき13桁 int）を持つか**
- トークン数らしきフィールドがあるか（あれば消費トークンを諦めなくて済む）

→ 分岐: 種別＋時刻が揃えば、導入前セッションも導入後とほぼ同粒度で扱える。無ければ「会話本文のみの劣化データ」として別表示にする。

### Q2. `agent-transcripts/<ID>/` の ID 形式 ★重要

- UUID 形式（8-4-4-4-12）か、単なる16進ハッシュか

→ 分岐: UUID なら `composerId` = Hooks の `conversation_id` の可能性が高く、**導入前データと導入後データが同一キーで接続できる**。ここが繋がるかどうかで UI 設計が変わる。

### Q3. プロジェクトフォルダの命名規則

- フルパスを潰した長い名前（`C-Projects-hoge` 等）か、リーフのプロジェクト名だけか

→ 分岐: リーフ名だけなら、別パスの同名プロジェクトが衝突する。グルーピングキーの設計に影響。

### Q4. `transcript_path` の実在確認

- トランスクリプトが無効化されていないか（無効なら `transcript_path` は null）
- 最古のファイル日時 = どこまで遡れるか
- 総ファイル数・総サイズ = 初回インポートの重さ

### Q5. Hooks の実挙動確認

- `hooks.json` を置いて Customize > Hooks タブでロードされるか
- `conversation_id` と Q2 のフォルダ名 ID が**実際に一致するか**（要実測。ドキュメント上は保証されていない）
- 社内ポリシー的にフック（＝任意コマンド実行設定）が許容されるかの確認

### Q6. 参考値

- globalStorage の `state.vscdb` の実サイズ（DB非使用判断の根拠）

---

## 6. 調査スクリプトに持たせる要件

Cursor に書かせる際の仕様として。

**必須の安全要件**

- 完全リードオンリー。Cursor 配下へ一切書き込まない
- **SQLite に接続しない**（サイズは `Get-Item` のファイル情報で見る）
- Cursor 起動中でも安全に実行できること

**スクリプトA: 構造調査**

1. `~/.cursor/projects` 配下のフォルダを3分類（13桁数値 / Temp / プロジェクト名）
2. `agent-transcripts` を再帰探索し、ファイル数・拡張子・総サイズ・最古/最新日時
3. サブフォルダ名（会話ID候補）が UUID 形式かを判定
4. `%APPDATA%\Cursor\User\globalStorage\state.vscdb` と `-wal` / `-shm` のサイズ
5. `hooks.json`（User / Enterprise）の存在確認

**スクリプトB: JSONL スキーマ抽出**

1. JSONL を1行ずつ `ConvertFrom-Json`
2. **値を型プレースホルダに置換**（`<string:len=N>` / `<int>` / `<bool>`）して本文を出さない
3. ただし列挙値になりやすいキー（`type` `role` `event` `kind` `status` `name` `tool_name` `model` 等）は実値を残す。完全に伏せるオプションも用意
4. 13桁の整数は `<int:epoch_ms?>` と印を付ける（タイムスタンプ検出）
5. 出力: トップレベルキーの出現頻度 / イベント種別の分布 / 構造パターン上位N件

**社外へ結果を持ち出す場合は、値が伏せられていることを目視確認してから。**

---

## 7. 参照

- Cursor 公式 Hooks ドキュメント: https://cursor.com/docs/hooks
- Cursor のチャット保存形式のリバースエンジニアリング記録（cursaves）:
  https://github.com/Callum-Ward/cursaves/blob/main/docs/how-cursor-stores-chats.md

---

## 8. リスク・留意点

- **スキーマは非公式**。Cursor 3.0 で実際に破壊的マイグレーションが発生している。パーサーはバージョン別ストラテジで差し替え可能にしておく
- トークン数・コスト表示は Hooks では不可能。無理に推定値を出すなら「概算」と明示する
- 導入前データと導入後データは粒度が異なる可能性が高い。UI でバッジ等により明示的に区別する
- サブエージェントは `task-toolu_...` のような prefix 付き別 composerId として現れる（DB側の観測）。一覧のノイズになるなら親子で畳む
- Hooks の設定は「AI が任意コマンドを実行する仕組み」でもあるため、社内ポリシーの確認を推奨
