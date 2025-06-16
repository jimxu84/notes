
http://mysql.taobao.org/monthly/2015/04/01/

Undo log（回滚日志）是 InnoDB 实现事务原子性(Atomicity)和多版本并发控制(MVCC)的核心机制。下面我将全面深入地介绍 Undo log 的工作原理和实现细节。

### 1. Undo Log 的基本概念

#### 核心作用
- **事务回滚**：记录事务修改前的数据状态，用于回滚操作
- **MVCC 实现**：为读操作提供历史版本数据
- **崩溃恢复**：协助 Redo log 完成崩溃恢复

#### 关键特性
- **逻辑日志**：记录行记录的变化逻辑（不同于 Redo 的物理日志）
- **多版本存储**：维护数据的多个历史版本
- **链式结构**：通过指针形成版本链

### 2. Undo Log 的存储结构

#### 物理存储
- **系统表空间**：MySQL 5.7及之前版本存储在 ibdata1 中
- **独立表空间**：MySQL 8.0+ 支持独立的 undo tablespace
- **回滚段**：默认128个回滚段（MySQL 8.0+），每个段支持1024个事务

#### 内存结构
- **Undo Log Buffer**：undo 页的内存缓冲
- **Undo Log Segment**：回滚段的内存管理结构

#### 日志类型
| 类型 | 描述 | 生命周期 |
|------|------|----------|
| Insert Undo | 记录INSERT操作 | 事务提交后可立即清除 |
| Update Undo | 记录UPDATE/DELETE操作 | 需要支持MVCC，可能保留较长时间 |

### 3. Undo Log 的工作流程

#### 事务修改数据时
1. 在修改数据前，将原始数据拷贝到 Undo log
2. 记录修改类型（INSERT/UPDATE/DELETE）
3. 记录事务ID和回滚指针（DB_ROLL_PTR）
4. 修改数据行的 DB_TRX_ID 和 DB_ROLL_PTR 字段

#### 事务回滚时
1. 根据 Undo log 找到修改前的数据
2. 将数据还原为原始状态
3. 清理对应的 Undo log 记录

#### MVCC 读操作时
1. 通过 DB_ROLL_PTR 找到 Undo log 中的历史版本
2. 根据 ReadView 判断哪些版本对当前事务可见

### 4. Undo Log 的版本链

示例数据变化：
1. 事务101插入记录R
2. 事务102更新记录R
3. 事务103删除记录R

版本链结构：
```
当前记录 → [trx_id=103, del_flag=1, roll_ptr→102版本] 
           → [trx_id=102, roll_ptr→101版本] 
           → [trx_id=101, 原始插入数据]
```

### 5. Undo Log 的清理机制

#### 清理条件
- 没有活跃事务需要访问该版本
- 系统执行 purge 操作时

#### 相关参数
| 参数 | 说明 |
|------|------|
| `innodb_purge_threads` | purge 线程数量（默认4） |
| `innodb_max_purge_lag` | 控制 purge 延迟 |
| `innodb_undo_log_truncate` | 是否启用undo表空间截断 |

#### 清理过程
1. 后台 purge 线程扫描 Undo log
2. 标记可清理的空间
3. 物理释放空间（MySQL 8.0+支持动态收缩）

### 6. Undo Log 的重要参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `innodb_undo_directory` | ./ | undo表空间目录 |
| `innodb_undo_tablespaces` | 0 | undo表空间数量（8.0默认2） |
| `innodb_undo_log_truncate` | OFF | 是否启用undo截断 |
| `innodb_max_undo_log_size` | 1GB | 触发截断的大小阈值 |

### 7. Undo Log 与 Redo Log 的关系

1. **写入顺序**：先写 Undo log 的 Redo log，再写 Undo log 本身
2. **崩溃恢复**：先通过 Redo log 恢复 Undo log，再用 Undo log 回滚未提交事务
3. **空间管理**：Undo log 的空间管理也依赖 Redo log 保证一致性

### 8. Undo Log 的优化实践

1. **监控 Undo 空间使用**
   ```sql
   SELECT tablespace_name, status, size_in_bytes/1024/1024 
   FROM information_schema.innodb_tablespaces 
   WHERE row_format='Undo';
   ```

2. **避免长事务**
   - 长事务会导致 Undo log 无法及时清理
   - 监控长事务：
     ```sql
     SELECT * FROM information_schema.innodb_trx 
     ORDER BY trx_started ASC LIMIT 10;
     ```

3. **合理配置 Undo 表空间**
   - MySQL 8.0+ 建议使用独立 Undo 表空间
   - 设置自动截断：
     ```ini
     innodb_undo_log_truncate=ON
     innodb_max_undo_log_size=1G
     ```

4. **调整 purge 线程**
   - 高并发系统可增加 purge 线程：
     ```ini
     innodb_purge_threads=8
     ```

### 9. Undo Log 的常见问题

#### 空间膨胀
- **现象**：ibdata1 或 undo 表空间不断增长
- **解决方案**：
  - 升级到 MySQL 8.0 使用独立 undo 表空间
  - 避免长事务
  - 定期监控

#### 版本链过长
- **现象**：MVCC 读性能下降
- **解决方案**：
  - 优化事务设计，减少事务持续时间
  - 适当调整隔离级别

#### Purge 延迟
- **现象**：出现 "undo log purge lag" 警告
- **解决方案**：
  - 增加 purge 线程
  - 调整 `innodb_max_purge_lag` 参数

Undo log 是 InnoDB 实现事务特性的关键组件，合理管理和优化 Undo log 对数据库性能和稳定性至关重要。理解其工作原理有助于更好地设计数据库应用和排查相关问题。
