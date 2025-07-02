# Claude Code オープンチャット分析システム

## 概要
Claude Codeは、LINEオープンチャット（オプチャグラフ）のデータを高度に分析し、ユーザーの要求に応じた洞察を提供します。

## 分析フロー

### 1. 要件分析と掘り下げ
ユーザーの要求を受けたら、まず以下を明確化：
- 分析の目的（トレンド把握、特定チャット調査、カテゴリ分析など）
- 対象期間（直近、過去3ヶ月、全期間など）
- 重視する指標（成長率、絶対数、ランキングなど）
- **⚠️ 必須：現在時刻を取得し、growth_ranking_*テーブルのデータ時点を判定・明示**
  - 現在時刻の分が30分未満なら前の時間の30分頃のデータ
  - 現在時刻の分が30分以上なら現在時間の30分頃のデータ
  - growth_ranking_weeklyは当日0:00のデータ

### 2. データ取得と分析
- 適切なSQLクエリを構築
- 必要に応じて複数回のクエリで段階的に分析
- 大量データは一時ファイルに保存して処理

### 3. 結果の解釈と出力
- 専門用語を避けた分かりやすい説明
- グラフィカルな表現（テーブル、順位表示など）
- 具体的な数値とその意味の解説

## 主要分析パターン

### ランキング分析（2種類の性質の異なるランキング）

#### 1. 人数変化ランキング（growth_ranking_*）
オプチャグラフが収集した人数統計のみに基づくランキング：
```sql
-- 1時間の急上昇（毎時更新）
SELECT grh.ranking_position, grh.member_increase_count, grh.growth_rate_percent,
       om.display_name, om.current_member_count
FROM growth_ranking_hourly grh
JOIN openchat_master om ON grh.openchat_id = om.openchat_id
ORDER BY grh.ranking_position
LIMIT 20;

-- 24時間の成長ランキング（毎時更新）
SELECT grd.*, om.display_name
FROM growth_ranking_daily grd
JOIN openchat_master om ON grd.openchat_id = om.openchat_id;

-- 週間成長ランキング（日次更新）
SELECT grw.*, om.display_name
FROM growth_ranking_weekly grw
JOIN openchat_master om ON grw.openchat_id = om.openchat_id;
```

#### 2. LINE公式活動ランキング（line_official_activity_*）
LINE側の複雑なアルゴリズム（活動量等を含む）によるランキング履歴：
```sql
-- LINE公式「ランキング」履歴
SELECT loar.*, om.display_name, c.category_name
FROM line_official_activity_ranking_history loar
JOIN openchat_master om ON loar.openchat_id = om.openchat_id
LEFT JOIN categories c ON loar.category_id = c.category_id
WHERE loar.openchat_id = ?
ORDER BY loar.record_date DESC;

-- LINE公式「急上昇」履歴
SELECT loat.*, om.display_name
FROM line_official_activity_trending_history loat
JOIN openchat_master om ON loat.openchat_id = om.openchat_id
WHERE loat.record_date >= DATE_SUB(CURDATE(), INTERVAL 7 DAY);
```

### 個別チャット詳細分析
```sql
-- メンバー数推移（日次統計）
SELECT record_id, member_count, statistics_date,
       LAG(member_count) OVER (ORDER BY statistics_date) as prev_count,
       member_count - LAG(member_count) OVER (ORDER BY statistics_date) as daily_change
FROM daily_member_statistics 
WHERE openchat_id = ? 
ORDER BY statistics_date DESC
LIMIT 30;
```

### カテゴリ別統計
```sql
-- カテゴリ別の規模と特徴
SELECT c.category_id, c.category_name, 
       COUNT(*) as total_chats,
       AVG(om.current_member_count) as avg_members,
       MAX(om.current_member_count) as max_members,
       SUM(CASE WHEN om.verification_badge IS NOT NULL THEN 1 ELSE 0 END) as verified_count
FROM openchat_master om
JOIN categories c ON om.category_id = c.category_id
GROUP BY c.category_id
ORDER BY total_chats DESC;
```

## 高度な分析手法

### 時系列分析
- **必須：現在日時を基準に分析期間を設定**
- 成長率計算：LAG()関数で前期比較
- 移動平均：複数期間の平均で傾向把握
- 異常値検出：標準偏差を使った外れ値特定
- 期間指定例：`WHERE statistics_date >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)`

### 相関分析
- カテゴリと成長率の関係
- 認証バッジの影響度
- 参加方法別の特徴

### 予測分析
- 過去データからの成長予測
- 季節性・周期性の検出
- トレンドの持続性評価

## 環境セットアップ

Pythonコード実行時にモジュールエラーが発生した場合は、pipでインストールしてください：
```bash
# 例: ModuleNotFoundError: No module named 'requests'
pip install requests
```

## 技術仕様

### API接続
```python
import requests
import urllib.parse
from datetime import datetime

current_datetime = datetime.now()
api_url = "http://localhost:10000/?query="
query = "SELECT ..."
url = api_url + urllib.parse.quote(query)
response = requests.get(url, timeout=10)
```

### growth_ranking_*テーブルのデータ時点判定（重要）
- **growth_ranking_hourly/daily**: 毎時30分頃更新
  - 現在時刻の分 < 30分 → 前の時間の30分頃のデータ
  - 現在時刻の分 ≥ 30分 → 現在時間の30分頃のデータ
- **growth_ranking_weekly**: 毎日0:00更新（当日0:00のデータ）
- **必須**: 分析開始時に現在時刻を確認し、ランキングデータの時点を明示すること

### エラーハンドリング
- タイムアウト設定（10秒）
- 大量データ時の分割取得
- SQLエラーの適切な処理

## データベーススキーマ

### 完全スキーマ定義

```sql
CREATE TABLE `categories` (
  `category_id` int(11) NOT NULL COMMENT 'カテゴリID',
  `category_name` varchar(30) NOT NULL COMMENT 'カテゴリ名',
  PRIMARY KEY (`category_id`)
) COMMENT='カテゴリマスタテーブル';

CREATE TABLE `daily_member_statistics` (
  `record_id` int(11) NOT NULL COMMENT 'レコードID',
  `openchat_id` int(11) NOT NULL COMMENT 'オプチャグラフでオープンチャットを識別するための主キー（openchat_masterと紐づく）',
  `member_count` int(11) NOT NULL COMMENT 'メンバー数',
  `statistics_date` date NOT NULL COMMENT '統計日',
  PRIMARY KEY (`record_id`),
  KEY `idx_open_chat_date` (`openchat_id`,`statistics_date`)
) COMMENT='オープンチャットのメンバー数統計（毎日1件、オープンチャットIDと日付でユニーク）';

CREATE TABLE `growth_ranking_daily` (
  `ranking_position` int(11) NOT NULL COMMENT '順位（1位、2位...）',
  `openchat_id` int(11) NOT NULL COMMENT 'オプチャグラフでオープンチャットを識別するための主キー（openchat_masterと紐づく）',
  `member_increase_count` int(11) NOT NULL COMMENT 'メンバー増加数',
  `growth_rate_percent` float NOT NULL COMMENT '成長率（%）',
  PRIMARY KEY (`ranking_position`)
) COMMENT='オプチャグラフが集計しているオープンチャットの成長ランキング（日次）';

CREATE TABLE `growth_ranking_hourly` (
  `ranking_position` int(11) NOT NULL COMMENT '順位（1位、2位...）',
  `openchat_id` int(11) NOT NULL COMMENT 'オプチャグラフでオープンチャットを識別するための主キー（openchat_masterと紐づく）',
  `member_increase_count` int(11) NOT NULL COMMENT 'メンバー増加数',
  `growth_rate_percent` float NOT NULL COMMENT '成長率（%）',
  PRIMARY KEY (`ranking_position`)
) COMMENT='オプチャグラフが集計しているオープンチャットの成長ランキング（時間単位）';

CREATE TABLE `growth_ranking_weekly` (
  `ranking_position` int(11) NOT NULL COMMENT '順位（1位、2位...）',
  `openchat_id` int(11) NOT NULL COMMENT 'オプチャグラフでオープンチャットを識別するための主キー（openchat_masterと紐づく）',
  `member_increase_count` int(11) NOT NULL COMMENT 'メンバー増加数',
  `growth_rate_percent` float NOT NULL COMMENT '成長率（%）',
  PRIMARY KEY (`ranking_position`)
) COMMENT='オプチャグラフが集計しているオープンチャットの成長ランキング（週間）';

CREATE TABLE `line_official_activity_ranking_history` (
  `record_id` int(11) NOT NULL AUTO_INCREMENT COMMENT 'レコードID',
  `openchat_id` int(11) NOT NULL COMMENT 'オプチャグラフでオープンチャットを識別するための主キー（openchat_masterと紐づく）',
  `category_id` int(11) NOT NULL COMMENT 'カテゴリID（0=すべて、1以上=各カテゴリ）',
  `activity_ranking_position` int(11) NOT NULL COMMENT 'その日のLINE公式「ランキング」順位（中央値、何件中何位かはline_official_ranking_total_countで確認）',
  `recorded_at` datetime NOT NULL COMMENT '記録日時',
  `record_date` date NOT NULL DEFAULT '2024-01-01' COMMENT '記録日',
  PRIMARY KEY (`record_id`)
) COMMENT='LINEオープンチャット公式サイトの「ランキング」履歴（カテゴリ別・全体、1日1件、中央値保存）';

CREATE TABLE `line_official_activity_trending_history` (
  `record_id` int(11) NOT NULL AUTO_INCREMENT COMMENT 'レコードID',
  `openchat_id` int(11) NOT NULL COMMENT 'オプチャグラフでオープンチャットを識別するための主キー（openchat_masterと紐づく）',
  `category_id` int(11) NOT NULL COMMENT 'カテゴリID（0=すべて、1以上=各カテゴリ）',
  `activity_trending_position` int(11) NOT NULL COMMENT 'その日のLINE公式「急上昇」順位（最大値、何件中何位かはline_official_ranking_total_countで確認）',
  `recorded_at` datetime NOT NULL COMMENT '記録日時',
  `record_date` date NOT NULL DEFAULT '2024-01-01' COMMENT '記録日',
  PRIMARY KEY (`record_id`)
) COMMENT='LINEオープンチャット公式サイトの「急上昇」履歴（カテゴリ別・全体、1日1件、最大値保存）';

CREATE TABLE `line_official_ranking_total_count` (
  `record_id` int(11) NOT NULL AUTO_INCREMENT COMMENT 'レコードID',
  `activity_trending_total_count` int(11) NOT NULL COMMENT 'その時間のLINE公式「急上昇」総件数（何件中何位かを知るために使用）',
  `activity_ranking_total_count` int(11) NOT NULL COMMENT 'その時間のLINE公式「ランキング」総件数（何件中何位かを知るために使用）',
  `recorded_at` datetime NOT NULL COMMENT '記録日時（毎時間更新）',
  `category_id` int(11) NOT NULL COMMENT 'カテゴリID（0=すべて、1以上=各カテゴリ）',
  PRIMARY KEY (`record_id`)
) COMMENT='LINEオープンチャット公式サイトの全ランキング総件数履歴（「ランキング」・「急上昇」、カテゴリ別・全体、毎時間記録）';

CREATE TABLE `openchat_master` (
  `openchat_id` int(11) NOT NULL AUTO_INCREMENT COMMENT 'オプチャグラフでオープンチャットを識別するための主キー',
  `display_name` text NOT NULL COMMENT 'オープンチャット名',
  `profile_image_url` varchar(128) NOT NULL COMMENT 'オープンチャットのメイン画像',
  `description` text NOT NULL COMMENT '説明',
  `current_member_count` int(11) NOT NULL COMMENT '現在のメンバー数',
  `first_seen_at` timestamp NOT NULL DEFAULT current_timestamp() COMMENT '初回取得日時',
  `last_updated_at` timestamp NOT NULL DEFAULT current_timestamp() COMMENT '最終更新日時',
  `line_internal_id` varchar(255) DEFAULT NULL COMMENT 'LINE内部ID（emid）',
  `established_at` int(11) DEFAULT NULL COMMENT '開設日時（Unixタイムスタンプ）',
  `invitation_url` text DEFAULT NULL COMMENT 'オープンチャット招待用URL（参加リンク）',
  `verification_badge` varchar(20) DEFAULT NULL COMMENT '認証バッジ（NULL:なし, 公式認証, スペシャル）',
  `join_method` varchar(30) NOT NULL COMMENT '参加方法（全体公開, 参加承認制, 参加コード入力制）',
  `category_id` int(11) DEFAULT NULL COMMENT 'カテゴリID（categoriesと紐づく）',
  PRIMARY KEY (`openchat_id`)
) COMMENT='オプチャグラフがLINEオープンチャット公式サイトから収集したオープンチャットマスターテーブル';
```

### データベース仕様
- **エンジン**: MariaDB 10.5
- **文字コード**: UTF-8
- **API接続**: `http://localhost:10000/?query=` + urllib.parse.quote(SQL)

### 時系列データの扱い
- **growth_ranking_*テーブル**（3種）：人数変化のみに基づく最新ランキング（履歴なし、日時カラムなし）
  - hourly: 1時間の変化（**毎時30分頃更新**）
  - daily: 24時間の変化（**毎時30分頃更新**）  
  - weekly: 1週間の変化（**毎日0:00更新**）
  - ⚠️ これらのテーブルには更新日時カラムがないため、現在時刻から更新タイミングを推定する必要がある
- **line_official_activity_*テーブル**（2種）：LINE公式の活動量ベースランキング履歴
  - ranking_history: 「ランキング」の日次履歴（record_date, recorded_atあり）
  - trending_history: 「急上昇」の日次履歴（record_date, recorded_atあり）
- **daily_member_statistics**：個別チャットの人数推移を日次で完全記録（statistics_dateあり）

### 注意事項
- MariaDB 10.5構文を厳守
- 大量テキストはLEFT(column, 30)で制限
- インデックスを活用した高速クエリ
- established_atはUnixタイムスタンプ形式（FROM_UNIXTIME()で変換）

## 出力形式

### 基本構成
1. **要約** - 3行以内で結論
2. **詳細データ** - 表形式で主要指標
3. **トレンド解説** - 変化の意味を説明
4. **推奨アクション** - 次の分析提案

### リンク生成
個別オープンチャット詳細：
`https://openchat-review.me/oc/{openchat_id}`