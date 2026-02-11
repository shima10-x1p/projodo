# AGENTS.md

## プロジェクト概要
projodo という、TUIベースのToDoリスト管理アプリケーションの開発を目的としたプロジェクトです。

## 技術スタック
- 言語: Python 3.14
- フレームワーク: Textual 7.5.0 (TUIフレームワーク)
- ToDoリスト管理: todo.txt フォーマット
- パッケージ管理: uv
- リンター・フォーマッター: ruff

## ディレクトリ構成
```
projodo/
├── src/                        # ソースコード
│   ├── ui/                     # UI関連のコード
│   │   └── screens/            # 画面定義
│   │       ├── main.py         # メイン画面
│   │       └── ...             # その他の画面
│   ├── core/                   # コアロジック
│   │   ├── application/        # アプリケーションサービス
│   │   └── domain/             # ドメインモデル
│   ├── ports/                  # ポート（インターフェース）
│   ├── adapters/               # アダプター（実装）
│   ├── app.py                  # アプリケーションのエントリポイント
│   └── __init__.py             # パッケージ初期化ファイル
├── tests/                      # テストコード
│   ├── core/                   # コアロジックのテスト
│   └── adapters/               # アダプターのテスト
├── pyproject.toml              # プロジェクト設定ファイル
├── AGENTS.md                   # エージェントに関するドキュメント
└── README.md                   # プロジェクト概要ドキュメント
```

## 設計方針
- このプロジェクトは、ポート・アダプターアーキテクチャ（Hexagonal Architecture）に基づいて設計されている。
- 各コンポーネントは疎結合で設計されており、テスト容易性と拡張性を重視している。
- UIはTextualフレームワークを使用して構築され、ユーザー体験を向上させる。

## 開発ガイドライン
- コーディング規約としてPEP 8を遵守する。
  - 詳細は `.github/instructions/python.instructions.md` を参照。
- docstringはGoogleスタイルを採用する。
- 型ヒントは引数・戻り値ともに必須とする。
- テストコードはpytestを使用し、主要な機能についてはユニットテストを実装する。

## 開発コマンド
- `uv add <package>`: 新しいパッケージの追加
- `uv add --dev <package>`: 開発用パッケージの追加(テストやリンター用)
- `uv run <module>`: 指定したモジュールを実行
- `uv run pytest`: テストの実行
- `uv run ruff check .`: コードの静的解析
- `uv run ruff format .`: コードの自動修正

## 自己改善
- AGENTS.md ファイルを定期的に見直し、プロジェクトの進行に合わせて更新する。
- 開発中に発生した問題や改善点を記録し、次回の開発に活かす。

## 参考資料
### 設計書
- [01_projodo_TUI_ToDoアプリ_要件定義](docs/document/01_RD/01_projodo_TUI_ToDoアプリ_要件定義.md)
- [02_projodo_TUI_ToDoアプリ_全体設計](docs/document/02_DD/01_projodo_TUI_ToDoアプリ_全体設計.md)
- [03_projodo_TUI_ToDoアプリ_画面・操作・状態設計](docs/document/02_DD/02_projoto_TUI_ToDoアプリ_画面・操作・状態設計.md)

### ライブラリ
- [Textual](https://textual.textualize.io/): TUIフレームワーク
- [todo.txt](https://github.com/todotxt/todo.txt): ToDoリスト管理フォーマット