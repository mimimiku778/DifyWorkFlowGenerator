# Dify ワークフロージェネレーター

## 概要

Claude CodeまたはGemini CLIと直接会話してDifyワークフローのYAMLファイルを生成できるツールです。複雑なセットアップは不要で、自然言語でワークフローの要件を伝えるだけで、Difyに直接インポート可能なワークフローが生成されます。

## 使用方法

Claude CodeまたはGemini CLIに以下のように話しかけてください：

```
Difyワークフローを生成してください。要件は以下の通りです：
- 目的：[ワークフローの目的]
- 処理フロー：
  1. [ステップ1の説明]
  2. [ステップ2の説明]
  3. [ステップ3の説明]
```

### 例：料理レシピ検索ワークフロー

```
Difyワークフローを生成してください。要件は以下の通りです：
- 目的：料理のレシピを調べて記事にする
- 処理フロー：
  1. 料理名をユーザーから入力
  2. インターネットで検索して3つのURLを取得
  3. 各URLからレシピ情報を並列取得
  4. 情報を統合して記事形式で出力
```

## ファイル構成

- `CLAUDE.md` - Claude Code用の詳細な生成アルゴリズム
- `GEMINI.md` - Gemini CLI用の詳細な生成アルゴリズム
- `workflow_generator_prompt.yml` - ワークフロー生成プロンプト仕様
- `README.md` - このファイル

## サポートされるノードタイプ

**基本ノード**：開始、LLM、終了  
**分岐・制御**：質問分類器、IF/ELSE分岐、Variable Aggregator  
**データ処理**：知識検索、HTTPリクエスト、JSONパース、コード実行  
**テキスト・メディア処理**：Document Extractor、Template Transform、Parameter Extractor、YouTubeトランスクリプト、JinaReader、TavilySearch

詳細な仕様は`workflow_generator_prompt.yml`を参照してください。


## ライセンス

MITライセンス

## 貢献について

プルリクエストや課題の報告を歓迎します。

## 関連リンク

- [Dify公式サイト](https://dify.ai)
- [Difyドキュメント](https://docs.dify.ai) 