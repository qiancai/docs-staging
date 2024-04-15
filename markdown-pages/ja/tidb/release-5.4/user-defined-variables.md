---
title: User-Defined Variables
summary: Learn how to use user-defined variables.
---

# ユーザー定義変数 {#user-defined-variables}

このドキュメントでは、TiDBのユーザー定義変数の概念と、ユーザー定義変数を設定および読み取る方法について説明します。

> **警告：**
>
> ユーザー定義変数はまだ実験的機能です。実稼働環境で使用することはお勧めし**ません**。

ユーザー定義変数の形式は`@var_name`です。 `var_name`を構成する文字は、数字`0-9` 、文字`a-zA-Z` 、下線`_` 、ドル記号`$` 、UTF-8文字など、識別子を構成できる任意の文字にすることができます。さらに、英語の期間`.`も含まれます。ユーザー定義変数では大文字と小文字は区別されません。

ユーザー定義変数はセッション固有です。つまり、1つのクライアント接続によって定義されたユーザー変数は、他のクライアント接続によって表示または使用することはできません。

## ユーザー定義変数を設定します {#set-the-user-defined-variables}

`SET`ステートメントを使用してユーザー定義変数を設定できます。構文は`SET @var_name = expr [, @var_name = expr] ...;`です。例えば：


```sql
SET @favorite_db = 'TiDB';
```


```sql
SET @a = 'a', @b = 'b', @c = 'c';
```

代入演算子には、 `:=`を使用することもできます。例えば：


```sql
SET @favorite_db := 'TiDB';
```

代入演算子の右側の内容は、任意の有効な式にすることができます。例えば：


```sql
SET @c = @a + @b;
```


```sql
set @c = b'1000001' + b'1000001';
```

## ユーザー定義変数を読み取る {#read-the-user-defined-variables}

ユーザー定義変数を読み取るには、 `SELECT`ステートメントを使用して次のクエリを実行できます。


```sql
SELECT @a1, @a2, @a3
```

```
+------+------+------+
| @a1  | @a2  | @a3  |
+------+------+------+
|    1 |    2 |    4 |
+------+------+------+
```

`SELECT`ステートメントで値を割り当てることもできます。

```sql
SELECT @a1, @a2, @a3, @a4 := @a1+@a2+@a3;
```

```
+------+------+------+--------------------+
| @a1  | @a2  | @a3  | @a4 := @a1+@a2+@a3 |
+------+------+------+--------------------+
|    1 |    2 |    4 |                  7 |
+------+------+------+--------------------+
```

変数`@a4`が変更されるか、接続が閉じられる前は、その値は常に`7`です。

ユーザー定義変数の設定時に16進リテラルまたは2進リテラルが使用されている場合、TiDBはそれを2進文字列として扱います。数値に設定する場合は、手動で`CAST`変換を追加するか、式で数値演算子を使用できます。


```sql
SET @v1 = b'1000001';
SET @v2 = b'1000001'+0;
SET @v3 = CAST(b'1000001' AS UNSIGNED);
```


```sql
SELECT @v1, @v2, @v3;
```

```
+------+------+------+
| @v1  | @v2  | @v3  |
+------+------+------+
| A    | 65   | 65   |
+------+------+------+
```

初期化されていないユーザー定義変数を参照する場合、その変数の値はNULLであり、文字列のタイプがあります。


```sql
SELECT @not_exist;
```

```
+------------+
| @not_exist |
+------------+
| NULL       |
+------------+
```

`SELECT`ステートメントを使用してユーザー定義変数を読み取ることに加えて、別の一般的な使用法は`PREPARE`ステートメントです。例えば：


```sql
SET @s = 'SELECT SQRT(POW(?,2) + POW(?,2)) AS hypotenuse';
PREPARE stmt FROM @s;
SET @a = 6;
SET @b = 8;
EXECUTE stmt USING @a, @b;
```

```
+------------+
| hypotenuse |
+------------+
|         10 |
+------------+
```

ユーザー定義変数の内容は、SQLステートメントでは識別子として認識されません。例えば：


```sql
SELECT * from t;
```

```
+---+
| a |
+---+
| 1 |
+---+
```


```sql
SET @col = "`a`";
SELECT @col FROM t;
```

```
+------+
| @col |
+------+
| `a`  |
+------+
```

## MySQLの互換性 {#mysql-compatibility}

`SELECT ... INTO <variable>`を除いて、MySQLとTiDBでサポートされている構文は同じです。

詳細については、 [MySQLのユーザー定義変数](https://dev.mysql.com/doc/refman/5.7/en/user-variables.html)を参照してください。