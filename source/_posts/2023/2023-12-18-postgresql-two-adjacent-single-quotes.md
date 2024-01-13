---
title: 如何理解 PostgreSQL 中两个邻近的单引号
date: 2023-12-18 20:09:24
updated: 2023-12-18 20:09:24
tags:
---

在 PostgreSQL 中执行如下的 SQL 语句

```sql
SELECT 'AND c1 = ''' || 5 || '''';
```

将输出 `AND c1 = '5'`。我们的疑问是 `5` 两边的单引号 `'` 是怎么得到的？

<!-- more -->

在官方文档的 [4.1.2.1. String Constants](https://www.postgresql.org/docs/16/sql-syntax-lexical.html#SQL-SYNTAX-CONSTANTS) 一节中给出了字符串的定义

>A string constant in SQL is an arbitrary sequence of characters bounded by single quotes (`'`), for example `'This is a string'`. To include a single-quote character within a string constant, write two adjacent single quotes, e.g., `'Dianne''s horse'`.

同时我们从官法文档的 [9.4. String Functions and Operators](https://www.postgresql.org/docs/16/functions-string.html#FUNCTIONS-STRING) 一节中知道字符串串接操作符 `||` 可以把两个字符串连接起来得到一个新的字符串，它也可以接收一个非字符串输入，比如数字 `5`，此时会自动地把非字符串输入转换为字符串再进行串接。

现在再来看前面的 SQL 语句

{% asset_img string.drawio.svg %}

`||` 操作符将 `SELECT` 子句分为了 3 部分。第 ① 和第 ③ 部分是字符串，第 ② 部分是数字。

第 ① 部分字符串的内容是由下标 0 和 下标 12 处的单引号包围的内容，即 `AND c1 = ''`，下标 10 和下标 11 两个邻近的单引号将输出 `5` 左边的单引号。第 ③ 部分字符串的内容是由下标 20 和下标 23 处的单引号包围的内容，即 `''`，下标 21 和下标 22 两个邻近的单引号将输出 `5` 右边的单引号。

我们来看另一个有趣的例子

```sql
SELECT 'AND c1 = ''''''';
```

这个例子将输出 `AND c1 = '''`。除去首尾的单引号，中间的 6 个单引号 `''''''` 每两个邻近的单引号输出一个单引号，我们可以用 `/` 作为分隔符分割一下 `''/''/''`，即第 1 和第 2 个单引号输出结果中的第一个单引号，第 3 和第 4 个单引号输出结果中的第二个单引号，第 5 和第 6 个单引号输出结果中的第三个单引号。
