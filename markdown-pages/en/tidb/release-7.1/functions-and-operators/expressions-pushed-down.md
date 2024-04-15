---
title: List of Expressions for Pushdown
summary: Learn a list of expressions that can be pushed down to TiKV and the related operations.
---

# List of Expressions for Pushdown

When TiDB reads data from TiKV, TiDB tries to push down some expressions (including calculations of functions or operators) to be processed to TiKV. This reduces the amount of transferred data and offloads processing from a single TiDB node. This document introduces the expressions that TiDB already supports pushing down and how to prohibit specific expressions from being pushed down using blocklist.

TiFlash also supports pushdown for the functions and operators [listed on this page](/tiflash/tiflash-supported-pushdown-calculations.md).

## Supported expressions for pushdown to TiKV

| Expression Type | Operations |
| :-------------- | :------------------------------------- |
| [Logical operators](/functions-and-operators/operators.md#logical-operators) | AND (&&), OR (&#124;&#124;), NOT (!), XOR |
| [Bit operators](/functions-and-operators/operators.md#operators) | [&](https://dev.mysql.com/doc/refman/5.7/en/bit-functions.html#operator_bitwise-and), [~](https://dev.mysql.com/doc/refman/5.7/en/bit-functions.html#operator_bitwise-invert), [\|](https://dev.mysql.com/doc/refman/5.7/en/bit-functions.html#operator_bitwise-or), [`^`](https://dev.mysql.com/doc/refman/5.7/en/bit-functions.html#operator_bitwise-xor), [`<<`](https://dev.mysql.com/doc/refman/5.7/en/bit-functions.html#operator_left-shift), [`>>`](https://dev.mysql.com/doc/refman/5.7/en/bit-functions.html#operator_right-shift) |
| [Comparison functions and operators](/functions-and-operators/operators.md#comparison-functions-and-operators) | [`<`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_less-than), [`<=`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_less-than-or-equal), [`=`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_equal), [`!= (<>)`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_not-equal), [`>`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_greater-than), [`>=`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_greater-than-or-equal), [`<=>`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_equal-to), [BETWEEN ... AND ...](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_between), [COALESCE()](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#function_coalesce), [IN()](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_in), [INTERVAL()](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#function_interval), [IS NOT NULL](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_is-not-null), [IS NOT](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_is-not), [IS NULL](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_is-null), [IS](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_is), [ISNULL()](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#function_isnull), [LIKE](https://dev.mysql.com/doc/refman/5.7/en/string-comparison-functions.html#operator_like), [NOT BETWEEN ... AND ...](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_not-between), [NOT IN()](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_not-in), [NOT LIKE](https://dev.mysql.com/doc/refman/5.7/en/string-comparison-functions.html#operator_not-like), [STRCMP()](https://dev.mysql.com/doc/refman/5.7/en/string-comparison-functions.html#function_strcmp) |
| [Numeric functions and operators](/functions-and-operators/numeric-functions-and-operators.md) | [+](https://dev.mysql.com/doc/refman/5.7/en/arithmetic-functions.html#operator_plus), [-](https://dev.mysql.com/doc/refman/5.7/en/arithmetic-functions.html#operator_minus), [*](https://dev.mysql.com/doc/refman/5.7/en/arithmetic-functions.html#operator_times), [/](https://dev.mysql.com/doc/refman/5.7/en/arithmetic-functions.html#operator_divide), [DIV](https://dev.mysql.com/doc/refman/5.7/en/arithmetic-functions.html#operator_div), [% (MOD)](https://dev.mysql.com/doc/refman/5.7/en/arithmetic-functions.html#operator_mod), [-](https://dev.mysql.com/doc/refman/5.7/en/arithmetic-functions.html#operator_unary-minus), [ABS()](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_abs), [ACOS()](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_acos), [ASIN()](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_asin), [ATAN()](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_atan), [ATAN2(), ATAN()](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_atan2), [CEIL()](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_ceil), [CEILING()](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_ceiling), [CONV()](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_conv), [COS()](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_cos), [COT()](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_cot), [CRC32()](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_crc32), [DEGREES()](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_degrees), [EXP()](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_exp), [FLOOR()](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_floor), [LN()](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_ln), [LOG()](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_log), [LOG10()](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_log10), [LOG2()](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_log2), [MOD()](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_mod), [PI()](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_pi), [POW()](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_pow), [POWER()](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_power), [RADIANS()](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_radians), [RAND()](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_rand), [ROUND()](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_round), [SIGN()](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_sign), [SIN()](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_sin), [SQRT()](https://dev.mysql.com/doc/refman/5.7/en/mathematical-functions.html#function_sqrt) |
| [Control flow functions](/functions-and-operators/control-flow-functions.md) | [CASE](https://dev.mysql.com/doc/refman/5.7/en/flow-control-functions.html#operator_case), [IF()](https://dev.mysql.com/doc/refman/5.7/en/flow-control-functions.html#function_if), [IFNULL()](https://dev.mysql.com/doc/refman/5.7/en/flow-control-functions.html#function_ifnull) |
| [JSON functions](/functions-and-operators/json-functions.md) | [JSON_ARRAY([val[, val] ...])](https://dev.mysql.com/doc/refman/5.7/en/json-creation-functions.html#function_json-array),<br/> [JSON_CONTAINS(target, candidate[, path])](https://dev.mysql.com/doc/refman/5.7/en/json-search-functions.html#function_json-contains),<br/> [JSON_EXTRACT(json_doc, path[, path] ...)](https://dev.mysql.com/doc/refman/5.7/en/json-search-functions.html#function_json-extract),<br/> [JSON_INSERT(json_doc, path, val[, path, val] ...)](https://dev.mysql.com/doc/refman/5.7/en/json-modification-functions.html#function_json-insert),<br/> [JSON_LENGTH(json_doc[, path])](https://dev.mysql.com/doc/refman/5.7/en/json-attribute-functions.html#function_json-length),<br/> [JSON_MERGE(json_doc, json_doc[, json_doc] ...)](https://dev.mysql.com/doc/refman/5.7/en/json-modification-functions.html#function_json-merge),<br/> [JSON_OBJECT([key, val[, key, val] ...])](https://dev.mysql.com/doc/refman/5.7/en/json-creation-functions.html#function_json-object),<br/> [JSON_REMOVE(json_doc, path[, path] ...)](https://dev.mysql.com/doc/refman/5.7/en/json-modification-functions.html#function_json-remove),<br/> [JSON_REPLACE(json_doc, path, val[, path, val] ...)](https://dev.mysql.com/doc/refman/5.7/en/json-modification-functions.html#function_json-replace),<br/> [JSON_SET(json_doc, path, val[, path, val] ...)](https://dev.mysql.com/doc/refman/5.7/en/json-modification-functions.html#function_json-set),<br/> [JSON_TYPE(json_val)](https://dev.mysql.com/doc/refman/5.7/en/json-attribute-functions.html#function_json-type),<br/> [JSON_UNQUOTE(json_val)](https://dev.mysql.com/doc/refman/5.7/en/json-modification-functions.html#function_json-unquote),<br/> [JSON_VALID(val)](https://dev.mysql.com/doc/refman/5.7/en/json-attribute-functions.html#function_json-valid) |
| [Date and time functions](/functions-and-operators/date-and-time-functions.md) | [DATE()](https://dev.mysql.com/doc/refman/5.7/en/date-and-time-functions.html#function_date), [DATE_FORMAT()](https://dev.mysql.com/doc/refman/5.7/en/date-and-time-functions.html#function_date-format), [DATEDIFF()](https://dev.mysql.com/doc/refman/5.7/en/date-and-time-functions.html#function_datediff), [DAYOFMONTH()](https://dev.mysql.com/doc/refman/5.7/en/date-and-time-functions.html#function_dayofmonth), [DAYOFWEEK()](https://dev.mysql.com/doc/refman/5.7/en/date-and-time-functions.html#function_dayofweek), [DAYOFYEAR()](https://dev.mysql.com/doc/refman/5.7/en/date-and-time-functions.html#function_dayofyear), [FROM_DAYS()](https://dev.mysql.com/doc/refman/5.7/en/date-and-time-functions.html#function_from-days), [HOUR()](https://dev.mysql.com/doc/refman/5.7/en/date-and-time-functions.html#function_hour), [MAKEDATE()](https://dev.mysql.com/doc/refman/5.7/en/date-and-time-functions.html#function_makedate), [MAKETIME()](https://dev.mysql.com/doc/refman/5.7/en/date-and-time-functions.html#function_maketime), [MICROSECOND()](https://dev.mysql.com/doc/refman/5.7/en/date-and-time-functions.html#function_microsecond), [MINUTE()](https://dev.mysql.com/doc/refman/5.7/en/date-and-time-functions.html#function_minute), [MONTH()](https://dev.mysql.com/doc/refman/5.7/en/date-and-time-functions.html#function_month), [MONTHNAME()](https://dev.mysql.com/doc/refman/5.7/en/date-and-time-functions.html#function_monthname), [PERIOD_ADD()](https://dev.mysql.com/doc/refman/5.7/en/date-and-time-functions.html#function_period-add), [PERIOD_DIFF()](https://dev.mysql.com/doc/refman/5.7/en/date-and-time-functions.html#function_period-diff), [SEC_TO_TIME()](https://dev.mysql.com/doc/refman/5.7/en/date-and-time-functions.html#function_sec-to-time), [SECOND()](https://dev.mysql.com/doc/refman/5.7/en/date-and-time-functions.html#function_second), [SYSDATE()](https://dev.mysql.com/doc/refman/5.7/en/date-and-time-functions.html#function_sysdate), [TIME_TO_SEC()](https://dev.mysql.com/doc/refman/5.7/en/date-and-time-functions.html#function_time-to-sec), [TIMEDIFF()](https://dev.mysql.com/doc/refman/5.7/en/date-and-time-functions.html#function_timediff), [WEEK()](https://dev.mysql.com/doc/refman/5.7/en/date-and-time-functions.html#function_week), [WEEKOFYEAR()](https://dev.mysql.com/doc/refman/5.7/en/date-and-time-functions.html#function_weekofyear), [YEAR()](https://dev.mysql.com/doc/refman/5.7/en/date-and-time-functions.html#function_year) |
| [String functions](/functions-and-operators/string-functions.md) | [ASCII()](https://dev.mysql.com/doc/refman/5.7/en/string-functions.html#function_ascii), [BIT_LENGTH()](https://dev.mysql.com/doc/refman/5.7/en/string-functions.html#function_bit-length), [CHAR()](https://dev.mysql.com/doc/refman/5.7/en/string-functions.html#function_char), [CHAR_LENGTH()](https://dev.mysql.com/doc/refman/5.7/en/string-functions.html#function_char-length), [CONCAT()](https://dev.mysql.com/doc/refman/5.7/en/string-functions.html#function_concat), [CONCAT_WS()](https://dev.mysql.com/doc/refman/5.7/en/string-functions.html#function_concat-ws), [ELT()](https://dev.mysql.com/doc/refman/5.7/en/string-functions.html#function_elt), [FIELD()](https://dev.mysql.com/doc/refman/5.7/en/string-functions.html#function_field), [HEX()](https://dev.mysql.com/doc/refman/5.7/en/string-functions.html#function_hex), [LENGTH()](https://dev.mysql.com/doc/refman/5.7/en/string-functions.html#function_length), [LIKE](https://dev.mysql.com/doc/refman/5.7/en/string-comparison-functions.html#operator_like), [LTRIM()](https://dev.mysql.com/doc/refman/5.7/en/string-functions.html#function_ltrim), [MID()](https://dev.mysql.com/doc/refman/5.7/en/string-functions.html#function_mid), [NOT LIKE](https://dev.mysql.com/doc/refman/5.7/en/string-comparison-functions.html#operator_not-like), [NOT REGEXP](https://dev.mysql.com/doc/refman/5.7/en/regexp.html#operator_not-regexp), [REGEXP](https://dev.mysql.com/doc/refman/5.7/en/regexp.html#operator_regexp), [REGEXP_INSTR ()](https://dev.mysql.com/doc/refman/8.0/en/regexp.html#function_regexp-instr), [REGEXP_LIKE()](https://dev.mysql.com/doc/refman/8.0/en/regexp.html#function_regexp-like), [REGEXP_REPLACE()](https://dev.mysql.com/doc/refman/8.0/en/regexp.html#function_regexp-replace), [REGEXP_SUBSTR()](https://dev.mysql.com/doc/refman/8.0/en/regexp.html#function_regexp-substr), [REPLACE()](https://dev.mysql.com/doc/refman/5.7/en/string-functions.html#function_replace), [REVERSE()](https://dev.mysql.com/doc/refman/5.7/en/string-functions.html#function_reverse), [RIGHT()](https://dev.mysql.com/doc/refman/5.7/en/string-functions.html#function_right), [RLIKE](https://dev.mysql.com/doc/refman/5.7/en/regexp.html#operator_regexp), [RTRIM()](https://dev.mysql.com/doc/refman/5.7/en/string-functions.html#function_rtrim), [SPACE()](https://dev.mysql.com/doc/refman/5.7/en/string-functions.html#function_space), [STRCMP()](https://dev.mysql.com/doc/refman/5.7/en/string-comparison-functions.html#function_strcmp), [SUBSTR()](https://dev.mysql.com/doc/refman/5.7/en/string-functions.html#function_substr), [SUBSTRING()](https://dev.mysql.com/doc/refman/5.7/en/string-functions.html#function_substring) |
| [Aggregation functions](/functions-and-operators/aggregate-group-by-functions.md#aggregate-group-by-functions) | [COUNT()](https://dev.mysql.com/doc/refman/5.7/en/aggregate-functions.html#function_count), [COUNT(DISTINCT)](https://dev.mysql.com/doc/refman/5.7/en/aggregate-functions.html#function_count-distinct), [SUM()](https://dev.mysql.com/doc/refman/5.7/en/aggregate-functions.html#function_sum), [AVG()](https://dev.mysql.com/doc/refman/5.7/en/aggregate-functions.html#function_avg), [MAX()](https://dev.mysql.com/doc/refman/5.7/en/aggregate-functions.html#function_max), [MIN()](https://dev.mysql.com/doc/refman/5.7/en/aggregate-functions.html#function_min), [VARIANCE()](https://dev.mysql.com/doc/refman/5.7/en/aggregate-functions.html#function_variance), [VAR_POP()](https://dev.mysql.com/doc/refman/5.7/en/aggregate-functions.html#function_var-pop), [STD()](https://dev.mysql.com/doc/refman/5.7/en/aggregate-functions.html#function_std), [STDDEV()](https://dev.mysql.com/doc/refman/5.7/en/aggregate-functions.html#function_stddev), [STDDEV_POP](https://dev.mysql.com/doc/refman/5.7/en/aggregate-functions.html#function_stddev-pop), [VAR_SAMP()](https://dev.mysql.com/doc/refman/5.7/en/aggregate-functions.html#function_var-samp), [STDDEV_SAMP()](https://dev.mysql.com/doc/refman/5.7/en/aggregate-functions.html#function_stddev-samp), [JSON_ARRAYAGG(key)](https://dev.mysql.com/doc/refman/5.7/en/aggregate-functions.html#function_json-arrayagg), [JSON_OBJECTAGG(key, value)](https://dev.mysql.com/doc/refman/5.7/en/aggregate-functions.html#function_json-objectagg) |
| [Encryption and compression functions](/functions-and-operators/encryption-and-compression-functions.md#encryption-and-compression-functions) | [MD5()](https://dev.mysql.com/doc/refman/5.7/en/encryption-functions.html#function_md5), [SHA1(), SHA()](https://dev.mysql.com/doc/refman/5.7/en/encryption-functions.html#function_sha1), [UNCOMPRESSED_LENGTH()](https://dev.mysql.com/doc/refman/5.7/en/encryption-functions.html#function_uncompressed-length) |
| [Cast functions and operators](/functions-and-operators/cast-functions-and-operators.md#cast-functions-and-operators) | [CAST()](https://dev.mysql.com/doc/refman/5.7/en/cast-functions.html#function_cast), [CONVERT()](https://dev.mysql.com/doc/refman/5.7/en/cast-functions.html#function_convert) |
| [Miscellaneous functions](/functions-and-operators/miscellaneous-functions.md#supported-functions) | [UUID()](https://dev.mysql.com/doc/refman/5.7/en/miscellaneous-functions.html#function_uuid) |

## Blocklist specific expressions

If unexpected behavior occurs in the calculation process when pushing down the [supported expressions](#supported-expressions-for-pushdown-to-tikv) or specific data types (**only** the [`ENUM` type](/data-type-string.md#enum-type) and the [`BIT` type](/data-type-numeric.md#bit-type)), you can restore the application quickly by prohibiting the pushdown of the corresponding functions, operators, or data types. Specifically, you can prohibit the functions, operators, or data types from being pushed down by adding them to the blocklist `mysql.expr_pushdown_blacklist`. For details, refer to [Add to the blocklist](/blocklist-control-plan.md#disable-the-pushdown-of-specific-expressions).