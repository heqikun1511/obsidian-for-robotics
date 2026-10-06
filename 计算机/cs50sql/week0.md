---
tags:
  - 计算机基础
  - sql
---




# Week 0 — Querying（查询）

## 0.1 数据库、表、字段与记录

- 数据库（database）：有组织的数据集合。
    
- 表（table）：同类实体的集合。
    
- 列（column/field）：实体的属性和数据类型。
    
- 行（row/record）：一个具体实体或一条记录。
    
- SQL 是声明式语言：描述“想要什么”，而非逐步规定“怎么取”。
    

SQLite 命令行常用命令：

```
.open database.db
.tables
.schema
.schema table_name
.headers on
.mode column
.quit
```

以点开头的是 SQLite CLI 命令，不是 SQL，结尾通常不写分号。


## 0.2 SELECT 基础

```
SELECT * FROM books;

SELECT title, author
FROM books;

SELECT title AS book_title
FROM books;
```

- `*` 方便探索数据，但正式程序中应尽量明确列名。
    
- `AS` 给列或表达式取临时别名，不会修改原表。
    
- SQL 关键字通常不区分大小写；建议统一大写以增强可读性。
## 0.3 WHERE：筛选行

```
SELECT title
FROM books
WHERE year >= 2020;
```

常用运算符：

| 类型  | 运算符                                   |     |
| --- | ------------------------------------- | --- |
| 比较  | `=`, `!=`, `<>`, `<`, `<=`, `>`, `>=` |     |
| 逻辑  | `AND`, `OR`, `NOT`                    |     |
| 范围  | `BETWEEN ... AND ...`                 |     |
| 集合  | `IN (...)`                            |     |
| 空值  | `IS NULL`, `IS NOT NULL`              |     |
| 模式  | `LIKE`                                |     |

```
SELECT * FROM books
WHERE year BETWEEN 2015 AND 2020;

SELECT * FROM books
WHERE format IN ('hardcover', 'paperback');

SELECT * FROM books
WHERE translator IS NULL;
```

### NULL 不是普通值

`NULL` 表示未知、缺失或不适用。不能写 `column = NULL`：

```
-- 错误思路
WHERE translator = NULL

-- 正确
WHERE translator IS NULL
```

SQL 条件存在三值逻辑：`TRUE`、`FALSE`、`UNKNOWN`。涉及 `NULL` 的普通比较通常得到 `UNKNOWN`。

## 0.4 LIKE 与通配符

```
SELECT title
FROM books
WHERE title LIKE '%robot%';
```

- `%`：任意长度字符串，包括空字符串。
    
- `_`：任意单个字符。
    

```
WHERE name LIKE 'A%';    -- A 开头
WHERE code LIKE '__-__'; -- 两字符、横线、两字符
```

注意：SQLite 默认对 ASCII 字母的 `LIKE` 通常不区分大小写，但不同数据库或排序规则可能不同。

## 0.5 ORDER BY、LIMIT 与 OFFSET

```
SELECT title, rating
FROM books
ORDER BY rating DESC, title ASC
LIMIT 10 OFFSET 20;
```

- 默认升序 `ASC`，降序用 `DESC`。
    
- 多列排序时，后一列用于打破前一列的并列。
    
- 没有 `ORDER BY` 时，数据库不保证返回顺序。
    
- 大偏移量分页可能变慢，实际系统常改用基于游标/主键的分页。

## 0.6 聚合函数

```
SELECT COUNT(*) FROM books;
SELECT AVG(rating) FROM ratings;
SELECT MIN(year), MAX(year) FROM books;
SELECT SUM(quantity) FROM inventory;
```

- `COUNT(*)` 统计行数。
    
- `COUNT(column)` 不统计该列为 `NULL` 的行。
    
- `COUNT(DISTINCT column)` 统计不同的非空值。
    

```
SELECT ROUND(AVG(rating), 2) AS average_rating
FROM ratings;
```


## 0.7 DISTINCT、GROUP BY 与 HAVING

```
SELECT DISTINCT publisher
FROM books;
```

按类别聚合：

```
SELECT publisher_id, COUNT(*) AS book_count
FROM books
GROUP BY publisher_id
HAVING COUNT(*) >= 10
ORDER BY book_count DESC;
```

- `WHERE` 在分组前过滤行。
    
- `HAVING` 在分组后过滤聚合结果。
    

逻辑处理顺序可近似记为：

```
FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT
```