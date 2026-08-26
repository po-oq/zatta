# browser-agent — 要件定義 / 技術評価

**認証が必要なWebページのDOMを、AIエージェントに代わって取得するローカルアプリ**

| 項目 | 内容 |
|---|---|
| ドキュメント種別 | 要件定義 + 技術実現性評価 |
| ステータス | ドラフト（実装未着手） |
| 最終更新 | 2026-08-27 |

---

## 1. 背景と課題

AIエージェント（Claude Code 等）は、認証が必要なWebページの内容を取得できない。

- エージェント自身はブラウザセッションを持たないため、社内システムやSSO配下のページにアクセスできない
- 一般的な解決策であるMCPサーバー経由のブラウザ操作は、**利用環境の管理者権限ポリシーによりMCPが利用できない**ため採用不可

このため、**エージェントから起動できるローカルEXE**として、認証を人間に委譲しつつDOMを取得する仕組みを構築する。

### 中心的なアイデア

> 「認証が必要かどうかの自動判定」という難問を解くのではなく、**人間を検出器として使う**。
> ログイン画面が出たらウィンドウを表示し、人間がログインする。認証完了の検知はURL遷移で行う。

---

## 2. 用語

| 用語 | 定義 |
|---|---|
| Agent | 本アプリを起動する側のAIエージェント（Claude Code 等） |
| requestedUrl | Agentが引数で渡した、取得したい目的ページのURL |
| currentUrl | WebView2が現在表示しているURL |
| UDF | User Data Folder。WebView2がCookie等を保存するフォルダ |
| 認証壁 | 未認証時にリダイレクトされるログイン画面（自サイト内 or 外部IdP） |
| settle / 静穏 | 一定時間、新しいナビゲーションイベントが発生しない状態 |

---

## 3. 要件定義

### 3.1 ユースケース

```
[UC-1] 認証不要ページの取得
  Agent → browser-agent.exe "https://aaa.com/public/123"
        → (UIは表示されない)
        → stdout に DOM情報のJSON、exit 0
  所要時間: 1〜3秒。ユーザーの操作は一切不要。

[UC-2] 認証必要ページの取得（初回）
  Agent → browser-agent.exe "https://aaa.com/detail/123"
        → https://aaa.com/login にリダイレクトされる
        → ★ここでウィンドウが表示される
        → ユーザーがSSOログイン
        → 目的ページに到達
        → stdout に DOM情報のJSON、ウィンドウ自動クローズ、exit 0
  ユーザーの操作: ログインのみ。ボタン等の追加操作は不要。

[UC-3] 認証必要ページの取得（2回目以降）
  Agent → browser-agent.exe "https://aaa.com/detail/456"
        → UDFに残ったセッションで認証済みのまま表示
        → UC-1と同じ挙動（UIは表示されない）
```

### 3.2 機能要件

| ID | 要件 | 優先度 |
|---|---|---|
| FR-01 | コマンドライン引数でURLを1つ受け取る | 必須 |
| FR-02 | WebView2で当該URLへナビゲートする | 必須 |
| FR-03 | 目的ページに到達したと判定したら、DOM情報を標準出力にJSONで書き出す | 必須 |
| FR-04 | 出力後、アプリは自動的に終了する | 必須 |
| FR-05 | 認証不要と判定した場合、UIウィンドウを表示しない | 必須 |
| FR-06 | 認証壁を検知した場合、UIウィンドウを表示して前面に出す | 必須 |
| FR-07 | ユーザーのログイン完了を自動検知し、追加操作なしでFR-03へ進む | 必須 |
| FR-08 | セッション情報（Cookie等）を固定UDFに永続化し、次回起動時に再利用する | 必須 |
| FR-09 | 処理結果を終了コードで表現する | 必須 |
| FR-10 | タイムアウトを設け、期限内に完了しない場合は諦めて終了する | 必須 |
| FR-11 | 出力形式を選択できる（markdown / text / html） | 推奨 |
| FR-12 | DOMをファイルに出力するオプション（`--out`） | 推奨 |
| FR-13 | 認証状態のクリア（`--clear-session`） | 任意 |

### 3.3 非機能要件

| ID | 要件 |
|---|---|
| NFR-01 | 標準出力にはJSONのみを出力する。ログ・進捗は全て標準エラー出力へ |
| NFR-02 | **Cookie・アクセストークン・セッションIDをAgentに返さない** |
| NFR-03 | DOM出力前に `input[type=password]` の値を除去する |
| NFR-04 | UC-1の所要時間は3秒以内を目標とする |
| NFR-05 | UDFは適切なACLで保護する（実質「ログイン済み全サイトの鍵束」であるため） |
| NFR-06 | `--help` はAgentが読んで使い方を理解できる粒度で記述する |

### 3.4 スコープ外

- ページ内の操作（クリック、フォーム入力等のブラウザ自動化）
- 複数URLの一括取得
- ヘッドレス実行（そもそも人間の認証を前提とするため）
- 認証情報（ID/パスワード）のアプリによる保管・自動入力
- **OAuthトークンの抜き取り**（§6.3 参照。設計上明確に禁止）
- Windows以外のプラットフォーム

### 3.5 前提条件

| 前提 | 状態 |
|---|---|
| 対象サイトの認証は主にSSO。未認証時はログインURLへリダイレクトされる | ユーザー確認済み |
| 実行環境はWindows | 確認済み |
| MCPは管理者権限により利用不可 | ユーザー確認済み |
| WebView2 Runtimeが導入済み（Win11は標準搭載） | **要検証** |
| 自作の未署名EXEが実行可能（AppLocker/WDAC制限なし） | **要検証** |
| Agentの実行環境（Windowsネイティブ / WSL） | **未確認** |
| .NETのターゲットバージョン | **未確認** |

---

## 4. 技術実現性評価

### 4.1 総合判定

**実現可能。** 技術的なブロッカーは存在しない。唯一の設計課題であった「認証が必要かの判定」は、人間にUIを見せる方式により回避される。

| 要素 | 実現性 | 備考 |
|---|---|---|
| C#でWebView2をホストする | ◎ | WPF/WinFormsで標準的 |
| 引数でURLを受け取る | ◎ | — |
| DOM取得（`ExecuteScriptAsync`） | ◎ | 戻り値の二重デコードに注意 |
| Cookie / LocalStorage / IndexedDB の永続化 | ◎ | 固定UDFで実現 |
| **セッションCookieの永続化** | **△** | §4.3 参照。要実測 |
| 認証壁の検知 | ○ | ヒューリスティック。§5.3 |
| 認証完了の検知 | ○ | URL遷移監視。§5.3 |
| GUIアプリからの標準出力 | ○ | §6.1 の対応が必須 |
| 認証不要時のUI非表示 | ○ | §6.2 の対応が必須 |

### 4.2 技術選択

#### 採用：WebView2（WPF ホスト）

| 選択肢 | 評価 | 判断 |
|---|---|---|
| **WebView2 + WPF** | Windows専用。UI組み込みの自由度が高い。.NETネイティブ | **採用** |
| Playwright for .NET | クロスプラットフォーム。2回目以降のヘッドレス化が可能 | 不採用。ヘッドレス化は本件の要件ではなく、ブラウザバイナリの追加配布が負担 |
| 既存Chromeへ CDP アタッチ | 既存ログインを流用できれば理想的 | **不可**（§4.4） |

#### 採用：単一EXE（常駐なし）

当初は「薄いCLI + 常駐UIプロセス（Named Pipe）」の2プロセス構成を検討したが、
**UIウィンドウ自体が認証待ちの受け皿になる**ため、プロセスを分離する必要がなくなった。

```
Agent ──起動──> browser-agent.exe（単一プロセス）
                    ├─ 画面外にウィンドウ生成（WebView2はHWNDが必要）
                    ├─ 認証不要 → そのままキャプチャして終了（ウィンドウは不可視のまま）
                    └─ 認証必要 → ウィンドウを画面内へ移動して表示
                                  → ログイン完了を検知 → キャプチャ → 終了
```

### 4.3 セッション永続化に関する検証結果

**確認済みの事実：**

WebView2はUDFにブラウザデータを保存する。
> [Manage user data folders — Microsoft Learn](https://learn.microsoft.com/en-us/microsoft-edge/webview2/concepts/user-data-folder)
> UDFはユーザーのマシン上に保存されるフォルダで、Cookie、パーミッション、キャッシュ済みリソースなどのブラウザデータを格納する。カスタムUDFの場所を明示指定できる。

したがって、**永続Cookie（`Expires`/`Max-Age` あり）およびLocalStorage/IndexedDBは、アプリ再起動をまたいで維持される。**

**注意が必要な事実：**

| Cookie種別 | ディスクに書かれるか | アプリ再起動後 |
|---|---|---|
| 永続Cookie（`Expires` あり） | ○ | **残る** |
| セッションCookie（`Expires` なし） | ✕（HTTP仕様上メモリのみ） | **消える** |
| `sessionStorage` | ✕ | **消える（確定）** |

セッションCookieの永続化オプションについて、WebView2に該当機能がない旨の要望が挙がっている。
> [Issue #3444 — How to make session cookie persistence](https://github.com/MicrosoftEdge/WebView2Feedback/issues/3444)
> CEFには `persistent_session_cookies` というセッションCookie永続化オプションがあったが、Edge WebView2には同等のサポートが見当たらない。

また、SSO環境で再起動のたびに再認証が要求される事象の報告がある（**Closed済み。ただし解決内容は未確認、2021年の古いRuntimeでの報告のため現在は解消されている可能性あり**）。
> [Issue #1167 — Some session data are not persisted in the same way as Edge/Chrome](https://github.com/MicrosoftEdge/WebView2Feedback/issues/1167)

**影響とリスク：**

- 「ログイン状態を保持する」型の永続Cookieを発行するサイト → **問題なし**
- 素のASP.NET / Java のセッション、クライアント証明書認証、`sessionStorage` にトークンを置くSPA → **アプリ終了で認証が切れる可能性**

**対応方針：**
Phase 0 で実測して判定する（§8）。仮にセッションが切れる場合でも、UC-2のフロー（ログイン画面が出たらUIを表示する）は正常に機能するため、**「毎回ログインが必要になる」という不便さに留まり、機能不全にはならない。**
どうしても回避したい場合のみ、常駐プロセス方式へ切り替える。

### 4.4 検討したが不採用とした案

#### 既存Chromeへの CDP アタッチ

ユーザーが日常的に使っているChromeにアタッチできれば、既存のログインセッションをそのまま流用できるが、**Chrome 136以降で封じられている。**

> [Changes to remote debugging switches to improve security — Chrome for Developers](https://developer.chrome.com/blog/remote-debugging-port)
> Chrome 136から `--remote-debugging-port` および `--remote-debugging-pipe` は、デフォルトのChromeデータディレクトリをデバッグしようとする場合には尊重されなくなる。これらのスイッチは、非標準のディレクトリを指す `--user-data-dir` スイッチと併用しなければならない。

`--user-data-dir` を指定した時点で新規プロファイルとなり、ログイン状態はゼロになる。**本案は成立しない。**

---

## 5. 設計

### 5.1 コマンドラインインターフェース

```
browser-agent.exe <url> [options]

  <url>                取得したいページのURL（必須）

  --timeout <sec>      認証待ちを含む全体のタイムアウト（既定: 240）
  --format <fmt>       markdown | text | html （既定: markdown）
  --out <path>         標準出力ではなく指定パスへ本文を書き出す
  --strict             目的URLへの完全到達を必須とする（§5.4 loose match を無効化）
  --clear-session      UDFを削除してから起動する
  --help
```

### 5.2 出力仕様

**標準出力（JSONのみ）:**

```jsonc
{
  "status": "SUCCESS",
  "requestedUrl": "https://aaa.com/detail/123",
  "finalUrl": "https://aaa.com/detail/123?code=0.AXk...",
  "title": "詳細 123",
  "httpStatus": 200,
  "authWasRequired": true,      // 途中でログインを挟んだか
  "matched": "exact",           // exact | loose
  "elapsedMs": 42310,
  "chars": 18420,
  "content": "# 詳細 123\n\n..."  // --out 指定時は contentPath になる
}
```

**終了コード:**

| コード | 意味 | Agentの想定動作 |
|---|---|---|
| 0 | SUCCESS | contentを利用する |
| 1 | 引数エラー | 使い方を修正して再実行 |
| 2 | TIMEOUT（認証が完了しなかった） | ユーザーに確認し再実行を提案 |
| 3 | CANCELLED（ユーザーがウィンドウを閉じた） | 中止として報告 |
| 4 | FAILED（DNS・ネットワーク・4xx/5xx等） | エラーを報告 |
| 5 | BUSY（別インスタンスがUDFを使用中） | 待機して再試行 |

**標準エラー出力：** 進捗ログ・デバッグ情報（Agentは無視してよい）

### 5.3 状態機械

```
                    Navigate(requestedUrl)
                            │
                            ▼
                      ┌──────────┐
              ┌──────>│ LOADING  │
              │       └────┬─────┘
              │            │ 800ms 静穏（新規ナビゲーションなし）
        ナビ発生            ▼
              │       ┌──────────┐
              └───────┤  JUDGE   │
                      └────┬─────┘
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        IsTarget=true  認証シグナル   どちらでもない
              │         あり          │
              │            │          │ 2xx かつ認証シグナルなし
              ▼            ▼          ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │  READY   │ │AUTH_WAIT │ │  READY   │
        │ (exact)  │ │          │ │ (loose)  │
        └────┬─────┘ └────┬─────┘ └────┬─────┘
             │            │  ★UIを画面内へ表示  │
             │            │  ★監視は継続       │
             │            │                   │
             │            │ 認証壁を抜けた     │
             │            ▼                   │
             │      ┌───────────────┐          │
             │      │ RE-NAVIGATE   │──────────┤
             │      │ (1回のみ)      │          │
             │      └───────────────┘          │
             ▼                                 ▼
        ┌───────────────────────────────────────┐
        │  CAPTURE → stdout → Exit(0)           │
        └───────────────────────────────────────┘
```

**タイムアウト / ウィンドウクローズはどの状態からでも発生しうる。**

### 5.4 判定ロジックの詳細

#### (a) 「静穏」判定（デバウンス）

SSOのリダイレクト連鎖は複数回のナビゲーションを発生させる（JSリダイレクト、SAMLの自動POSTフォーム等）。
**「止まった」はイベントではなく、イベントの不在でしか判定できない。**

```
NavigationStarting        → タイマー停止、状態を LOADING へ
NavigationCompleted       → タイマーを 800ms でリスタート
SourceChanged             → タイマーを 800ms でリスタート
タイマー発火               → JUDGE へ
```

購読すべきイベントは以下。**特に `SourceChanged` が重要。**

> [CoreWebView2 — Microsoft Learn](https://learn.microsoft.com/en-us/microsoft-edge/webview2/reference/winrt/microsoft_web_webview2_core/corewebview2)
> `SourceChanged` は別サイトへの遷移やフラグメントナビゲーションで発火する。ページ更新や、現在のURLと同じURLでの `history.pushState` では発火しない。
> `ContentLoading` は同一ページ内ナビゲーション（フラグメントや `history.pushState`）では発火しない。
> イベント順序は `NavigationStarting` → `SourceChanged` → `ContentLoading` → `HistoryChanged` → `NavigationCompleted`。

| イベント | 通常遷移 | pushState | フラグメント |
|---|---|---|---|
| `NavigationStarting` | ○ | ✕ | ✕ |
| `SourceChanged` | ○ | ○ | ○ |
| `NavigationCompleted` | ○ | ✕ | ✕ |

→ SPAに着地した場合、`NavigationCompleted` だけを見ていると遷移を取りこぼす。

#### (b) URL一致判定（正規化必須）

SSO往復から戻ると、URLは文字列として一致しないことが多い。

```
要求: https://aaa.com/detail/123
戻り: https://aaa.com/detail/123?code=0.AXk...&state=abc   ← OAuthコールバック残骸
戻り: https://aaa.com/detail/123/                          ← 末尾スラッシュ
戻り: https://aaa.com/detail/123#/tab/summary              ← SPAがfragment付与
```

**ホスト＋パスのみを比較し、クエリとフラグメントは無視する。**

```csharp
static bool IsTarget(string current, Uri requested)
{
    if (!Uri.TryCreate(current, UriKind.Absolute, out var c)) return false;
    return string.Equals(c.Host, requested.Host, StringComparison.OrdinalIgnoreCase)
        && string.Equals(c.AbsolutePath.TrimEnd('/'),
                         requested.AbsolutePath.TrimEnd('/'),
                         StringComparison.OrdinalIgnoreCase);
}
```

#### (c) 認証壁の検知

シグナルを強い順に評価する。**単純なURLキーワードマッチより「オリジンが変わったか」のほうが遥かに信頼できる。**

| # | シグナル | 強度 | 根拠 |
|---|---|---|---|
| 1 | HTTPステータスが 401 / 403 | 確実 | 明示的な認証・認可エラー |
| 2 | **ホストが requestedUrl と異なる** | 確実 | 企業SSOは外部IdPへ飛ぶ（`login.microsoftonline.com`, `adfs.corp.local`, `*.okta.com`） |
| 3 | **戻り先を保持するクエリの存在** | 確実 | `ReturnUrl` `redirect_uri` `RelayState` `wctx` `next` — 「元の場所に戻す意図」の証拠 |
| 4 | パスがログイン系パターン | 有力 | `/login` `/signin` `/auth` `/sso` `/oauth2` `/adfs` `/saml` |
| 5 | `input[type=password]` が存在 | 有力 | — |

`/adfs/ls/` や `/oauth2/v2.0/authorize` のようにパスに `login` を含まないケースがあるため、#4 単独に依存してはならない。

#### (d) loose match（重要な設計判断）

`IsTarget = false` かつ認証シグナルもない場合（例：`/detail/123` → `/detail/123/overview`）、
**厳格に判定するとUIが無駄に表示されてしまう。**

→ **HTTPステータスが2xxで認証シグナルがなければ、`matched: "loose"` として成功扱いとする。**
厳格に扱いたい場合は `--strict` を指定する。

#### (e) 認証後の再ナビゲート（重要な設計判断）

SSOから戻ったとき、目的URLではなく `/home` や `/dashboard` に着地するサイトが存在する。
そのままでは `IsTarget` が永遠に成立しない。

→ **AUTH_WAIT状態から「認証壁を抜けた」と判定したら、requestedUrl へ1回だけ再ナビゲートする。**
これにより判定ロジックを単純に保てる。**無限ループ防止のため再ナビゲートは1回まで。**

---

## 6. 技術的リスクと対策

### 6.1 【最重要】GUIアプリの標準出力が消える

WPFアプリの既定である `OutputType=WinExe` では、**`Console.WriteLine` が何も出力せず、実行時エラーも出ない。**

> [How to get console output from a WPF application — Gary Ewan Park](https://www.gep13.co.uk/blog/how-to-get-console-output-from-wpf-application)
> `OutputType` が `WinExe`（WPFアプリがそうである）でコンパイルされると、コンソールに何も出力されない。実行時エラーも出ないため、何が起きているのか気づきにくい。

**対策：`OutputType` を `Exe`（コンソールサブシステム）にする。**

```xml
<PropertyGroup>
  <OutputType>Exe</OutputType>
  <DisableWinExeOutputInference>true</DisableWinExeOutputInference>
  <TargetFramework>net8.0-windows</TargetFramework>
  <UseWPF>true</UseWPF>
</PropertyGroup>
```

コンソールアプリからでもWPFウィンドウは問題なく表示できる。`Console.Out` は最初から正しく接続され、リダイレクトもそのまま機能する。

**トレードオフ：**
> [Creating Dual Use Windows GUI and Console Applications — Rick Strahl](https://weblog.west-wind.com/posts/2026/Jun/23/Creating-Dual-Use-Windows-GUI-and-Console-Applications)
> コンソールアプリケーションとしてコンパイルすると常にコンソールウィンドウが表示され、起動時にそれを完全に隠す良い方法はない。

ただし本件では、**Agentがプロセスを起動する際は親のコンソールを継承する（または `CreateNoWindow`）ため、新規の黒いウィンドウは出現しない。** 手動でダブルクリック起動した場合のみコンソールが表示されるが、想定用途外のため許容する。

`DisableWinExeOutputInference` は保険。.NET SDK 5.0.100 で `Exe`→`WinExe` の自動変換が入り、.NET 6 で撤回された経緯がある。
- [自動変換の導入](https://learn.microsoft.com/en-us/dotnet/core/compatibility/sdk/5.0/automatically-infer-winexe-output-type)
- [.NET 6での撤回](https://learn.microsoft.com/en-us/dotnet/core/compatibility/sdk/6.0/outputtype-not-set-automatically)

**代替案：** `WinExe` + `AttachConsole(ATTACH_PARENT_PROCESS)` のP/Invoke。黒いウィンドウは絶対に出ないが、リダイレクト時の挙動やプロンプト復帰のタイミングに癖がある。まず `Exe` で試し、問題があれば切り替える。

### 6.2 見えないウィンドウでWebView2が初期化されない

WebView2はHWNDを必要とするため、ウィンドウ自体は生成しなければならない。
しかし `Visibility.Hidden` にするとWPFのレイアウトが走らず、初期化や描画が正常に完了しないことがある。

**対策：画面外に配置して `Show()` する。**

```csharp
var win = new MainWindow {
    WindowStartupLocation = WindowStartupLocation.Manual,
    Left = -32000, Top = -32000,   // 画面外
    ShowInTaskbar = false,
    Width = 1200, Height = 900
};
win.Show();   // 「表示」はするが見えない
```

認証が必要になった時点で画面内へ移動し、前面化する。
バックグラウンドプロセスからの `SetForegroundWindow` はWindowsに拒否されるため、`Topmost` のON/OFFトグル等の回避策が必要。

### 6.3 OAuth埋め込みブラウザとしての利用に関する注意

Microsoftは、WebView2をOAuthの埋め込みブラウザとして使い、リダイレクトURLやCookieからトークンを抜き取る設計を避けるよう案内している。

本設計は以下の線引きを厳守する：

| | 内容 |
|---|---|
| **行う** | WebView2内でユーザーが通常どおり認証し、**認証状態はWebView2のプロファイル内に閉じ込める**。認証済みページのDOMのみを読む |
| **行わない** | リダイレクトURLやCookieから `access_token` を抽出し、Agentに渡す |

NFR-02（Cookie・トークンをAgentに返さない）はこの原則の実装である。

なお、Microsoft Entra ID の条件付きアクセス等で、埋め込みブラウザからの認証自体がブロックされる可能性がある。**対象サイトごとに要確認。**

### 6.4 その他のリスク

| # | リスク | 対策 |
|---|---|---|
| 1 | UDFの排他ロック（1UDFにつき同時1セッション） | Mutexで単一化。競合時は exit 5 (BUSY) |
| 2 | `ExecuteScriptAsync` の戻り値がJSONエンコードされた文字列 | 二重デコードが必要（`JsonSerializer.Deserialize<string>` してから中身をパース） |
| 3 | iframe内のコンテンツは `outerHTML` に含まれない | MVPでは非対応。必要なら `CoreWebView2Frame.ExecuteScriptAsync` |
| 4 | 目的URLに到達したが中身が「権限がありません」 | HTTPステータスと本文長を併せて評価。UIが出ていればユーザーが判断できる |
| 5 | DOM全文の標準出力によるコンテキスト圧迫 | 既定を `markdown` に。`--out` でファイル出力も選択可 |
| 6 | WebView2 Runtime未導入 | 起動時に検出し、明示的なエラーメッセージで終了 |
| 7 | 認証待ち中のAgent側ツールタイムアウト | `--timeout` をAgent側タイムアウトより短く設定。超過時は exit 2 で素直に諦める（ぶら下がらない） |
| 8 | AppLocker / WDAC による未署名EXEの実行制限 | **Phase 0 で先に検証**（§8） |

---

## 7. 未解決事項

| # | 項目 | 影響 |
|---|---|---|
| 1 | Agentの実行環境（Windowsネイティブ / WSL） | `--out` のパス形式（`C:\...` vs `/mnt/c/...`）とプロセス起動方法が変わる |
| 2 | .NETのターゲットバージョン | 現状 net8.0-windows を想定 |
| 3 | 対象サイトのセッションCookie挙動 | §4.3。常駐方式への切り替え要否が決まる |
| 4 | AppLocker等による実行制限の有無 | 制限があれば配布方式ごと見直しが必要 |
| 5 | Entra ID 条件付きアクセスによる埋め込みブラウザ拒否の有無 | 拒否される場合、本方式自体が対象サイトで使えない |

---

## 8. 段階的実装計画

### Phase 0 — 前提検証（所要 30分・**実装前に必須**）

設計を1行も書く前に、以下を実機で確認する。結果次第で構成が変わるため。

1. 自作の未署名EXEが実行できるか（AppLocker / WDAC）
2. WebView2 Runtimeの導入有無
3. Agentの実行環境（Windowsネイティブ / WSL）
4. 対象サイトが埋め込みブラウザからの認証を許可するか

### Phase 1 — セッション永続テスト（所要 30分・**最重要**）

最小のWebView2アプリ（固定UDF、URL入力欄のみ）で以下を実測する。

```
1. アプリ起動 → 対象サイトにログイン → 目的ページ表示
2. アプリを完全終了（プロセス消滅を確認）
3. 再起動して同じURLを開く
4. ログイン画面に飛ぶか？ 飛ばないか？
```

- **飛ばない** → 単一EXE方式で確定。Phase 2へ
- **飛ぶ** → 「毎回ログインが必要」を許容するか、常駐方式に切り替えるかを判断

### Phase 2 — MVP

- 単一EXE、UC-1 / UC-2 の基本フロー
- 状態機械（デバウンス800ms、URL正規化、認証シグナル判定）
- 画面外ウィンドウ → 認証時のみ可視化
- 標準出力JSON + 終了コード
- 出力は `text`（`document.body.innerText`）のみ

### Phase 3 — 実用化

- markdown変換（`--format`）
- `--out` によるファイル出力
- loose match / `--strict`
- 認証後の再ナビゲート
- パスワード欄のマスク
- `--clear-session`

### Phase 4 — 拡張（必要になったら）

- サイト固有ルールの設定ファイル（ログインURL・認証済み判定セレクタをドメイン単位で定義）
- iframe対応
- 常駐化（Phase 1の結果次第）

---

## 9. 参考資料

### WebView2

- [Manage user data folders](https://learn.microsoft.com/en-us/microsoft-edge/webview2/concepts/user-data-folder)
- [CoreWebView2 クラス（イベント一覧）](https://learn.microsoft.com/en-us/microsoft-edge/webview2/reference/winrt/microsoft_web_webview2_core/corewebview2)
- [CoreWebView2.SourceChanged](https://learn.microsoft.com/en-us/dotnet/api/microsoft.web.webview2.core.corewebview2.sourcechanged)
- [CoreWebView2NavigationCompletedEventArgs.HttpStatusCode](https://learn.microsoft.com/en-us/dotnet/api/microsoft.web.webview2.core.corewebview2navigationcompletedeventargs.httpstatuscode)
- [Issue #3444 — セッションCookie永続化の要望](https://github.com/MicrosoftEdge/WebView2Feedback/issues/3444)
- [Issue #1167 — セッションデータがEdge/Chromeと同様に永続化されない](https://github.com/MicrosoftEdge/WebView2Feedback/issues/1167)

### .NET

- [OutputType の自動推論（.NET 5 で導入）](https://learn.microsoft.com/en-us/dotnet/core/compatibility/sdk/5.0/automatically-infer-winexe-output-type)
- [OutputType 自動設定の撤回（.NET 6）](https://learn.microsoft.com/en-us/dotnet/core/compatibility/sdk/6.0/outputtype-not-set-automatically)
- [Creating Dual Use Windows GUI and Console Applications](https://weblog.west-wind.com/posts/2026/Jun/23/Creating-Dual-Use-Windows-GUI-and-Console-Applications)
- [How to get console output from a WPF application](https://www.gep13.co.uk/blog/how-to-get-console-output-from-wpf-application)

### 不採用案の根拠

- [Changes to remote debugging switches to improve security（Chrome 136）](https://developer.chrome.com/blog/remote-debugging-port)
