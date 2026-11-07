[[MYSQL 工程设计问题]]
`ON UPDATE CASCADE` 是外键约束的**参照动作（referential action）**之一，定义当**父表被引用的键值被修改时**，子表该如何响应。

### ⚙️ 含义：级联更新

当父表中被引用的列（如 `sensor.sensor_id`）的值被 `UPDATE` 修改时，数据库会**自动把子表中所有引用该旧值的行，同步更新为新值**，保持关联关系不断开。

### 📋 举例说明

假设有父子两张表：

sql

-- 父表
sensor(sensor_id, ...)          -- sensor_id 主键
-- 子表
sensor_point(sensor_id, ...)    -- 外键 REFERENCES sensor(sensor_id)

现在父表执行：

sql

UPDATE sensor SET sensor_id = 200 WHERE sensor_id = 100;

如果子表有 3 行引用 `sensor_id = 100`，那么：

- **有 `ON UPDATE CASCADE`**：这 3 行的 `sensor_id` 自动变成 `200`，关联保持完好。
    
- **没有该选项（默认 `RESTRICT`/`NO ACTION`）**：父表这次更新会被**拒绝并报错**，因为有子表数据还引用着旧值，防止产生"孤儿引用"。
    

### ⚠️ 注意点

- **只影响"被引用的键值"的变更**：改父表其他非引用列不会触发；改的是被外键引用的那一列才会级联。
    
- **级联可能连锁触发**：如果子表本身又是另一张表的父表，更新可能沿链条继续级联，设计时要留意影响范围。
    
- **与 `ON DELETE CASCADE` 的区别**：`ON UPDATE CASCADE` 管的是"值变了"，`ON DELETE CASCADE` 管的是"行被删了"，两者独立配置。
    
- **主键频繁变更不推荐**：级联更新虽然方便，但主键/唯一键本身应尽量稳定，频繁修改 ID 会带来额外开销和连锁影响。实践中主键一般不改，所以 `ON UPDATE CASCADE` 更多是一种"兜底保险"。
    

### 📋 结合你的上下文

在 `FOREIGN KEY (sensor_id) REFERENCES sensor(sensor_id) ON UPDATE CASCADE` 中，含义是：**如果 `sensor` 表里某个传感器的 `sensor_id` 被改号，所有引用它的测点记录会自动跟着改成新号**，避免子表指向一个已不存在的旧 ID。配合之前的 `ON DELETE RESTRICT`，整体策略是"改号自动跟随，删除严格限制"。