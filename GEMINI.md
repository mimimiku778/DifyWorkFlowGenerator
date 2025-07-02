# オープンチャット分析システム

## 必須事項
- **現在時刻からデータ時点判定**：
  - growth_ranking_hourly/daily: 現在時刻の分<30分→前時間30分頃、≥30分→現在時間30分頃
  - growth_ranking_weekly: 当日0:00
- Pythonエラー時は`pip install`

## API接続
```python
import requests, urllib.parse
from datetime import datetime
current_datetime = datetime.now()
url = "http://localhost:10000/?query=" + urllib.parse.quote("SELECT ...")
response = requests.get(url, timeout=10)
```

## スキーマ
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
  `recorded_at` datetime NOT NULL COMMENT '記録日時（line_official_ranking_total_countと紐づく）',
  `record_date` date NOT NULL DEFAULT '2024-01-01' COMMENT '記録日（１つのオープンチャットにつき同じ日付は１件。そのユニークキー用のカラム。）',
  PRIMARY KEY (`record_id`)
) COMMENT='LINEオープンチャット公式サイトの「ランキング」履歴（カテゴリ別・全体、1日1件、中央値保存）';

CREATE TABLE `line_official_activity_trending_history` (
  `record_id` int(11) NOT NULL AUTO_INCREMENT COMMENT 'レコードID',
  `openchat_id` int(11) NOT NULL COMMENT 'オプチャグラフでオープンチャットを識別するための主キー（openchat_masterと紐づく）',
  `category_id` int(11) NOT NULL COMMENT 'カテゴリID（0=すべて、1以上=各カテゴリ）',
  `activity_trending_position` int(11) NOT NULL COMMENT 'その日のLINE公式「急上昇」順位（最大値、何件中何位かはline_official_ranking_total_countで確認）',
  `recorded_at` datetime NOT NULL COMMENT '記録日時（line_official_ranking_total_countと紐づく）',
  `record_date` date NOT NULL DEFAULT '2024-01-01' COMMENT '記録日（１つのオープンチャットにつき同じ日付は１件。そのユニークキー用のカラム。）',
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

### 人数変化ランキング（growth_ranking_*）
オプチャグラフが収集した人数統計のみに基づくランキング

### LINE公式活動ランキング（line_official_activity_*）
LINE側の複雑なアルゴリズム（活動量等を含む）によるランキング履歴

### データベース仕様
- **エンジン**: MariaDB 10.5
- **API接続**: `http://localhost:10000/?query=` + urllib.parse.quote(SQL)

### 注意事項
- MariaDB 10.5構文を厳守
- 大量テキストはLEFT(column, 100)で制限

## ユーザー要求分析・ヒアリング

### 基本方針
曖昧な要求には段階的ヒアリングで要件確定

### ヒアリング項目
1. **目的**: 何を知りたいか、結果をどう活用するか
2. **期間**: hourly/daily/weekly、具体的日付範囲
3. **対象**: カテゴリ、ランキング種別（成長 or LINE公式）、順位範囲
4. **条件**: メンバー数・成長率の最小値、除外条件
5. **出力**: 一覧表/サマリー/グラフ、表示項目、ソート順

### 実行手順
要求分析→段階的質問→要件確定→SQL作成・実行

### サンプル
```
ユーザー: 「人気のオープンチャットを調べて」
アシスタント: 「以下を確認させてください：
1. 人気の定義: 成長率？LINE公式活動度？
2. 期間: 時間/日/週ランキング？
3. カテゴリ: 全て？特定カテゴリ？
4. 表示: 上位何位？どんな情報？」
```

## 高度な分析手法

### 形態素解析とトレンド分析
オープンチャットを分析するときは適宜Mecabで形態素分析を行い、適宜キーワードをWEBで検索して意味を調べたり、トレンドの傾向やどのようなものかを明らかにしながら分析を進める

### 露出制限の検出と分析
ユーザーがよく気にすることとして、「検索落ち・検索非表示・検索できない・ランキングに表示されない」などの話題があり、これはLINEオープンチャット公式サイトやLINEアプリで特定のオープンチャットをキーワード検索しても結果に表示されない露出制限状態となっているという意味です。

#### 露出制限の検出方法
データベースから露出制限の傾向を調べる方法として、`line_official_activity_ranking_history`テーブルに特定のオープンチャットについて過去のデータがあるがある時点からデータがない場合、露出制限状態となっている可能性があります。

ただし、他に露出制限と同様の状態になる条件として以下があります：
- 単なるランキング圏外
- ユーザーがオープンチャットの情報を最後に更新してから24時間以内

#### 精度の高い露出制限検出
露出制限の傾向を調べるときは、`line_official_ranking_total_count`と結合して、`line_official_ranking_total_count.activity_ranking_total_count`の50％より低い`activity_ranking_position`にしぼり、更に直近数日以上ランキングが途切れている`openchat_id`を抽出すると精度が高まります。

つまり、最後のランキング順位が上位50％だが、数日間以上ランキングに掲載がないルームはLINE運営による露出制限の可能性が高いです。

#### 露出制限の理由
露出制限となる最大の理由はLINEオープンチャットのガイドラインにグレーゾーンのレベルで抵触している場合です。完全に抵触している場合は露出制限より前にLINE運営によるオープンチャットの削除がなされます。

ガイドラインURL: https://openchat-jp.line.me/other/guideline
Admins' Hub - よくある質問: https://openchat-jp.line.me/admin/questions

必要に応じてガイドラインを参照して分析の情報として利用しましょう。ただし、露出制限と異なるコンテキストで露出制限やガイドラインの話題を無理に結びつける必要はありません。