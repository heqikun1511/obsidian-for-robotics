---
tags:
  - 计算机基础
  - sql
---

[week2]


# Week 3 — Writing（写入）

## 3.1 INSERT

```
INSERT INTO authors (name)
VALUES ('Alice');

INSERT INTO books (title, year)
VALUES
    ('Book A', 2024),
    ('Book B', 2025);
```

显式列出列名，避免表结构变化后把数据写错位置。

从查询结果插入：

```
INSERT INTO archived_books (id, title)
SELECT id, title
FROM books
WHERE year < 2000;
```




## 3.2 UPDATE

```
UPDATE books
SET price = price * 0.9
WHERE publisher_id = 3;
```

执行前先用相同条件查询：

```
SELECT * FROM books WHERE publisher_id = 3;
```

没有 `WHERE` 会更新整张表。

## 3.3 DELETE

```
DELETE FROM books
WHERE id = 42;
```

没有 `WHERE` 会删除所有行，但表结构仍存在。`DROP TABLE` 才会删除表本身。

## 3.4 冲突处理

SQLite 示例：

```
INSERT INTO users (username)
VALUES ('alice')
ON CONFLICT(username) DO NOTHING;
```

UPSERT：

```
INSERT INTO inventory (product_id, quantity)
VALUES (10, 5)
ON CONFLICT(product_id)
DO UPDATE SET quantity = quantity + excluded.quantity;
```


## 3.5 事务

事务将多条语句视为一个整体：要么全部成功，要么全部失败。

```
BEGIN TRANSACTION;

UPDATE accounts
SET balance = balance - 100
WHERE id = 1;

UPDATE accounts
SET balance = balance + 100
WHERE id = 2;

COMMIT;
```

发生错误时：

```
ROLLBACK;
```

### ACID

- Atomicity 原子性：事务不可只完成一半。
    
- Consistency 一致性：事务前后约束仍成立。
    
- Isolation 隔离性：并发事务不会随意互相干扰。
    
- Durability 持久性：提交后结果能够保存。

## 3.6 并发与竞态条件

“先查询余额，再更新余额”若分散在多个事务中，可能发生竞态。解决思路：

- 把相关操作放进同一事务；
    
- 让更新条件直接表达业务约束；
    
- 根据数据库选择锁、隔离级别或乐观并发控制。
    

```
UPDATE accounts
SET balance = balance - 100
WHERE id = 1 AND balance >= 100;
```

随后检查实际受影响行数，判断扣款是否成功。