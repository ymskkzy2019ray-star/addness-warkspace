---
name: addness-daily-updates
description: 毎朝、Addnessの「今日の自分の更新」を回す。事務員さんが請求・実績管理フォルダに置いたカイポケ帳票から、KPIの月の値を記録する。
---

# 【毎朝】Addness 今日の自分の更新

あなたは や和 のPCで、誰も見ていない状態で動く。質問はせず、最後までやり切る。
読み書きは Addness のMCPツールだけで行う。推定値では絶対に埋めない。

## 0. 前提

- 帳票の置き場：`C:\Users\user\メイン作業場\projects\請求・実績管理\`
  （場所が違えば、`メイン作業場\projects\請求・実績管理` をPCの中で探す。見つからなければ 4 へ）
- 帳票の出し方：Addnessのナレッジ「毎月、分析用のカイポケの帳票を出して置く」(39e7bed0-db92-4b47-9754-2ec4947a839e)
- 各KPIの数え方の正本は、KPI定義の method（get_analytics_recipe）。この手順と食い違ったら method に従う。

## 1. 今日の更新を取る

1. `list_my_updates_today` を呼ぶ（next_cursor があれば取り切る）。返る `goal_id` が今日の「今日の自分の更新」ゴール。
2. そのゴールに `report_goal_agent_activity`（agent=claude_code, state=working）を送る。
3. 各行（kind=recipe）について、`get_analytics_recipe` で method と最新の点を読む。

## 2. 記録する月を決める

- 対象は「前の月」と、それより前で点が無い月（最新点の翌月から前の月まで）。
- 各月について、method に書かれたファイルがあるかを見る：
  - 月間売上(請求)額：`請求情報_サービス提供年月YYYYMM.xlsx`（`請求・実績管理\` 直下・`請求情報\請求実績\`・`カイポケ分析資料一覧\`）
  - 月間訪問件数／1人1日あたり訪問件数：`スタッフ別訪問件数\YYYYMM訪問件数スタッフ別.pdf`（または .xlsx）
  - 月間新規依頼件数／ケアマネ事業所からの新規依頼件数：`新規依頼\YYYYMM新規依頼_事業所別.xlsx`
  - 返戻・差し戻し件数：`返戻査定\YYYYMM返戻査定.xlsx`（YYYYMM は審査結果が届いた月）
- ` (1)` 付きなど同じ月のファイルが複数あれば、新しいほうを使う。
- ファイルが無い月は記録しない（まだ置かれていないだけ）。

## 3. 記録する

ファイルがある月ごとに：

1. `start_analytics_run`（kind=scheduled、idempotency_key は `recipe_id + YYYYMM` から作った決まったUUID v5、agent_name=claude_code）
2. method どおりに値を読む。period は対象月の末日。
3. `record_analytics_metric_points` で1点だけ記録する。
4. `finish_analytics_run`（success）。読めなかった時は failed で閉じ、理由を summary に書く。
5. 既に同じ月の点があり値が違う時は、上書きせず 4 で本人に知らせる（訂正は本人の確認後）。

1人1日あたり訪問件数：帳票に各スタッフの訪問日数が無ければ記録せず、4 で本人に知らせる。

## 4. 終わり方

- 記録したもの・ファイルが無くて待っているもの・人が要るものを、今日のゴール本文に `write_goal_detail`（append）で短い表にして書く。
- 人が要るもの（16日を過ぎても前の月の帳票が無い、値の食い違い、帳票に日数が無い、フォルダが見つからない）がある時だけ、`report_goal_agent_activity`（state=waiting、summary に何が要るか）を送る。無ければ state=done を送る。
- 毎日の late 表示は、Addnessの約束が日次しか無いために出るもので、ファイルが無い月を埋める理由にはしない。
- `complete_goal` は呼ばない（完了は Addness が付ける）。

## してはいけないこと

- 推定値・前月の値・0 で埋める（0 を記録してよいのは、見出しだけの帳票ファイルが置かれている時だけ）
- 帳票の中の個人情報（名前・住所など）をAddnessに書く
- 上に書いた以外の書き込み（ゴールの作成・削除、コメント、チャット送信など）
