


### 🔒 `CONSTRAINT uk_transformer_point UNIQUE (transformer_id, point_name)`

这是一个**复合唯一约束**（多列联合唯一）。

- **含义**：`transformer_id` 和 `point_name` 这两个列的**组合值**在整张表中必须唯一。注意不是各自唯一，而是"同一个变压器下，测点名称不能重复"。
    
- **实际效果**：
    
    - 同一台变压器（相同 `transformer_id`）下，不能有两个同名测点（`point_name` 相同）→ 插入会报错。
        
    - 不同的变压器可以有相同名称的测点（如 A 变压器有"温度"点，B 变压器也有"温度"点）→ 允许。
        
- **约束名 `uk_transformer_point`**：`uk` = unique key，命名便于后续维护（如 `DROP INDEX uk_transformer_point`）。
    
- **底层实现**：InnoDB 中通过复合唯一索引实现，同时能加速按 `(transformer_id, point_name)` 组合查询。