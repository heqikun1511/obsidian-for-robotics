---
tags:
  - 计算机基础
  - sql
---
[[week0]]

# Week 1 — Relating（关系）

## 1.1 为什么拆成多张表

若把作者姓名反复写入每本书的记录，会产生：

- 数据冗余；
    
- 修改异常：作者姓名要改很多处；
    
- 插入异常：没有书时无法记录作者；
    
- 删除异常：删掉最后一本书可能连作者信息也丢失。
    

关系型数据库通过主键和外键连接实体。

```
CREATE TABLE authors (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE books (
    id INTEGER PRIMARY KEY,
    title TEXT NOT NULL,
    author_id INTEGER,
    FOREIGN KEY (author_id) REFERENCES authors(id)
);
```

## 1.3 JOIN

### INNER JOIN

仅保留两侧成功匹配的行：

```
SELECT books.title, authors.name
FROM books
JOIN authors ON books.author_id = authors.id;
```

### LEFT JOIN

保留左表全部行；未匹配的右表列填 `NULL`：

```
SELECT authors.name, books.title
FROM authors
LEFT JOIN books ON authors.id = books.author_id;
```

查找没有图书的作者：

```
SELECT authors.name
FROM authors
LEFT JOIN books ON authors.id = books.author_id
WHERE books.id IS NULL;
```

其他连接：

- `RIGHT JOIN`：保留右表全部行；SQLite 新版本才支持。
    
- `FULL OUTER JOIN`：保留两侧全部行；兼容性要检查。
    
- `CROSS JOIN`：笛卡尔积，行数等于两表行数之积。

## 1.4 多表与多对多查询

```
SELECT books.title, authors.name
FROM books
JOIN authored ON books.id = authored.book_id
JOIN authors ON authored.author_id = authors.id
ORDER BY books.title;
```

建议总是显式写 `table.column`，尤其是多张表都含有 `id` 时。


## 1.5 子查询

```
SELECT title
FROM books
WHERE author_id = (
    SELECT id
    FROM authors
    WHERE name = 'Alice'
);
```

子查询可出现在 `SELECT`、`FROM`、`WHERE` 等位置。若子查询可能返回多行，应使用 `IN`、`EXISTS` 或重新设计条件。

```
SELECT name
FROM authors
WHERE EXISTS (
    SELECT 1
    FROM books
    WHERE books.author_id = authors.id
);
```

`EXISTS` 只关心是否存在匹配行，不关心子查询选择了什么列。