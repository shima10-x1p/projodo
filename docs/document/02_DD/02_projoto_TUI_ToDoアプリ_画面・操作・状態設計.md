# Projodo TUI MVP 画面・操作・状態設計

## 1. 目的

MainScreenを中心に、project選択、検索、追加、編集、完了トグル、保存、終了をキーボード中心で完結させる。

## 2. 画面一覧

* MainScreen

  * 左：project一覧

  * 中央：タスク一覧

  * 下：ステータス行＋フッター＋検索入力（必要時のみ表示）

* AddTaskModal（ModalScreen）

  * タスク追加

* EditTaskModal（ModalScreen）

  * タスク編集

## 3. MainScreen 画面構成

### 3.1 レイアウト

* 上

  * 任意でHeader（アプリ名、ファイル名）

* 本体

  * 左ペイン：ProjectList（ListView）

  * 右ペイン：TaskList（ListView）

* 下

  * StatusLine（Labelなど）

  * Footer（Textual標準）

  * SearchInput（Input。通常は非表示）

### 3.2 ウィジェット構成案

* ProjectList

  * ListViewで project を縦に並べる

  * 先頭に疑似projectの all

  * 以降に +project を表示

* TaskList

  * ListViewでタスクを縦に並べる

  * 各行の表示は Task の build_line（保存時の規定順）をベースにする

  * 完了タスクは先頭の x で判別できる

* StatusLine

  * 表示例

    * project: +k8s-mito | query: metallb | shown: 12/87 | dirty

  * dirty が false のときは dirty 表記を出さない

* SearchInput

  * / で表示してフォーカス

  * Esc で閉じて search_query をクリア

## 4. MainScreen 状態（State）

### 4.1 画面状態

* selected_project: str

  * all または +project

* search_query: str

  * 検索文字列

* focus_target: enum

  * sidebar / tasks / search

* visible_tasks: list[Task]

  * list_tasks(selected_project, search_query) の結果

* selected_task_ref: TaskRef | None

  * visible_tasks の選択行に対応する Task.ref

### 4.2 Store状態（参照のみ）

* todo: TodoUseCase
* dirty: bool

### 4.3 状態の持ち方（TextualのReactive前提）

* selected_project と search_query は reactive として管理する

* watch_selected_project / watch_search_query で visible_tasks を再計算し、リスト表示を更新する

* dirty は TodoUseCase 側の状態を参照して表示する

## 5. MainScreen 操作（キー・フォーカス・イベント）

### 5.1 キー定義方針

* キーは基本的に MainScreen に定義する

* AddTaskModal / EditTaskModal は ModalScreen とし、下の画面のキー処理が走らない前提にする

* SearchInput にフォーカスがある間は入力を優先する

  * priority binding は使わない

### 5.2 キーバインド

* Tab

  * sidebar と tasks を切り替える

* ↑/↓

  * フォーカス中のListViewで選択移動

* /

  * SearchInput を表示してフォーカス

* Esc

  * focus_target が search のとき

    * SearchInput を非表示

    * search_query を空にする

    * focus_target を tasks に戻す

* a

  * AddTaskModal を開く

* e

  * 選択中タスクがあるとき EditTaskModal を開く

* Space

  * 選択中タスクがあるとき todo.toggle_done(ref)

* s

  * dirty のとき todo.save()

* q

  * dirty のとき todo.save() して終了

  * dirty でなければそのまま終了

### 5.3 ProjectList 操作

* 選択変更（ハイライト移動）

  * 選択確定はListViewの選択イベントで行う

  * selected_project を更新し、watchでvisible_tasksを更新する

### 5.4 TaskList 操作

* 選択変更

  * 選択中の visible_tasks[n].ref を selected_task_ref に反映する

* Space

  * selected_task_ref を使って完了トグル

* e

  * selected_task_ref を渡して編集モーダルを開く

### 5.5 SearchInput 操作

* 表示

  * / で SearchInput を表示し、focus_target を search にする

* 入力

  * 入力内容を search_query に反映

  * 反映は on_change か、一定の入力確定（Enter）で行う

* 終了

  * Esc で SearchInput を閉じ、search_query を空にする

## 6. AddTaskModal（追加）

### 6.1 入力

* body（必須）

### 6.2 生成ルール

* selected_project が all 以外なら、本文末尾に +project を付与

* すでに本文中に同じ +project が含まれている場合は付与しない

* completion_date / creation_date / priority は付与しない

### 6.3 操作

* Enter

  * todo.add_task(body, selected_project)

  * 成功したら閉じる

* Esc

  * 何もせず閉じる

## 7. EditTaskModal（編集）

### 7.1 入力

* body（必須）

* is_done（チェック）

### 7.2 更新ルール

* todo.update_task(ref, body, is_done)

* 保存時の行の再構成は TodoTxtParser.build_line の規定順で行う

* 未知トークンは保持しない

### 7.3 操作

* Enter

  * 更新して閉じる

* Esc

  * 破棄して閉じる

## 8. 表示更新のルール

* project変更

  * visible_tasks を再計算

  * TaskList を差し替え

  * TaskList の選択を先頭に寄せる

* 検索語変更

  * visible_tasks を再計算

  * TaskList を差し替え

* 追加／編集／完了トグル

  * TodoUseCase の結果が反映された tasks を元に

    * project一覧を再生成

    * visible_tasks を再計算

    * 両リストとStatusLineを更新

* 保存

  * dirty が false になったら StatusLine の dirty 表記を消す

## 9. エラーと境界条件

* todo.txt が存在しない

  * 空のタスクとして開始し、保存時に新規作成する

* 解析できない行

  * パースに失敗した場合は、その行をスキップする

* 選択対象が無い

  * TaskList が空のときは e / Space を無効

* 保存失敗

  * StatusLine に短いエラー表示を出し、dirty を維持する
