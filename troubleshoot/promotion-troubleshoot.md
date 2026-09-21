# プロモーション申請バリデーションエラー トラブルシュート

## 対象エラー

```
バリデーションルール assertion_rule_promotion_description を満たしていません（pattern mismatch）
```

このエラーは `gameday-workflow-application-approval` が申請内容（`promotion.description`）を
正規表現ルールで検証した際に、パターン不一致で発生する。

## よくある事象例

ユーザーに見えているプロモーション申請内容:

```
対象者：yamada.taro@learn.nrkk.technology
内容：プロモーション：L3からL4

現職でのプロジェクトマネジメント実績、技術リード経験を評価し、昇進を申請します。
```

ユーザーには「入力内容が誤っている」ことしか分からず、どの記法が正しいのかは表示されない。
実際に要求されているパターンは DB に登録された正規表現次第であり、**会社（`company_id`）ごとに
異なる可能性がある**。そのため、このドキュメントには具体的な正規表現やOK/NG例を記載しない。
必ず調査手順に従って、その時点で実際に適用されているルールを都度確認すること。

## 原因

- バリデーションルールは DB テーブル `assertion_rules` に登録されている
  （`gameday-workflow-application-approval/db/init/04-init-assertion-rules.sql` が初期データ）。
- `application_type = 'promotion'`, `target_field = 'description'` に対して `regex_pattern` 型などのルールが
  適用され、`company_id` によって異なる設定が入っている場合がある。
- ここでの正規表現パターンは初期データ・運用中の変更内容によって変わり得るため、固定的な知識として
  扱わず、必ず New Relic 上の実データから取得すること。

## 調査手順（AI/オペレーター向け）

エラーが発生した際、ユーザーに提示すべき「正しいバリデーションルール」を特定するための手順。

### 1. New Relic で該当トランザクションエラーを特定する

- サービス: `gameday-workflow-application-approval`
- New Relic GUID: `ODQwMDYyOXxBUE18QVBQTElDQVRJT058NDM3NjU`

```sql
SELECT *
FROM TransactionError
WHERE appName = 'gameday-workflow-application-approval'
  AND error.message LIKE '%ASSERTION_RULE_VIOLATION%'
SINCE 1 hour ago
LIMIT 20
```

対象者のメールアドレスや申請時刻で絞り込む場合は `WHERE` 句に追加する。

### 2. カスタム属性からルール設定を読み取る

`app/services/rules/evaluator.py` の実装により、評価対象になった各ルールは
以下のカスタム属性として TransactionError に記録されている（`{n}` はルールの `order` 値）。

| 属性名 | 内容 |
|---|---|
| `assertion_rule{n}_id` | ルールID |
| `assertion_rule{n}_type` | ルール種別（`regex_pattern` / `min_length` / `max_length` / `forbidden_words` / `required_keyword`） |
| `assertion_rule{n}_target_field` | 検証対象フィールド（例: `description`） |
| `assertion_rule{n}_config` | ルールの設定（JSON文字列）。`regex_pattern` の場合は `{"pattern": "..."}` |
| `assertion_rule{n}_value` | 実際に検証された入力値 |
| `assertion_rule{n}_result` | 検証結果（`true`/`false`）。`false` のものが違反ルール |

`assertion_rule1_config`, `assertion_rule2_config` ... のように、申請フォームに設定されているルール数分だけ属性が並ぶ。
`assertion_rule{n}_result = false` になっている番号の `config` を見れば、実際に違反したルールが分かる。

```sql
SELECT
  assertion_rule1_id, assertion_rule1_config, assertion_rule1_result,
  assertion_rule2_id, assertion_rule2_config, assertion_rule2_result,
  assertion_rule3_id, assertion_rule3_config, assertion_rule3_result
FROM TransactionError
WHERE appName = 'gameday-workflow-application-approval'
  AND error.message LIKE '%ASSERTION_RULE_VIOLATION%'
SINCE 1 hour ago
LIMIT 20
```

### 3. 取得した `config` からルールの意味をその場で解釈する

- `assertion_rule{n}_result = false` の行の `config.pattern`（または他の rule_type のパラメータ）を確認する。
- 正規表現の意味は事前知識やドキュメントの記載に頼らず、その都度 `pattern` の内容そのものを読んで解釈する。
  （会社ごとに異なるパターンが設定されている可能性があるため、他社や過去事例のパターンを前提にしない）
- 併せて `assertion_rule{n}_value`（実際にユーザーが入力した値）と比較し、どこが不一致なのかを具体的に特定する。

## ユーザーへの回答テンプレート

取得した実際の `pattern` と `value` を基に、以下の形式で回答する（`{...}` は調査結果で置き換える）。

```
プロモーション申請の「内容」欄が、システムのバリデーションルールに一致していないためエラーになっています。

現在の記載: 「{ユーザーが入力した値}」
必要な形式: {config.pattern から読み取った、要求されるフォーマットの説明}

以下のように修正して再申請してください。

修正後の例:
{pattern に一致する具体的な記載例}
```

## 補足

- ルールは `company_id` ごとに異なる場合があるため、必ず New Relic の `assertion_rule{n}_config` で
  **実際に適用されたルール** を確認してから回答すること。過去の事例やこのドキュメントに書かれた
  想定パターンを鵜呑みにしない。
- ルール自体の妥当性（表記の許可範囲など）に疑問がある場合は、バリデーションルール変更の要否を
  別途エンジニアリングチームに確認する。
