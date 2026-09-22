# ImpactMap Design Doc

C#ソリューションを静的解析し、指定したクラス／メソッドの「呼び出し元」をAPIの入口（Controller）までたどった影響範囲マップをMarkdownで出力するCLIツール。

\---

## 1\. 背景と目的

### 1.1 背景

* 基幹システム（ASP.NET / .NET 10）の改修で、AIエージェント（Cursor / GitHub Copilot）に既存コード調査・仕様書作成をさせている
* トークンは社内でプールされており、使いすぎたユーザーは利用停止になる
* 調査フェーズでは、AIがgrep等でリポジトリを探索するターンが多く、トークン消費が大きい
* 一方で、呼び出し関係の追跡を人間が行うと、深い呼び出し階層（A→B→C→D）やインターフェース経由の呼び出しで必ず漏れが出る

### 1.2 目的

* 「どのメソッドが、どの経路で、どのAPIエンドポイントから呼ばれているか」の列挙を、LLMではなくコンパイラ（Roslyn）で行う
* 出力したマップをAIに渡し、AIの仕事を「探索」から「マップに基づく分析・検証」に変えることで、調査品質を落とさずにトークンを削減する

### 1.3 ゴール

* 静的に追跡可能な呼び出し関係を、漏れなく、LLMトークン0で列挙できること
* 出力がAIにそのまま渡せる程度にコンパクトであること

### 1.4 非ゴール（本バージョンではやらない）

* UI（Angular）側の解析（grepで別途対応する）
* テーブル名・SQL文字列・ストアドプロシージャの依存解析
* リフレクションや文字列指定による動的呼び出しの追跡
* GUI、IDE拡張、MCPサーバー化

\---

## 2\. 利用イメージ

```bash
# クラス全体を起点にする（主な使い方）
dotnet run -- C:\\src\\App.sln MyApp.Repositories.ARepository > impact.md

# メソッドを絞る
dotnet run -- C:\\src\\App.sln MyApp.Repositories.ARepository --method GetOrders --depth 8 > impact.md
```

生成された `impact.md` をAIに渡し、「このマップに載っているファイルだけを読んで分析すること。マップに載っていない依存に気づいたら報告すること」と指示する。

\---

## 3\. CLI仕様

```
ImpactMap <slnPath> <typeFullName> \[--method <methodName>] \[--depth <maxDepth>]
```

|引数|必須|説明|
|-|-|-|
|`slnPath`|○|解析対象の .sln のパス|
|`typeFullName`|○|起点クラスの完全名（名前空間込み。例：`MyApp.Repositories.ARepository`）|
|`--method`|×|起点メソッド名。省略時はクラス全体が起点|
|`--depth`|×|呼び出し元をたどる最大深さ。デフォルト `10`|

### 3.1 入出力

* 結果（Markdown）は **標準出力** に出す（リダイレクトでファイル化する想定）
* 進捗・警告・エラーは **標準エラー出力** に出す（結果ファイルを汚さないため）
* 標準出力のエンコーディングは UTF-8

### 3.2 終了コード

|コード|意味|
|-|-|
|0|正常終了|
|1|引数不正、または起点が見つからない|

\---

## 4\. 起点の決定ルール

### 4.1 `--method` 指定あり

* 指定クラスの、指定名のメソッドすべて（オーバーロードは全部）を起点にする

### 4.2 `--method` 省略（クラス全体）

* 指定クラスで宣言されている以下の条件をすべて満たすメソッドを起点にする

  * 通常のメソッドである（コンストラクタ、プロパティのget/set、イベントアクセサ等は除外）
  * コンパイラが暗黙的に生成したものではない
  * アクセス修飾子が `private` ではない（privateは外部から呼ばれないため起点にしない）
* 起点はソース上の宣言順に並べる（出力を実行ごとに安定させるため）

### 4.3 クラスの特定

* ソリューション内の各プロジェクトのコンパイル結果から、完全名で型を探す
* **その型を定義しているプロジェクト自身** で見つかった場合のみ採用する（参照先経由で同じ型が複数回見つかる重複を防ぐ）

\---

## 5\. 処理フロー

1. MSBuildの場所を登録する
2. ソリューションを読み込む
3. 読み込み失敗の診断情報を標準エラー出力に警告として出す
4. 起点メソッドを決定する（4章）。見つからなければ終了コード1で終了
5. 各起点について、呼び出し元を再帰的にたどる（6章）
6. Markdownを標準出力に書き出す

\---

## 6\. 呼び出し元探索の仕様

### 6.1 検索対象シンボル

あるメソッドの呼び出し元を探すとき、以下すべての呼び出し元を集めて重複排除する。

* そのメソッド自身
* そのメソッドがオーバーライドしている基底メソッド
* そのメソッドが実装しているインターフェースのメソッド

**理由**：対象システムはDIでインターフェース経由の呼び出し（`IARepository.GetOrders()` 等）が標準のため、実装クラスのメソッドだけを検索すると呼び出し元を取りこぼす。

### 6.2 再帰と停止条件

見つかった呼び出し元ごとに、次の順で判定する。

|順|条件|動作|
|-|-|-|
|1|テストプロジェクト内のシンボル|出力しない・たどらない|
|2|すでに出力済みのシンボル|`↻` を付けて出力し、たどらない|
|3|Controllerのメソッド|`🎯` とルート情報を付けて出力し、たどらない|
|4|上記以外|出力し、さらに呼び出し元をたどる|

* 深さが `--depth` を超えたら `⚠️ 最大深さ到達（ここで打ち切り）` を出力して打ち切る

### 6.3 既出判定のスコープ

* 既出判定（`↻`）は **1回の実行全体で共有** する（起点メソッドごとにリセットしない）
* 目的：クラス全体を起点にしたとき出力が爆発するのを防ぐ。AIに渡す上では「影響するルートがすべて載っていること」が重要で、メソッドごとの完全なツリーは不要

### 6.4 テストプロジェクトの判定

* シンボルが属するプロジェクト名に `Test` を含む場合、テストプロジェクトとみなす
* 判定キーワードはコード先頭の定数で変更可能にする

### 6.5 Controllerの判定

* メソッドを含むクラス、またはその基底クラスのいずれかが以下に該当すればControllerとみなす

  * クラス名が `ControllerBase` または `Controller`
  * クラス名が `Controller` で終わる

### 6.6 ルート情報

* Controllerクラスとメソッドに付いている属性のうち、名前が `Http` で始まるもの（`HttpGet` 等）と `Route` を並べて表示する
* 属性の第1引数（ルート文字列）があれば併記する
* 例：`\[Route("api/orders")] \[HttpGet("search")]`

\---

## 7\. 出力フォーマット

```markdown
# 影響範囲マップ: MyApp.Repositories.ARepository（クラス全体・3メソッド）

🎯 = Controller（APIの入口）到達 / ↻ = 別ルートで既出

- `ARepository.GetOrders(int)` (Repositories/ARepository.cs:42)
  - `OrderService.Search(SearchCondition)` (Services/OrderService.cs:88)
    - 🎯 `OrderController.Search(SearchRequest)` (Controllers/OrderController.cs:30) \[Route("api/orders")] \[HttpGet("search")]
  - `ShippingRegister.Build(ShippingData)` (Registers/ShippingRegister.cs:120)
    - `ShippingService.Execute(ShippingRequest)` (Services/ShippingService.cs:55)
      - 🎯 `ShippingController.Post(ShippingRequest)` (Controllers/ShippingController.cs:25) \[Route("api/shipping")] \[HttpPost]
- `ARepository.GetOrderDetail(int)` (Repositories/ARepository.cs:70)
  - ↻ `OrderService.Search(SearchCondition)` (Services/OrderService.cs:88)
```

* ネストしたMarkdownリスト（インデントは深さ×スペース2つ）
* 各行のラベルは ``型名.メソッド名(引数型)` (相対パス:行番号)`

  * パスは .sln のあるディレクトリからの相対パス、区切り文字は `/` に統一
  * 行番号は1始まり
* 起点メソッドはリストのトップレベルに出す

\---

## 8\. 技術仕様

### 8.1 環境

* .NET 10（ツール自身の TargetFramework は `net10.0`）
* 解析対象ソリューションは事前に `dotnet restore` 済みで、Visual Studioで正常に開ける状態であること

### 8.2 使用パッケージ

|パッケージ|役割|
|-|-|
|`Microsoft.Build.Locator`|インストール済みMSBuildの検出・登録|
|`Microsoft.CodeAnalysis.Workspaces.MSBuild`|.sln / .csproj の読み込み|
|`Microsoft.CodeAnalysis.CSharp.Workspaces`|C#プロジェクトの解析（これがないとC#プロジェクトを読めない）|

バージョンは実装時点の最新安定版を `dotnet add package` で取得する。

### 8.3 使用する主なAPI

|用途|API|
|-|-|
|MSBuild登録|`MSBuildLocator.RegisterDefaults()`|
|ソリューション読み込み|`MSBuildWorkspace.Create()` / `OpenSolutionAsync()`|
|読み込み失敗の確認|`MSBuildWorkspace.Diagnostics`（`WorkspaceDiagnosticKind.Failure`）|
|型の検索|`Compilation.GetTypeByMetadataName()`|
|呼び出し元検索|`SymbolFinder.FindCallersAsync()`|
|インターフェース実装の対応|`ITypeSymbol.AllInterfaces` / `FindImplementationForInterfaceMember()`|
|オーバーライド元|`IMethodSymbol.OverriddenMethod`|
|シンボル比較|`SymbolEqualityComparer.Default`|
|所属プロジェクトの取得|`Solution.GetDocument(SyntaxTree)`|

### 8.4 実装上の注意

* `MSBuildLocator.RegisterDefaults()` は、MSBuild関連の型を参照するコードより **前** に、**別メソッド・別クラス** から呼ぶこと。同じメソッド内で `MSBuildWorkspace` を使うと、JITによる型ロードが先に走って失敗する
* 巨大ソリューションは読み込みに時間がかかるため、読み込み開始時に標準エラー出力へ進捗を出す

### 8.5 ファイル構成

```
ImpactMap/
├─ ImpactMap.csproj
├─ Program.cs        … MSBuild登録、引数解析、ImpactMapper呼び出し
└─ ImpactMapper.cs   … 解析本体（起点決定・再帰探索・出力）
```

\---

## 9\. 既知の制約

静的解析で追えないため、マップに現れない依存がある。AIへの指示および人間のレビューで補完する。

* リフレクション、文字列指定による動的呼び出し
* SQL文字列・ストアドプロシージャ内の依存
* 設定ファイルやDB設定値で振る舞いが変わる箇所
* DI登録が条件分岐している箇所（どの実装が注入されるかは実行時に決まる）
* 読み込みに失敗したプロジェクト内の呼び出し（3章の警告で検知する）

\---

## 10\. 受け入れ条件・検証方法

実装完了後、以下を満たすことを確認する。

|#|確認内容|方法|
|-|-|-|
|1|ビルドが通る|`dotnet build`|
|2|引数不足・存在しない型で終了コード1になる|手動実行|
|3|メソッド指定時の結果が正しい|よく知っているメソッドで実行し、Visual Studioの「呼び出し階層の表示」（Ctrl+K, Ctrl+T）の結果と突き合わせる|
|4|インターフェース経由の呼び出しが拾えている|DI経由で呼ばれているRepositoryのメソッドで実行し、Service側の呼び出しが出ることを確認|
|5|Controllerで止まり、ルートが表示される|出力の🎯行を確認|
|6|テストプロジェクトからの呼び出しが出ない|テストから呼ばれているメソッドで実行|
|7|クラス全体指定でprivateメソッドが起点に含まれない|privateメソッドを持つクラスで実行|
|8|同一ソリューション・同一引数で出力が毎回同じ|2回実行してdiff|

### 10.1 要検証ポイント

以下はRoslynの挙動に依存するため、実装時に実際の出力で確認する。

* ラムダ式・ローカル関数の中からの呼び出しで、呼び出し元として何が返るか（外側のメソッドになっているか）
* ジェネリックなRepository基底クラス（`RepositoryBase<T>` 等）を使っている場合に、呼び出し元が正しく拾えるか

\---

## 11\. 将来の拡張候補（今回は対象外）

* 🎯のルート文字列を使ったUI（Angular）側のgrep結果の自動結合
* テーブル名・カラム名のgrep結果の併記
* 出力のキャッシュ（コミットハッシュ単位）
* MCPサーバー化してエージェントから直接呼び出せるようにする

