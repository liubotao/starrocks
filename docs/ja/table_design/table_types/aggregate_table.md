---
displayed_sidebar: docs
sidebar_position: 40
---

# 集計テーブル

テーブル作成時に、集計キーを定義し、値カラムに対して集計関数を指定できます。同じ集計キーを持つ複数のデータ行がある場合、値カラムの値が集計されます。また、ソートキーを別に定義することもできます。クエリのフィルター条件にソートキーが含まれている場合、StarRocks はデータを迅速にフィルタリングし、クエリの効率を向上させます。

データ分析や集計のシナリオでは、集計テーブルを使用することで処理するデータ量を削減し、クエリの効率を向上させることができます。

## シナリオ

集計テーブルは、データ統計や分析のシナリオに適しています。以下はいくつかの例です。

- ウェブサイトやアプリのプロバイダーが、特定のウェブサイトやアプリでユーザーが費やすトラフィック量や時間、訪問回数を分析するのに役立ちます。

- 広告代理店が、顧客に提供する広告の総クリック数、総表示数、消費統計を分析するのに役立ちます。

- Eコマース企業が、年間の取引データを分析し、四半期や月ごとの地理的なベストセラーを特定するのに役立ちます。

上記のシナリオにおけるデータクエリと取り込みには以下の特徴があります。

- ほとんどのクエリは、SUM、MAX、MIN などの集計クエリです。
- 生の詳細データを取得する必要はありません。
- 履歴データは頻繁に更新されず、新しいデータのみが追加されます。

## 原理

データ取り込みからデータクエリまで、集計テーブル内のデータは以下のように複数回集計されます。

1. データ取り込みフェーズでは、データがバッチで集計テーブルにロードされるときに、各バッチのデータがバージョンを形成します。1つのバージョン内で、同じ集計キーを持つデータが集計されます。

2. バックグラウンドの Compaction フェーズでは、データ取り込みで生成された複数のデータバージョンのファイルが定期的に大きなファイルに圧縮される際に、StarRocks は大きなファイル内で同じ集計キーを持つデータを集計します。

3. データクエリフェーズでは、StarRocks はすべてのデータバージョン間で同じ集計キーを持つデータを集計し、クエリ結果を返す前に集計します。

集計操作は、処理するデータ量を削減し、クエリを高速化するのに役立ちます。

集計テーブルを使用して、次の4つの生データをテーブルにロードしたいとします。

| Date       | Country | PV   |
| ---------- | ------- | ---- |
| 2020.05.01 | CHN     | 1    |
| 2020.05.01 | CHN     | 2    |
| 2020.05.01 | USA     | 3    |
| 2020.05.01 | USA     | 4    |

StarRocks はデータ取り込み時に、これらの4つの生データを次の2つのデータに集計します。

| Date       | Country | PV   |
| ---------- | ------- | ---- |
| 2020.05.01 | CHN     | 3    |
| 2020.05.01 | USA     | 7    |

## テーブルの作成

異なる都市からのユーザーが異なるウェブページを訪問した回数を分析したいとします。この例では、`example_db.aggregate_tbl` という名前のテーブルを作成し、`site_id`、`date`、`city_code` を集計キーとして定義し、`pv` を値カラムとして定義し、`pv` カラムに対して SUM 関数を指定します。

テーブルを作成するためのステートメントは次のとおりです。

```SQL
CREATE TABLE aggregate_tbl (
    site_id LARGEINT NOT NULL COMMENT "id of site",
    date DATE NOT NULL COMMENT "time of event",
    city_code VARCHAR(20) COMMENT "city_code of user",
    pv BIGINT SUM DEFAULT "0" COMMENT "total page views"
)
AGGREGATE KEY(site_id, date, city_code)
DISTRIBUTED BY HASH(site_id);
```

> **NOTICE**
>
> - テーブルを作成する際には、`DISTRIBUTED BY HASH` 句を使用してバケット化カラムを指定する必要があります。詳細については、[bucketing](../data_distribution/Data_distribution.md#bucketing) を参照してください。
> - バージョン 2.5.7 以降、StarRocks はテーブルを作成する際やパーティションを追加する際に、バケット数 (BUCKETS) を自動的に設定できます。バケット数を手動で設定する必要はありません。詳細については、[set the number of buckets](../data_distribution/Data_distribution.md#set-the-number-of-buckets) を参照してください。

import Beta from '../../_assets/commonMarkdown/_beta.mdx'

## 一般的な集計関数の状態

<Beta />

StarRocks はバージョン 3.4.0 から一般的な集計関数の状態をサポートしています。

データ分析やサマリーにおいて、集計テーブルはクエリ中に処理されるデータを削減し、クエリパフォーマンスを向上させます。大規模なデータセットに対して、集計テーブルはクエリ前にデータを次元ごとに要約するため非常に効果的です。また、StarRocks における増分集計関数計算の重要な方法としても機能します。しかし、以前のバージョンでは、`SUM`、`MAX`、`MIN`、`REPLACE`、`HLL_UNION`、`PERCENTILE_UNION`、`BITMAP_UNION` などの組み込み関数のみがサポートされており、理論的にはすべての組み込み集計関数が集計テーブルで使用できるはずです。この制限に対処するために、すべての組み込み関数状態のストレージをサポートする一般的な集計状態が導入されました。

### 一般的な集計状態の保存

集計テーブルで関数名と入力パラメータータイプを指定することで、一般的な集計状態を定義し、集計関数を一意に識別できます。カラムタイプは、集計関数の中間状態タイプとして自動的に推論されます。

定義:

```SQL
col_name agg_func_name(parameter1_type, [parameter2_type], ...)
```

- **col_name**: カラムの名前。
- **agg_func_name**: 中間状態を保存する必要がある集計関数の名前。
- **parameter_type**: 集計関数の入力パラメータータイプ。パラメータータイプで関数を一意に識別できます。

:::note

- 少なくとも1つのパラメーターを持つ StarRocks の組み込み関数のみがサポートされています。Java および Python の UDAF はサポートされていません。
- 安定性と拡張性のために、集計状態カラムタイプは常に Nullable です（count 関数を除く）であり、変更できません。
- 複数パラメーター関数を定義する際にパラメーター値は不要です。タイプは推論でき、パラメーター値は計算に関与しません。
- ORDER BY や DISTINCT などの複雑なパラメーターはサポートされていません。
- `GROUP_CONCAT`、`WINDOW_FUNNEL`、`APPROX_TOP_K` などの特定の組み込み関数のサポートはまだ開発中です。将来のリリースでサポートされる予定です。詳細については、[FunctionSet.java#UNSUPPORTED_AGG_STATE_FUNCTIONS](https://github.com/StarRocks/starrocks/blob/main/fe/fe-core/src/main/java/com/starrocks/catalog/FunctionSet.java#L776) を参照してください。

:::

例:

```SQL
CREATE TABLE test_create_agg_table (
  dt VARCHAR(10),
  -- 一般的な集計状態ストレージを定義します。
  hll_sketch_agg ds_hll_count_distinct(varchar),
  avg_agg avg(bigint),
  array_agg_agg array_agg(int),
  min_by_agg min_by(varchar, bigint)
)
AGGREGATE KEY(dt)
PARTITION BY (dt) 
DISTRIBUTED BY HASH(dt) BUCKETS 4;
```

### コンビネータ関数

一般的な集計状態は、コンビネータ関数を使用して中間状態の計算とフローをカプセル化します。

#### `_state` コンビネータ関数

`_state` 関数は、入力パラメーターを中間状態タイプに変換します。

定義:

```SQL
agg_intermediate_type {agg_func_name}_state(input_col1, [input_col2], ...)
```

- **agg_func_name**: 入力パラメーターを中間状態タイプに変換する必要がある集計関数の名前。
- **input_col1/col2**: 集計関数の入力カラム。
- **agg_intermediate_type**: `_state` 関数の戻り値のタイプであり、集計関数の中間状態タイプです。

:::note

`_state` はスカラ関数です。入力パラメーター状態の計算のために集計カラムを定義する必要はありません。

:::

例:

```SQL
CREATE TABLE t1 (
  id BIGINT NOT NULL,
  province VARCHAR(64),
  age SMALLINT,
  dt VARCHAR(10) NOT NULL 
)
DUPLICATE KEY(id)
PARTITION BY (dt)
DISTRIBUTED BY HASH(id) BUCKETS 4;

INSERT INTO t1 SELECT generate_series, generate_series, generate_series % 10, "2024-07-24" FROM table(generate_series(1, 100));

-- _state コンビネータ関数を使用して t1 のデータを転送し、集計テーブルに挿入します。
INSERT INTO test_create_agg_table
SELECT
    dt,
    ds_hll_count_distinct_state(id),
    avg_state(id),
    array_agg_state(id),
    min_by_state(province, id)
FROM t1;
```

#### `_union` コンビネータ関数

`_union` 関数は、複数の中間状態カラムを1つの状態にマージします。

定義:

```SQL
-- 複数の集計中間状態を結合します。
agg_intermediate_type {agg_func_name}_union(input_col)
```

- **agg_func_name**: 集計関数の名前。
- **input_col**: 集計関数の入力カラム。入力カラムタイプは集計関数の中間状態タイプです。`_state` 関数を使用して取得できます。
- **agg_intermediate_type**: `_union` 関数の戻り値のタイプであり、集計関数の中間状態タイプです。

:::note

`_union` は集計関数です。関数の最終結果のタイプではなく、中間状態タイプを返します。

:::

例:

```SQL
-- ケース 1: 集計テーブルの中間状態を結合します。
SELECT 
    dt,
    ds_hll_count_distinct_union(hll_sketch_agg),
    avg_union(avg_agg),
    array_agg_union(array_agg_agg),
    min_by_union(min_by_agg)
FROM test_create_agg_table
GROUP BY dt
LIMIT 1;

-- ケース 2: _state コンビネータ関数によって入力された中間状態を結合します。
SELECT 
    dt,
    ds_hll_count_distinct_union(ds_hll_count_distinct_state(id)),
    avg_union(avg_state(id)),
    array_agg_union(array_agg_state(id)),
    min_by_union(min_by_state(province, id))
FROM t1
GROUP BY dt
LIMIT 1;
```

#### `_merge` コンビネータ関数

`_merge` コンビネータ関数は、集計関数を一般的な集計関数としてカプセル化し、複数の中間状態の最終集計結果を計算します。

定義:

```SQL
-- 複数の集計中間状態をマージします。
agg_result_type {agg_func_name}_merge(input_col)
```

- **agg_func_name**: 集計関数の名前。
- **input_col**: 集計関数の入力カラム。入力カラムタイプは集計関数の中間状態タイプです。`_state` 関数を使用して取得できます。
- **agg_intermediate_type**: `_merge` 関数の戻り値のタイプであり、集計関数の最終集計結果です。

例:

```SQL
-- ケース 1: 集計テーブルの中間状態をマージして最終集計結果を取得します。
SELECT 
    dt,
    ds_hll_count_distinct_merge(hll_sketch_agg),
    avg_merge(avg_agg),
    array_agg_merge(array_agg_agg),
    min_by_merge(min_by_agg)
FROM test_create_agg_table
GROUP BY dt
LIMIT 1;

-- ケース 2: _state コンビネータ関数によって入力された中間状態をマージして最終集計結果を取得します。
SELECT 
    dt,
    ds_hll_count_distinct_merge(ds_hll_count_distinct_state(id)),
    avg_merge(avg_state(id)),
    array_agg_merge(array_agg_state(id)),
    min_by_merge(min_by_state(province, id))
FROM t1
GROUP BY dt
LIMIT 1;
```

### マテリアライズドビューでの一般的な集計状態の使用

一般的な集計状態は、同期および非同期マテリアライズドビューで使用され、集計状態のロールアップによってクエリパフォーマンスを加速します。

#### 同期マテリアライズドビューでの一般的な集計状態

例:

```SQL
-- 集計状態を保存する同期マテリアライズドビュー test_mv1 を作成します。
CREATE MATERIALIZED VIEW test_mv1 
AS
SELECT 
    dt,
    -- オリジナルの集計関数。
    min(id) AS min_id,
    max(id) AS max_id,
    sum(id) AS sum_id,
    bitmap_union(to_bitmap(id)) AS bitmap_union_id,
    hll_union(hll_hash(id)) AS hll_union_id,
    percentile_union(percentile_hash(id)) AS percentile_union_id,
    -- 一般的な集計状態関数。
    ds_hll_count_distinct_union(ds_hll_count_distinct_state(id)) AS hll_id,
    avg_union(avg_state(id)) AS avg_id,
    array_agg_union(array_agg_state(id)) AS array_agg_id,
    min_by_union(min_by_state(province, id)) AS min_by_province_id
FROM t1
GROUP BY dt;

-- ロールアップ作成が完了するまで待ちます。
show alter table rollup;

-- 集計関数に対する直接クエリは、test_mv1 によって透過的に加速されます。
SELECT 
    dt,
    min(id),
    max(id),
    sum(id),
    bitmap_union_count(to_bitmap(id)), -- count(distinct id)
    hll_union_agg(hll_hash(id)), -- approx_count_distinct(id)
    percentile_approx(id, 0.5),
    ds_hll_count_distinct(id),
    avg(id),
    array_agg(id),
    min_by(province, id)
FROM t1
WHERE dt >= '2024-01-01'
GROUP BY dt;

-- 集計関数とロールアップに対する直接クエリも、test_mv1 によって透過的に加速されます。
SELECT 
    min(id),
    max(id),
    sum(id),
    bitmap_union_count(to_bitmap(id)), -- count(distinct id)
    hll_union_agg(hll_hash(id)), -- approx_count_distinct(id)
    percentile_approx(id, 0.5),
    ds_hll_count_distinct(id),
    avg(id),
    array_agg(id),
    min_by(province, id)
FROM t1
WHERE dt >= '2024-01-01';

DROP MATERIALIZED VIEW test_mv1;
```

#### 非同期マテリアライズドビューでの一般的な集計状態

例:

```SQL
-- 集計状態を保存する非同期マテリアライズドビュー test_mv2 を作成します。
CREATE MATERIALIZED VIEW test_mv2
PARTITION BY (dt)
DISTRIBUTED BY RANDOM
AS
SELECT 
    dt,
    -- オリジナルの集計関数。
    min(id) AS min_id,
    max(id) AS max_id,
    sum(id) AS sum_id,
    bitmap_union(to_bitmap(id)) AS bitmap_union_id,
    hll_union(hll_hash(id)) AS hll_union_id,
    percentile_union(percentile_hash(id)) AS percentile_union_id,
    -- 一般的な集計状態関数。
    ds_hll_count_distinct_union(ds_hll_count_distinct_state(id)) AS hll_id,
    avg_union(avg_state(id)) AS avg_id,
    array_agg_union(array_agg_state(id)) AS array_agg_id,
    min_by_union(min_by_state(province, id)) AS min_by_province_id
FROM t1
GROUP BY dt;

-- マテリアライズドビューをリフレッシュします。
REFRESH MATERIALIZED VIEW test_mv2 WITH SYNC MODE;

-- 集計関数に対する直接クエリは、test_mv2 によって透過的に加速されます。
SELECT 
    dt,
    min(id),
    max(id),
    sum(id),
    bitmap_union_count(to_bitmap(id)), -- count(distinct id)
    hll_union_agg(hll_hash(id)), -- approx_count_distinct(id)
    percentile_approx(id, 0.5),
    ds_hll_count_distinct(id),
    avg(id),
    array_agg(id),
    min_by(province, id)
FROM t1
WHERE dt >= '2024-01-01'
GROUP BY dt;

SELECT 
    min(id),
    max(id),
    sum(id),
    bitmap_union_count(to_bitmap(id)), -- count(distinct id)
    hll_union_agg(hll_hash(id)), -- approx_count_distinct(id)
    percentile_approx(id, 0.5),
    ds_hll_count_distinct(id),
    avg(id),
    array_agg(id),
    min_by(province, id)
FROM t1
WHERE dt >= '2024-01-01';
```

## 使用上の注意

- **集計キー**:
  - CREATE TABLE ステートメントでは、集計キーは他のカラムの前に定義する必要があります。
  - 集計キーは `AGGREGATE KEY` を使用して明示的に定義できます。`AGGREGATE KEY` には値カラムを除くすべてのカラムを含める必要があります。そうでない場合、テーブルの作成に失敗します。

    集計キーが `AGGREGATE KEY` を使用して明示的に定義されていない場合、値カラムを除くすべてのカラムがデフォルトで集計キーと見なされます。
  - 集計キーには一意性制約があります。

- **値カラム**: カラム名の後に集計関数を指定することで、カラムを値カラムとして定義します。このカラムは通常、集計が必要なデータを保持します。

- **集計関数**: 値カラムに使用される集計関数。集計テーブルでサポートされている集計関数については、[CREATE TABLE](../../sql-reference/sql-statements/table_bucket_part_index/CREATE_TABLE.md) を参照してください。

- **ソートキー**

  - バージョン 3.3.0 以降、集計テーブルでソートキーは集計キーから分離されています。集計テーブルは `ORDER BY` を使用してソートキーを指定し、`AGGREGATE KEY` を使用して集計キーを指定することをサポートしています。ソートキーと集計キーのカラムは同じである必要がありますが、カラムの順序は同じである必要はありません。

  - クエリが実行されるとき、ソートキーカラムは複数のデータバージョンの集計前にフィルタリングされ、値カラムは複数のデータバージョンの集計後にフィルタリングされます。したがって、フィルター条件として頻繁に使用されるカラムを特定し、これらのカラムをソートキーとして定義することをお勧めします。これにより、複数のデータバージョンの集計前にデータフィルタリングを開始し、クエリパフォーマンスを向上させることができます。

- テーブルを作成する際には、テーブルのキーカラムに対してのみビットマップインデックスまたはブルームフィルターインデックスを作成できます。

## 次に行うこと

テーブルが作成された後、さまざまなデータ取り込み方法を使用して StarRocks にデータをロードできます。StarRocks がサポートするデータ取り込み方法については、[Loading options](../../loading/Loading_intro.md) を参照してください。

> 注: 集計テーブルを使用するテーブルにデータをロードする際には、テーブルのすべてのカラムを更新する必要があります。たとえば、前述の `example_db.aggregate_tbl` テーブルを更新する場合、`site_id`、`date`、`city_code`、および `pv` のすべてのカラムを更新する必要があります。
```