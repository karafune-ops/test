---
name: deal-next-action-reminder
description: |
  荒舩さんの保有商談（Salesforce）の「次回アクション日」を毎営業日クローリングし、
  期限超過・当日の案件をSlackの商談チャンネルにメンション付きでリマインドするスキル。
  Claude Code Routineで毎営業日朝に自動実行することを想定。
  「次回アクションのリマインド」「商談リマインド」「次回アクション日チェック」
  「アクション漏れ確認」「期限超過の商談」「今日のアクション」などのキーワードで必ず発動する。
  設計の背景・意思決定はリポジトリ直下の DESIGN.md を参照。
---

# 商談 次回アクション日リマインド

保有商談の `next_action_date__c` を毎営業日チェックし、**期限超過**・**本日**・**日付未設定**の
案件を、Salesforce連携が商談ごとに自動作成しているSlackチャンネルへメンション付きで投稿する。

## 設定パラメータ（変更はここだけ）

```
対象OwnerId       : 005GC00000kynm7YAA（荒舩 健太）
対象範囲          : IsClosed = false（全フェーズ）
メンション対象     : 商談Owner + サブ担当（subhandler__c）。重複は排除
督促頻度          : 期限超過は解消まで毎日
予告リマインド     : なし（翌営業日期限の予告は行わない）
棚卸し警告閾値     : 8日以上の超過に「⚠️要棚卸し」を付す
アクション内容上限  : 200文字
1回の最大投稿件数  : 15件（超過分はサマリーのみ。暴走時の安全弁）
```

## 状態シート（FFDEヘッドレスシート）

| 用途 | シート名 | id |
|---|---|---|
| チャンネルキャッシュ | 商談リマインド_チャンネルキャッシュ | `9f74806e-6d5f-452d-b8d8-5a22890b75c7` |
| 投稿ログ | 商談リマインド_投稿ログ | `c42b0a77-e2f3-488a-b71c-7588cd86b0f3` |

Routineは毎回新規セッションで動くため、状態はすべてこのシートに持つ。セッション内に記憶しない。

## メンバー → Slack User ID

`salesforce_user.SlackID__c` から解決する。頻出メンバーは以下（クエリ省略可）。

```
荒舩 健太  005GC00000kynm7YAA → U033MA255R6
檜垣 知也  005Q900000CmNZiIAN → U08DGSES1J9
木村 清二  005GC00000lFeifYAC → U083B7V29K7
```

上記以外のIDが出てきたら `SELECT Id, Name, SlackID__c FROM salesforce_user WHERE Id IN (...)` で解決する。
`SlackID__c` が空の場合はメンションせず氏名テキストのみ表示する（`<@>` の不正形式を防ぐ）。

---

## 実行手順

### STEP 1: 対象商談の取得（1クエリ）

`clickhouse-salesforce` スキルのルールに従い `FFDE:query_clickhouse` を使う。Salesforce MCPは使わない。

```sql
SELECT
  Id,
  Name,
  StageName,
  toDate(next_action_date__c)                          AS na_date,
  replaceAll(replaceAll(substringUTF8(ifNull(next_action_content__c,''),1,200), '\n', ' '), '\r', ' ') AS na_content,
  ifNull(ACV__c, 0)                                    AS acv,
  ifNull(subhandler__c, '')                            AS subhandler,
  dateDiff('day', toDate(next_action_date__c), toDate(now('Asia/Tokyo'))) AS days_overdue
FROM salesforce_opportunity
WHERE IsClosed = false
  AND OwnerId = '005GC00000kynm7YAA'
  AND (next_action_date__c IS NULL
       OR next_action_date__c < toDate(now('Asia/Tokyo')) + 1)
ORDER BY na_date
LIMIT 200
```

**⚠️ `today()` を絶対に使うな。** ClickHouseサーバはUTCで動いており、`now()` はUTCを返す。
朝の実行（00:00〜09:00 JST）では `today()` が前日を指すため、**当日案件を丸ごと取り逃す**。
必ず `toDate(now('Asia/Tokyo'))` を使う。

その他のクエリ規約（`clickhouse-salesforce` スキル準拠）：
- `IsDeleted` はカラムが存在しない。WHERE句に書かない
- `ACV__c` はキャスト不要（`toFloat64OrNull()` を付けるとCode 43エラー）
- 一覧系はLIMIT必須

### STEP 2: 仕分け

| バケット | 条件 | 表記 |
|---|---|---|
| `overdue` | `days_overdue >= 1` | 🔴 期限超過（8日以上は「⚠️要棚卸し」を付す） |
| `today` | `days_overdue = 0` | 🟡 本日 |
| `nodate` | `na_date` が NULL | ⚪️ 日付未設定 |

対象が0件なら**Slack投稿を一切行わず**、「本日対象なし」とだけ報告して終了する。

### STEP 3: 商談 → Slackチャンネルの解決

1. チャンネルキャッシュシート（A2:F200）を読む
2. 対象商談IDがヒットしたら、その `channel_id` をそのまま使う（`status=ok` のみ）
   - `status=ext_skip` → 投稿せずスキップ
   - `status=notfound` → `resolved_at` が7日以上前なら再解決を試みる。7日以内なら再検索せずスキップ
3. キャッシュミスの商談のみ `slack_search_channels` で検索する
   - 検索語は商談名から作る。商談名は `UC_One_企業名_テーマ` 形式なので、`_` で分割して
     **「企業名 テーマ」**の形で検索する（例：`富士フイルムメディカル Sales Portal`）
   - 検索結果のチャンネル名は `ZC:<channel_id>:<商談名>` 形式で返る。**実チャンネルIDは
     permalink（`https://ninout.slack.com/archives/XXXX`）から取る**
   - 1件ヒット かつ 商談名が一致 → `status=ok` でキャッシュに追記
   - 0件 → `status=notfound` でキャッシュに追記し、サマリーに列挙
   - 複数ヒット → 商談名完全一致を優先。決まらなければ `notfound` 扱い

**推測で別チャンネルへ投稿してはならない。** 顧客も見得るチャンネルへの誤爆が
この仕組みで最悪の事故。特定できなければスキップしてサマリーに載せる。

**外部共有チャンネルのガード**：`#ext-` 始まり、または Slack Connect（外部共有）の
チャンネルは投稿不可かつ社内督促が顧客に見えるため、`status=ext_skip` として記録し投稿しない。
判定は初回解決時のみ行う。

### STEP 4: 重複投稿チェック

投稿ログシート（A2:F1000）を読み、**同一の `run_date`（JST当日）× `opportunity_id` で
`status=ok` の行が既にある商談はスキップ**する。Routineの再実行や手動実行で、同じ督促を
チャンネルに二重投稿するのを防ぐ。

### STEP 5: 商談チャンネルへ投稿

`slack_send_message` で **1商談1メッセージ**。メンションは商談Owner＋サブ担当（同一人物なら1つに集約）。

期限超過：

```
<@U033MA255R6> <@U083B7V29K7>
🔴 次回アクション日が 8日 超過しています ⚠️要棚卸し

*商談*：UC_One_株式会社アドバンテッジリスクマネジメント_商談記録の高度化による営業イネーブルメント支援
*フェーズ*：P2 課題の特定 ／ *ACV*：¥1,200,000
*次回アクション日*：2026-07-21（8日超過）
*次回アクション内容*：ゴール：商談記録の主要な分析担当とのコネクト→価値合意 アクション： 先方に社内展開情報を共有

次回アクション日の更新、または実施済みなら日付の巻き直しをお願いします。
<https://creativesurvey.lightning.force.com/lightning/r/Opportunity/006Q900001mcKUcIAM/view|Salesforceで開く>
```

本日：1行目の見出しを `🟡 本日が次回アクション日です` に差し替え、超過日数の記載を省く。
日付未設定：見出しを `⚪️ 次回アクション日が未設定です` にし、日付行を省いて入力を促す。

書式ルール：
- ACVは3桁カンマ区切り（`¥1,200,000`）
- `next_action_content__c` は200文字で切り、改行はスペースに正規化（STEP1のSQLで対応済み）
- **全角スペースは半角スペース2つに置換**（Slack投稿エラーの回避）
- 1メッセージ5,000文字上限。1商談1メッセージなら超えない
- 8日以上の超過のみ「⚠️要棚卸し」を付す

1件の投稿失敗で全体を止めない。失敗分は `status=failed` で記録し、残りの投稿を続行する。

### STEP 6: 状態シートの更新

- チャンネルキャッシュ：新規解決分を追記（既存行は上書きしない）
- 投稿ログ：投稿した商談ごとに1行追記（`run_date` / `opportunity_id` / `bucket` /
  `channel_id` / `message_ts` / `status`）

追記先の行番号は `headless_sheet_read` で最終行を確認してから決める（既存行の上書き防止）。

### STEP 7: 自分宛サマリー

`slack_send_message` の `channel_id` に `U033MA255R6` を指定してDMで送る。

```
📋 次回アクション リマインド（2026-07-29）

🔴 期限超過 4件
 ・[8日超過] UC_One_株式会社アドバンテッジリスクマネジメント_商談記録の高度化…（P2 / ¥1,200,000）
 ・[5日超過] ...
🟡 本日 1件
 ・UC_One_日本3Dプリンター株式会社_MCP（P3 / ¥360,000）
⚪️ 日付未設定 0件

投稿：5チャンネル成功 / 0件スキップ
※ Salesforce同期は約1時間前時点のデータです
```

チャンネル未特定・外部共有スキップ・投稿失敗があれば、商談名と理由を列挙する。

### STEP 8: 完了報告

チャット（Routineの場合は完了通知）に、対象件数・投稿件数・スキップ内容を1〜3行で報告する。

---

## 通知経路についての注意

Slack MCPは**荒舩さん本人のトークンで投稿する**（投稿者＝荒舩さん）。Slackは自分の投稿では
自分に通知を出さないため、**商談チャンネル投稿の自分宛メンションは通知が鳴らない**。

役割分担：

| 経路 | 役割 |
|---|---|
| 商談チャンネル投稿 | 記録と、サブ担当への可視化（サブ担当には通知が飛ぶ） |
| Routine完了プッシュ通知 | 荒舩さん本人への実質的なリマインド（スマホに届く） |
| 自分宛DMサマリー | Slack内で一覧を見返す用 |

## 異常系

| 事象 | 挙動 |
|---|---|
| ClickHouseがエラー | `clickhouse-salesforce` スキルのリトライ戦略に従う（1回目は即リトライ／2回連続でクエリ単純化／4回でユーザー報告）。取得失敗時は**Slack投稿を一切行わない**（部分投稿を作らない） |
| 対象0件 | Slack投稿なし。「本日対象なし」と報告のみ |
| チャンネル未特定 | その商談をスキップし、サマリーに列挙。推測で別チャンネルに投稿しない |
| 外部共有チャンネル | 投稿せずスキップ。`ext_skip` で記録しサマリーに理由付きで列挙 |
| 一部投稿失敗 | 残りは続行。失敗分を `failed` で記録しサマリーに列挙 |
| 同日重複実行 | 投稿ログで検出しスキップ。「既に投稿済み」と報告 |
| 対象が15件超 | 上位15件のみ投稿し、残りはサマリーのみ。**打ち切った件数を必ず明記する** |

## 注意点

- 対象は `OwnerId` ベース（荒舩さん保有分のみ）。`pipeline-coverage-daily` は `subhandler__c`
  ベースで集計しており**軸が違う**。混同しない
- スケジュール自体はClaude Code Routine側で管理する。このSKILL.mdは実行内容のみを定義する
- 商談チャンネルには他部署メンバーも参加している。`@here` や `@channel` は使わない
- Salesforceへの書き込みは一切行わない（読み取り専用）
