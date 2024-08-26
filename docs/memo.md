## 3 章

### まとめ

- SQL パフォーマンスはストレージへの I/O をどれだけ減らせるか
- UNION で条件分岐を表現したくなったら、冗長になっていないかを診断する
- IN や CASE 式で条件分岐を表現できれば、テーブルへのスキャンを大幅に減らせる可能性がある
- そのためにも、文から式へのパラダイムシフトを取得するべき

## 4 章

### まとめ

- GROUP BY 句やウィンドウ関数の PARTITION BY 句は集合のカットをしている
- GROUP BY 句やウィンドウ関数は内部的にハッシュまたはソートの処理が実行されている
- ハッシュやソートはメモリを多く使用する
  - もしメモリが不足した場合は一時領域としてストレージが使用され、パフォーマンスに問題を引き起こす
- GROUP BY 句やウィンドウ関数と CASE 式を組み合わせると非常に強力な表現力を持つ

### 演習問題

**問題**

```
# リスト4.8 頭文字のアルファベットごとに何人がテーブルに存在するか集計するSQL
SELECT SUBSTRING(name, 1, 1) AS label,
         COUNT(*)
  FROM Persons
 GROUP BY SUBSTRING(name, 1, 1);
```

上記の SQL について、DBMS における実行計画を取得して、GROUP BY・集約関数の演算にソートとハッシュのどちらが使用されているか調べろ。

**回答**

EXPLAIN の結果。

```
id	select_type	table	partitions	type	possible_keys	key	key_len	ref	rows	filtered	Extra
1	SIMPLE	Persons	NULL	index	PRIMARY	PRIMARY	34	NULL	9	100.00	Using index; Using temporary
```

Using temporary なのでどちらも使用していない？
書籍では`filter sort`が使われるとある。差異が出る理由 ↓

> まず、「EXPLAIN」の結果は、MySQL がクエリをどのように実行するかの「予測」を示しているんだ。この予測は MySQL のクエリオプティマイザによって作られるんだけど、そのオプティマイザはテーブルの統計情報（行数、インデックスの状態など）を基にしてクエリの最適な実行計画を決定するんだ。
> だから、実行計画は以下のような要素によって変わる可能性があるよ。
>
> 1. テーブルのデータ量
> 2. 使用するインデックスの種類や状態
> 3. クエリの構造
> 4. MySQL の設定（例えば、バッファプールのサイズなど）
>    そのため、書籍の例と実際の結果が異なる場合、それらの要素の違いが影響している可能性があるんだ。
>    ただし、「Using index; Using temporary」が表示されるということは、MySQL が一時テーブルを使ってクエリを実行していることを示しているよ。これは、GROUP BY 句があるために、MySQL がデータを一時的に保存する必要があるからだね。
>    なお、「filesort」が表示されないということは、MySQL がソート操作を行っていない、もしくはインデックスを使ってソート操作を行っていることを示しているよ。これは、クエリの構造や使用するインデックス、MySQL の設定などによって変わるんだ。

## 5 章

### 演習問題

下記と同値な SQL 文を相関サブクエリを使って作ってください。

```
■リスト5.3 ウィンドウ関数を使った解
INSERT INTO Sales2
SELECT company,
       year,
       sale,
       CASE SIGN(sale - MAX(sale)
                         OVER ( PARTITION BY company
                                    ORDER BY year
                                     ROWS BETWEEN 1 PRECEDING
                                              AND 1 PRECEDING) )
       WHEN 0 THEN '='
       WHEN 1 THEN '+'
       WHEN -1 THEN '-'
       ELSE NULL END AS var
  FROM Sales;
```

> ※相関クエリとは？
>
> 相関クエリは、外側のクエリと内側のサブクエリが相互に関連する SQL のクエリのこと。サブクエリの結果は外側のクエリの各行に影響を与える。つまり、外側のクエリが行を一つずつ処理するたびに、サブクエリもそれに合わせて実行される。

**回答**

```
SELECT
    company,
    year,
    sale,
    CASE
        WHEN sale - (
            SELECT sale
            FROM Sales S2
            WHERE S1.company = S2.company
            AND year = (
                SELECT MAX(year)
                FROM Sales S3
                WHERE S1.company = S3.company
                AND S1.year > S3.year
            )
        ) > 0 THEN '+'
        WHEN sale - (
            SELECT sale
            FROM Sales S2
            WHERE S1.company = S2.company
            AND year = (
                SELECT MAX(year)
                FROM Sales S3
                WHERE S1.company = S3.company
                AND S1.year > S3.year
            )
        ) < 0 THEN '-'
        ELSE '='
    END AS var
FROM Sales S1;

```

##  6章
### MEMO
- 結合と相関サブクエリのどちらを使っても算出できる場合は、結合を使用した方がパフォーマンスが良い
  - 理由: 相関サブクエリをスカラサブクエリとして使うと、結果行数の数だけ相関サブクエリを実行することになるため

### まとめ
- 結合はSQLの性能問題の火薬庫
- 基本はNested Loop、バッチやBI/DWH限定でHash。Hashを使う時はTEMP落ちに注意
- Nested Loopが効率的に動くには「小さな駆動表」と「内部表のインデックス」が必要
- 結合のアルゴリズムが複数あるために実行計画変動も起きやすい。これを防止するためには「結合を回避する」ことが重要な戦略になる
  - 非正規化も選択肢であるが、ウィンドウ関数や相関サブクエリを検討するべし
