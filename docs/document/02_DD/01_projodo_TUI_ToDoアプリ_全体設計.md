# Projodo TUI ToDoアプリ（MVP）設計

## 1. 全体設計

### 1.1 アーキテクチャ方針

* UI（Textual）と、todo.txt入出力・データ操作を分ける
* todo.txtの生テキストを最終的な正とし、内部では表示と検索に必要な最小限の抽出だけ行う
* 画面は1つのメイン画面＋追加・編集はモーダルを出して操作するようにする

### 1.2 コンポーネント分割（Ports & Adapters）

* コア（内側）

  * ドメイン

    * TaskRef

      * line_no: int（todo.txt の行番号。0-based）

    * Task

      * ref: TaskRef
      * is_done: bool
      * body: str
      * projects: list[str]（+project）
      * contexts: list[str]（@context）
      * kv: dict[str, str]（key:value）
      * raw_line: str（読み込み時の1行。デバッグ用）

  * アプリケーション（ユースケース）

    * TodoStore（TodoUseCase実装）

      * TaskRepository からロード／セーブする
      * UIからの操作を TaskRef で受けて tasks を更新する
      * 変更が入ったら dirty を true にする

  * ポート

    * Inbound

      * TodoUseCase（UIが呼ぶインターフェイス）

        * list_projects() -> list[str]
        * list_tasks(selected_project: str, query: str) -> list[Task]
        * add_task(body: str, selected_project: str) -> None
        * update_task(ref: TaskRef, body: str, is_done: bool) -> None
        * toggle_done(ref: TaskRef) -> None
        * save() -> None

    * Outbound

      * TaskRepository（永続化のインターフェイス）

        * load() -> list[Task]
        * save(tasks: list[Task]) -> None

* アダプタ（外側）

  * 入力アダプタ（Driving adapter）

    * Textual UI

      * ProjodoApp / MainScreen / AddTaskModal / EditTaskModal
      * TodoUseCase を呼び出し、結果を表示に反映する

  * 出力アダプタ（Driven adapter）

    * TodoTxtRepository（TaskRepository 実装）

      * ファイルI/O（todo.txt の読み書き）
      * TodoTxtParser を使って行のパース／組み立てを行う

    * TodoTxtParser

      * parse_line: 1行文字列から Task を生成（未知トークンは無視する）
      * build_line: Task から 1行文字列を生成（トークン順は固定）

* 構成（Composition Root）

  * app.py で TodoStore と TodoTxtRepository を組み立てて注入する

### 1.3 データフロー

* 起動

  * 引数から todo.txt パス確定

  * TodoStore.load（内部で TaskRepository.load）で読み込み

  * UIへ渡して初期描画（project一覧、タスク一覧）

* 画面操作

  * 選択project変更、検索語変更

    * TodoUseCase.list_tasksで表示対象を再計算

    * 中央リストを再描画

  * 追加

    * AddTaskModal で本文入力

    * TodoUseCase.add_task を呼ぶ

    * TodoStore が tasks を更新し、dirty を true にする

    * project一覧とタスク一覧を再計算して再描画

  * 編集／完了トグル

    * UIは選択行に対応する Task.ref（TaskRef）を渡す

    * TodoUseCase.update_task / toggle_done を呼ぶ

    * TodoStore が該当 Task（ref.line_no）を更新し、dirty を true にする

    * project一覧とタスク一覧を再計算して再描画

* 保存

  * s または q で TodoUseCase.save を呼ぶ

  * TodoStore が TaskRepository.save に tasks を渡して書き戻す

  * 保存完了で dirty を false に戻す

### 1.4 永続化と互換性

* 保存は todo.txt の1行=1タスクを維持する

* 保存時は TodoTxtParser.build_line により、トークン順を固定して出力する

  * 完了タスク

    * x

    * completion_date（存在する場合）

    * priority（存在する場合）

    * creation_date（存在する場合）

    * body

    * contexts（@context）

    * projects（+project）

    * key:value

  * 未完了タスク

    * priority（存在する場合）

    * creation_date（存在する場合）

    * body

    * contexts（@context）

    * projects（+project）

    * key:value

* completion_date / creation_date の自動付与は行わない

* 未知のトークン（このアプリが解釈できない表現）は、MVPでは保持しない

  * 読み込み時に無視し、保存時にも出力しない

### 1.5 例：ディレクトリ構成案

* projodo/

  * **init**.py

  * app.py

    * Composition Root（依存の組み立て、起動引数、App起動）

  * ui/

    * screens/

      * main.py

      * add_task.py

      * edit_task.py

  * core/

    * domain/

      * model.py

    * application/

      * store.py

  * ports/

    * inbound.py

      * TodoUseCase（Protocolなど）

    * outbound.py

      * TaskRepository（Protocolなど）

  * adapters/

    * todotxt_repository.py

      * TodoTxtRepository（TaskRepository実装）

    * todotxt_parser.py

      * todo.txt行のパース／組み立て

* tests/

  * core/

  * adapters/

    * in_memory_repository.py（必要なら）

* pyproject.toml

## 2. 画面・操作・状態設計

### 2.1 MainScreen（一覧画面）

#### 2.1.1 画面レイアウト

* 左サイドバー

  * project一覧（+projectをユニーク化して表示）

  * 先頭に疑似projectとして all を置き、全タスク表示に使う

* 中央

  * フィルタ後のタスク一覧

  * 表示は1行=1タスク（raw_lineベース）

* 下部

  * ステータス表示（選択project、検索語、件数、dirtyなど）

  * キーヘルプ（Footer相当）

  * 検索入力（通常は非表示、/で表示して入力開始）

#### 2.1.2 状態（State）

* store

  * todo: TodoUseCase（実体は TodoStore）

  * path: Path（todo.txt パス）

  * dirty: bool

    * 未保存の変更があるとき true

    * TodoUseCase.save の完了で false

* view state

  * selected_project: str（all または +project）

  * search_query: str

  * focus_target: sidebar または tasks または search

  * selected_task_index: int（中央一覧の選択行）

  * visible_tasks: list[Task]（表示中の一覧。TaskRef を含む）

#### 2.1.3 画面更新ルール

* selected_projectが変わったとき

  * filter_tasksで表示対象を再計算

  * selected_task_indexを0へ寄せる

* search_queryが変わったとき

  * filter_tasksで表示対象を再計算

  * 件数とステータスを更新

* tasksが変わったとき（追加、編集、完了トグル）

  * project一覧を再生成（出現した+projectを反映）

  * filter_tasksで表示対象を再計算

  * 中央一覧とステータスを更新

#### 2.1.4 フィルタルール

* projectフィルタ

  * all: 全件

  * +xxx: Task.projects に +xxx を含むもの

* 検索フィルタ

  * Task.raw_line に search_query が含まれるもの（大文字小文字はMVPではそのまま）

  * search_queryが空なら検索フィルタは無効

### 2.2 キー操作（MainScreen）

#### 2.2.1 キー定義の方針（優先順位と衝突回避）

* 基本は MainScreen にキーを定義する

* AddTaskModal / EditTaskModal は ModalScreen とし、下の画面のキー処理が走らない前提にする

* 検索入力（Input）にフォーカスがある間は、文字入力とカーソル移動を優先する

  * priority binding は使わない（矢印、Backspace、Enter などを潰しやすいため）

* 検索の終了（Esc）は SearchInput 側で処理し、フォーカス解除と search_query のクリアを行う

* 編集開始は e に寄せ、Enter はモーダル側の確定に使う

#### 2.2.2 キーバインド一覧

* ↑/↓

  * フォーカス中の一覧で選択を移動

* Tab

  * focus_target を sidebar と tasks で切替

* /

  * 検索入力を表示して focus_target を search にする

* Esc

  * focus_target が search のとき

    * 検索入力を閉じる

    * search_query を空にする

    * focus_target を tasks に戻す

  * モーダル表示中はキャンセル

* a

  * AddTaskModalを開く

* e

  * 選択中タスクがある場合、EditTaskModalを開く

* Space

  * 選択中タスクがある場合、完了トグル

* s

  * dirtyなら保存

* q

  * dirtyなら保存して終了

  * dirtyでなければそのまま終了

### 2.3 AddTaskModal（追加）

#### 2.3.1 入力

* body（必須）

#### 2.3.2 追加ルール

* 選択中projectが all 以外なら、本文末尾に +project を付与

* すでに本文中に同じ +project が含まれている場合は付与しない

* 生成する1行（raw_line）

  * 行頭や日付などは自動付与しない

  * 本文と +project のみで作る

#### 2.3.3 反映

* TodoStore.add_taskで末尾に追加
* tasks更新としてMainScreenへ戻す

### 2.4 EditTaskModal（編集）

#### 2.4.1 入力

* body（必須）

* is_done（チェック）

#### 2.4.2 更新ルール

* 本文編集

  * Task.bodyのみ差し替える

  * +project、@context、key:value は原則保持する

  * raw_lineの再生成は update_body_in_raw_line で行う

* 完了／未完了

  * toggle_done_in_raw_line により行頭の x の付け外し

#### 2.4.3 反映

* TodoStore.update_taskで該当行を更新
* tasks更新としてMainScreenへ戻す
