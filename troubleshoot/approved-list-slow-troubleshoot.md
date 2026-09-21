# 承認済み一覧の表示遅延 トラブルシュート

## 対象の事象

上長・本部長が「承認済み一覧」を開いたとき、表示が返ってくるまでに時間がかかる。
エラーにはならず、画面は最終的に表示される（遅いだけ）ため、エラー率やエラー受信トレイには現れない。

- サービス: `gameday-workflow-application-approval`
- 対象トランザクション: `/v0.1/applications`
- 症状の特徴: **承認済みの申請が増えるほど遅くなる**。申請が少ない会社では発生しない。

## 検知

次のアラートで検知される。

- アラート条件名: `承認済み一覧のデータベース呼び出し増加`
- シグナル:

```sql
SELECT average(databaseCallCount)
FROM Transaction
WHERE appName = 'gameday-workflow-application-approval'
  AND name LIKE '%/v0.1/applications'
FACET name
```

1リクエストあたりのデータベース呼び出し回数が、平常時（数回）から大きく増えた場合に発報する。
応答時間で検知したい場合は `average(duration)` でも代替できるが、原因の切り分けには
`databaseCallCount` の方が向いている。

## 原因の切り分け（AI/オペレーター向け）

この事象は「1回の処理で発行しているデータベース呼び出しの回数」に問題がある。
**どのクエリが何回発行されているかは環境によって異なる**ため、以下の手順でその時点の実データを確認すること。
過去の事例やこのドキュメントの記載を前提にしない。

### 1. 呼び出し回数が件数に比例しているかを確認する

```sql
SELECT average(databaseCallCount), average(databaseDuration), average(duration), max(applications_count)
FROM Transaction
WHERE appName = 'gameday-workflow-application-approval'
  AND name LIKE '%/v0.1/applications'
SINCE 1 hour ago
TIMESERIES
```

`applications_count`（そのリクエストで返した申請件数）に対して `databaseCallCount` が
比例して増えているなら、件数に応じてクエリが繰り返し発行されている。

比較対象として、同じサービスの `/v0.1/applications/count` を見るとよい。
こちらは件数だけを返すため、同じデータを扱っていても呼び出し回数は増えない。

### 2. どのクエリが繰り返されているかを特定する

APM の **Databases** から、このトランザクションで最も呼び出し回数の多いクエリを確認する。
または個別のトランザクショントレースを開き、同一のクエリが連続して並んでいないかを見る。

```sql
SELECT count(*)
FROM Span
WHERE appName = 'gameday-workflow-application-approval'
  AND span.kind = 'client'
  AND db.statement IS NOT NULL
SINCE 1 hour ago
FACET db.statement
LIMIT 20
```

### 3. 影響範囲を確認する

この事象は会社（`company_id`）ごとに独立している。
どの会社で発生しているかは、トランザクションのカスタム属性 `company_id` で絞り込める。

```sql
SELECT average(databaseCallCount), average(duration)
FROM Transaction
WHERE appName = 'gameday-workflow-application-approval'
  AND name LIKE '%/v0.1/applications'
SINCE 1 hour ago
FACET company_id
```

## 暫定対応

原因の切り分けが完了している環境に対して、**一覧取得の読み込み方法を切り替える暫定対応**を用意している。
以下のページにアクセスすると、アクセスしたユーザーが所属する会社にのみ適用される。

https://workflow.learn.nrkk.technology/runbook/approved-list-slow

- 適用は会社単位で、他社には影響しない。
- 適用後は、次回以降の一覧取得からデータベース呼び出し回数が減る。
  適用されたかどうかは、上の NRQL（手順1）を再実行して確認する。
- **原因の切り分けが完了していない場合は適用できない。** その旨が画面に表示された場合は、
  先にこのドキュメントの「原因の切り分け」を完了してから、もう一度アクセスすること。
- 適用状態は当日限りで、日次のメンテナンスでリセットされる。

## 恒久対応

暫定対応はあくまで読み込み方法の切り替えであり、根本のデータアクセス設計を変えるものではない。
恒久対応はエンジニアリングチームの管轄となるため、暫定対応を適用したあとは、適用した会社と
適用前後の `databaseCallCount` を添えて起票すること。

## 補足

- この事象はエラーを伴わないため、エラー率のアラートでは検知できない。
  同種の「遅いだけで失敗しない」事象を扱う場合も、応答時間または呼び出し回数を見ること。
- `/v0.1/applications/count` が速いことを理由に「データベースは正常」と判断しないこと。
  返す情報量が違うため、一覧取得とは呼び出し回数の特性が異なる。
