---
title: "「つなぎのLambda」はJSONataで消える。Step Functions 反復練習アプリ「JSONata ノック道場」"
emoji: "🥋"
type: "tech"
topics: ["aws", "stepfunctions", "jsonata", "serverless"]
published: true
---

こんにちは～　ウッキー(浮田)です。

この記事では、JSONata の「最初の5分」を解説したうえで、ブラウザだけで JSONata を反復練習できる無料アプリ「[JSONata ノック道場](https://ukkiy8787.github.io/Step-Functions/)」を作成したので紹介します。

ゴールは明確で、**データ変換のためだけに書いてきた「つなぎの Lambda」を JSONata で消すこと**。
基本の5つを押さえて50本打ち込めば、「この一行のためだけの Lambda」はもう書かなくてよくなります。



## 1. JSONata 以前 — 5つの設定の組み合わせパズル

かつて Step Functions のデータ加工は、InputPath → Parameters → ResultSelector → ResultPath → OutputPath という5つの設定の組み合わせで行っていました。それぞれ役割が微妙に違い、適用順序があり、`$` と `$.foo` と `"key.$"` の使い分けがありました。

そこに 2024年11月、JSONata の登場です。Step Functions が第2のクエリ言語として JSONata に対応し、データ加工の書き方が一変しました。

**Before（JSONPath 時代）**: Parameters と組み込み関数（intrinsics）で1項目ずつ組み立てる。文字列項目はこう書けても——

```json
{
  "Type": "Pass",
  "QueryLanguage": "JSONPath",
  "Parameters": {
    "to.$": "$.customer.email",
    "subject.$": "States.Format('ご注文 {} を受け付けました', $.orderId)"
  },
  "End": true
}
```

`.$` 付きキーが「式として評価」の合図、文字列連結は `&` ではなく `States.Format('… {} …', 値)`。そして——肝心の合計（各行の price × qty を足す）は組み込み関数では書けません。配列を合計する関数が無いのはもちろん、そもそも掛け算すら組み込み関数に無い（算術は2数を足す `States.MathAdd` だけ）ため、この一行のためだけに Lambda を1個用意するのが定番でした。

**After（JSONata）**: Output に式を1つ書くだけ。Lambda も5フィールドも消えます。

```json
{
  "Type": "Pass",
  "QueryLanguage": "JSONata",
  "Output": {
    "to": "{% $states.input.customer.email %}",
    "subject": "{% 'ご注文 ' & $states.input.orderId & ' を受け付けました' %}",
    "total": "{% $sum($states.input.items.(price * qty)) %}"
  },
  "End": true
}
```

注文データからメール用の3項目を組み立てて、ついでに合計金額の計算までやっています。JSONPath 時代は Lambda を1個書いていたような加工が、式だけで済む。これが JSONata です。

## 2. JSONata の基本 — まずこの5つだけ

### ① 入力は `$states.input` から始まる

Step Functions の JSONata では、ステートへの入力データは `$states.input` に入っています（詳細は公式の [Transforming data with JSONata](https://docs.aws.amazon.com/step-functions/latest/dg/transforming-data.html) を参照）。ここがすべての起点です。

```
入力: { "orderId": "ORD-123", "customer": { "name": "Alice" } }

$states.input.orderId          → "ORD-123"
$states.input.customer.name    → "Alice"
```

ドット記法でどこまでも潜れます。

ここで重要な注意をひとつ——`orderId` と裸で書いても動きません。本番の Step Functions では、入力は必ず `$states.input` 経由。これ、練習環境が緩いと身につかない罠です（後述）。

### ② 配列は「自動射影」される — JSONata の真骨頂

```
入力: { "items": [ { "name": "apple", "price": 100 },
                   { "name": "banana", "price": 50 } ] }

$states.input.items.name       → ["apple", "banana"]
```

配列に対してドットを使うと、全要素から値を集めてくれます。ループ不要。map 関数すら不要。

### ③ フィルタは `[条件]`

```
$states.input.items[price >= 100].name   → "apple"
```

SQL の WHERE 句を埋め込むような感覚です。

### ④ オブジェクトはそのまま組み立てる

```
{
  "summary": $states.input.orderId & " (" & $count($states.input.items) & "件)",
  "names": $states.input.items.name
}
```

`&` は文字列連結。出力の形をそのまま書けばいい。

### ⑤ 集計関数 — `$sum` / `$count` / `$average`

```
$sum($states.input.items.(price * qty))   → 各行の小計を出してから合計
```

`(price * qty)` のように、各要素の中で計算してから集める書き方ができます。ETL の変換ロジックの大半はこれで書けてしまいます。

## 3. で、どうやって練習する？ — 読むだけでは絶対に身につかない

ここまで読んで「なるほど、簡単そう」と思った方。……その感覚、3日で消えます。

クエリ言語は外国語と同じで、読めると書けるの間に深い谷があります。実務で「さあ書くぞ」となった瞬間、`$` はどこに付けるんだっけ、フィルタの括弧はどっちだっけ、と手が止まる。これを越える方法は一つしかありません。反復です。

そこで作ったのが——

## 4. JSONata ノック道場 — ブラウザだけ、無料、50本

[JSONata ノック道場](https://ukkiy8787.github.io/Step-Functions/) は、Step Functions の JSONata を「問題を解いて」身につける練習アプリです。

> 問題を読む → 式を書く → ⌘+Enter → その場で採点 ✓✗

### 特徴1: 50本を10レベルで段階的に

| Lv | テーマ | 例 |
| --- | --- | --- |
| 1 | アクセス基礎 | フィールド参照、配列、フィルタ |
| 2 | 変換 | オブジェクト構築、文字列連結、条件分岐 |
| 3 | 集計・並べ替え | `$sum` / `$average` / ソート |
| 4 | 関数 | 日時変換、`$merge`、`$keys` |
| 5 | Step Functions 実践 | `$states.result`、`$parse`、変数 |
| 6 | 高階関数 | `$map` / `$filter` / `$reduce` |
| 7 | 文字列・数値・日時 | `$split` / `$replace` / `$formatNumber` |
| 8 | Step Functions 実戦 | Map 結果集計、Catch エラー整形、Assign |
| 9 | 直接統合：入力を取り出す | API GW の body 展開、Pipes/SQS、Slack blocks、Bedrock リクエスト |
| 10 | 直接統合：結果を組み立てる | Bedrock 応答の抽出、DynamoDB 型付き形式、スコア判定 |

Lv.1 は「ドットでフィールドを取る」から始まるので、JSONata を今日初めて知った人でも入れます。Lv.8 まで終えると、Map ステートの結果集計や Catch で受けたエラーの整形——つまり実務でそのまま出てくる処理が書けるようになります。さらに Lv.9〜10 では、Lambda を JSONata + 直接統合で置き換える実戦パターン（API Gateway / EventBridge Pipes / Bedrock / DynamoDB / ECS）まで踏み込みます。

### 特徴2: 入力ルールと関数セットを「本番に合わせた」採点

これがこのアプリの一番のこだわりです。

世の中の JSONata プレイグラウンドは、入力 JSON をルートコンテキストに置くので、`orderId` と裸で書いても値が取れてしまいます。でも本番の Step Functions では動きません。練習環境が本番より緩いと、「道場では正解 → 本番でエラー」という最悪の学習体験が生まれます。

ノック道場は入力の参照ルールと関数セットを本番に合わせて採点します。具体的には、入力は `$states.input` 経由でしか参照できません。

裸の `orderId` はちゃんと不正解になり、「入力は `$states.input` から参照します」とガイドが出ます。

Step Functions 固有の `$parse` 関数も再現済み。`$eval` のような本番で使えない関数に頼る癖がつかないよう、問題設計の段階で排除しています。

:::message
厳密にはブラウザ上の JSONata エンジンなので、Step Functions が未対応の `??` / `?:` 演算子のように「道場では通っても本番では落ちる」構文がごく一部あります。これらは採点時に警告を出して気づけるようにしていますが、最後は実機でも一度流すのが確実です。
:::

### 特徴3: 続けやすさ

- **インストール不要** — ブラウザで開くだけ。スマホでも動きます
- **進捗は自動保存** — 途中でやめても「続きから」再開
- **ヒントと解説つき** — 詰まったら開く。解説には「なぜそう書くのか」まで
- **書きかけの式も保存** — 通勤電車で2問、昼休みに3問、が成立します

### 始め方

1. [JSONata ノック道場](https://ukkiy8787.github.io/Step-Functions/) を開く
2. 「一本目から始める →」を押す
3. 1日5本 × 10日で全50本。1本あたり1〜3分です

## 5. ノックのあとは — 実機で組む

道場で式が書けるようになったら、次は実機です。

- [公式: Transforming data with JSONata](https://docs.aws.amazon.com/step-functions/latest/dg/transforming-data.html) — Step Functions 上での JSONata の有効化、`{% %}` 記法、変数（Assign）、本番で使えない機能（`$eval`）までの一次情報
- [SFn道場 ハンズオン](https://ukkiy8787.github.io/Step-Functions/)（ノック道場と同じサイト内の Step Functions ハンズオン）— JSONata を組み込んだステートマシンを「カタ」として1本ずつ構築（バリデーション、ETL、DynamoDB CRUD まで）


「読む → 式を書く（いまここ） → 機械を組む」。この順で進めば、Step Functions のデータ加工で手が止まることは、もうありません。

## まとめ

- Step Functions のデータ加工は JSONata で書く時代。5つの設定パズルは卒業
- まず覚えるのは5つ: `$states.input` 起点・自動射影・フィルタ `[ ]`・オブジェクト構築・集計関数
- クエリ言語は反復でしか身につかない。[JSONata ノック道場](https://ukkiy8787.github.io/Step-Functions/) で50本、打ち込んでください
- 道場の採点は入力ルールと関数セットを本番に合わせてあるので、ここで身につけた書き方はそのまま実機で通用します（ごく一部の差は警告で気づけます）
