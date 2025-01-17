
**MySQL 死锁** 是指两个或多个事务互相等待对方持有的锁，从而导致所有事务都无法继续执行的现象。在 InnoDB 存储引擎中，死锁是通过锁机制产生的，特别是在并发较高、业务逻辑复杂的情况下，更容易发生死锁。


### 一、MySQL 死锁的成因


MySQL 的死锁一般发生在 **行级锁** 上。常见的死锁成因包括：


1. **事务 A 和事务 B 持有互相需要的锁**：事务 A 锁住了记录 1，事务 B 锁住了记录 2，事务 A 尝试获取记录 2 的锁，而事务 B 试图获取记录 1 的锁，造成了死锁。
2. **不同顺序的锁定**：两个事务对同一组资源请求加锁，但是加锁顺序不同，导致互相等待。例如，事务 A 按照顺序锁定记录 1 和记录 2，而事务 B 以相反的顺序锁定记录 2 和记录 1。
3. **使用了 gap lock (间隙锁)**：在 InnoDB 的 Next\-Key Locking 机制下，间隙锁定也可能导致死锁，尤其是在范围查询时，多个事务试图锁定同一间隙。
4. **长事务和锁等待时间过长**：事务执行时间长，未及时释放锁，造成其他事务等待锁超时或死锁。


### 二、死锁检测与处理


MySQL 使用 **死锁检测** 来处理死锁问题。MySQL 会自动检测事务是否处于死锁状态，并中止其中一个事务，释放锁以允许另一个事务继续执行。InnoDB 存储引擎通过引入死锁检测机制来解决这个问题，当检测到死锁时，会选择一个事务进行回滚，以打破僵局。被回滚的事务会抛出 `Deadlock found when trying to get lock` 错误。


### 三、如何避免和处理 MySQL 的死锁？


#### 1\. **合理设计索引**


使用合适的索引可以减少加锁的范围，降低死锁的发生概率。没有索引时，MySQL 会对表中的所有记录加锁，增加了锁冲突的机会。因此，合理地设计和使用索引，确保查询能够快速找到数据，避免不必要的锁争用，能够显著减少死锁风险。


#### 2\. **保持加锁顺序一致**


事务操作表中的多条记录时，保持一致的加锁顺序可以有效减少死锁问题。例如，如果两个事务都需要加锁相同的资源，确保它们按照相同的顺序请求锁，避免死锁。


#### 3\. **减少事务的锁定时间**


尽量缩短事务的执行时间，减少锁的持有时间。将事务划分为更小的逻辑单元，避免长时间占用资源。同时，将非必要的复杂操作尽量移到事务外执行。


#### 4\. **减少并发度**


在并发较高的情况下，增加锁冲突和死锁的几率较高。可以通过控制并发度来减少锁争用，比如使用乐观锁机制，避免频繁加锁。


#### 5\. **使用表锁替代行锁**


对于一些写操作集中的场景，可以考虑使用表锁替代行锁，以避免行级锁导致的死锁。不过表锁会导致并发性能下降，所以需要根据业务场景选择合适的锁。


#### 6\. **锁定更小的范围**


尽量通过使用主键索引和合适的条件，减少事务锁定的行范围。特别是在 `UPDATE` 或 `DELETE` 操作中，使用精准的查询条件来限制锁的作用范围。


#### 7\. **分批提交事务**


对于批量操作，考虑将大事务拆解成多个小事务，减少一次性加锁的行数和操作范围，减少锁的持有时间。


#### 8\. **选择合适的事务隔离级别**


适当降低事务隔离级别可以减少锁冲突的几率。例如，可以将事务隔离级别从 `Serializable` 调整为 `Read Committed` 或 `Repeatable Read`，来减少行锁定的情况。


#### 9\. **加锁操作使用`SELECT ... FOR UPDATE`**


当你需要在查询数据后立即进行更新时，可以使用 `SELECT ... FOR UPDATE` 来显式地锁定行，避免在更新时再去加锁造成的死锁。


### 四、常见死锁示例


以下是一个常见的死锁示例，两个事务尝试对相同的记录加锁但顺序不同：



```
-- 事务 A
START TRANSACTION;
UPDATE orders SET status = 'shipped' WHERE id = 1; -- 锁住记录 1
-- 此时，事务 B 在等待锁定记录 1

-- 事务 B
START TRANSACTION;
UPDATE orders SET status = 'shipped' WHERE id = 2; -- 锁住记录 2
-- 此时，事务 A 在等待锁定记录 2

-- 事务 A 尝试更新记录 2，但事务 B 持有锁，事务 A 等待
UPDATE orders SET status = 'shipped' WHERE id = 2;

-- 事务 B 尝试更新记录 1，但事务 A 持有锁，事务 B 等待

-- 死锁发生，MySQL 自动检测并回滚其中一个事务

```

### 五、如何检测和分析死锁？


通过以下方式可以检测和分析 MySQL 中的死锁：


#### 1\. **启用 `innodb_print_all_deadlocks` 参数**


通过设置 `innodb_print_all_deadlocks=ON`，可以在 MySQL 日志中输出所有的死锁信息，便于分析和调试。


#### 2\. **使用 SHOW ENGINE INNODB STATUS 命令**


在 MySQL 发生死锁后，可以使用 `SHOW ENGINE INNODB STATUS` 命令查看死锁信息。该命令会输出最近发生的死锁情况，帮助开发者找到死锁的根源。



```
SHOW ENGINE INNODB STATUS\G

```

输出中包含的信息包括：


* 哪个事务被回滚
* 发生死锁时，事务分别持有哪些锁，等待哪些锁
* 事务操作的 SQL 语句


#### 3\. **MySQL 慢查询日志**


开启 MySQL 慢查询日志，也可以间接帮助发现由于锁等待导致的性能问题，虽然不能直接显示死锁，但可以作为锁冲突问题排查的辅助工具。


### 六、死锁后的应对策略


当发生死锁时，MySQL 会自动回滚其中一个事务，开发人员需要捕获并处理这种异常。


在代码中，你可以使用如下方式处理死锁：



```
try {
    // 执行事务
    ...
} catch (SQLException e) {
    if (e.getErrorCode() == 1213) { // 1213 代表死锁错误代码
        // 死锁检测，进行重试
        retryTransaction();
    } else {
        // 其他异常处理
        throw e;
    }
}

```

通过捕获死锁异常并进行适当的重试，系统可以在发生死锁后继续执行，从而提升系统的健壮性。


### 七、总结


MySQL 死锁是数据库在并发场景下常见的问题，特别是对于大规模、复杂的业务系统，死锁问题更为频繁。通过合理的索引设计、保持加锁顺序一致、缩短事务时间、优化锁策略等手段，可以有效减少死锁的发生。同时，当死锁发生时，MySQL 具备死锁检测和自动回滚机制，开发人员可以通过合理的异常处理和重试机制，来提高系统的稳定性和可靠性。


秋是慢入的，但冷却是突然的，晴不知夏去，一雨方觉秋深！上海有点冷了。


 本博客参考[milou加速器](https://jiechuangmoxing.com)。转载请注明出处！
