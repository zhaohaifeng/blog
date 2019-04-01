---
layout: post
title: "项目遇到的一个gaplock死锁问题"
date: 2014-08-10
category : 数据库
tags: [db,数据库,deadlock,gap]
---
>前几天的中台项目遇到了一个诡异的死锁问题。所以就研究了一下。

<!-- more -->

## 死锁的场景

先来看看出现死锁的代码：
``` java
    @Override
    public int executeClaim(Integer userId) {
    	if (!canClaim(userId)) {
    		return -1;
    	}
    	Integer bizAccountId = dealTaskPoolMapper.findOneTaskBizAccountId();
    	if (bizAccountId == null) {
    		LOGGER.info("Task pool is empty");
    		Date now = new Date();
    		Date beginEndTime = DateUtils.truncate(now, Calendar.DATE);
    		Date lastEndTime = DateUtils.addDays(DateUtils.truncate(now, Calendar.DATE),
    				Constants.BEFORE_DAYS_OFF_LINE_AS_TASK + 1);
    		bizAccountId = dealTaskMapper.findDealTaskBizAccountIdForClaimPool(beginEndTime, lastEndTime);
    		addTaskToPoolIfNotFull();
    	}
    	if (bizAccountId == null) {
    		return 0;
    	}
    	...........
    		return count;
    }
```
应用做了一个任务队列池，用户认领的时候，先检查一下任务池中有没有，如果有，去处，如果没有，则往任务池中批量添加20个任务，再查找，执行处理。

为了防止在并发时，多个人认领同一个任务，所以是在数据库层用for update做了加锁处理。
``` sql
    <select id="findOneTaskBizAccountId" resultType="int">
    select biz_account_id from mt_task_claim_pool
    order by id asc
    limit 1
    for update
    </select>
    <!--这是往队列池中插入任务的sql-->
   	<insert id="insert" >
    	replace into mt_task_claim_pool (biz_account_id,created_time)
    values (#{bizAccountId,}, #{createdTime})
    </insert>
```
然后系统就开始爆出死锁异常。频率非常低。
在这里实际上会出现两种死锁。先说第一种吧。

## 死锁的重现

由于没有线上数据库相应的权限，所以我只能在线下猜测模拟。所幸要重现这个死锁并非难事：
如果任务池为空，并且同时有两个人认领时，就会死锁
![pic_1]({{ site.img_url }}/D34C8BCA-CFCF-4D48-8E3D-B2B2769D0360.png)
查看死锁日志：
``` sql
    2014-08-16 22:50:47 1301d4000
    *** (1) TRANSACTION:
    TRANSACTION 3857, ACTIVE 27 sec inserting
    mysql tables in use 1, locked 1
    LOCK WAIT 3 lock struct(s), heap size 376, 2 row lock(s)
    MySQL thread id 234, OS thread handle 0x130217000, query id 577 localhost root update
    replace into mt_task_claim_pool (biz_account_id,created_time) values (1,now())
    *** (1) WAITING FOR THIS LOCK TO BE GRANTED:
    RECORD LOCKS space id 7 page no 3 n bits 72 index `PRIMARY` of table `test`.`mt_task_claim_pool` trx id 3857 lock_mode X insert intention waiting
    Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
     0: len 8; hex 73757072656d756d; asc supremum;;
    *** (2) TRANSACTION:
    TRANSACTION 3858, ACTIVE 25 sec inserting
    mysql tables in use 1, locked 1
    3 lock struct(s), heap size 376, 2 row lock(s)
    MySQL thread id 237, OS thread handle 0x1301d4000, query id 586 localhost root update
    replace into mt_task_claim_pool (biz_account_id,created_time) values (2,now())
    *** (2) HOLDS THE LOCK(S):
    RECORD LOCKS space id 7 page no 3 n bits 72 index `PRIMARY` of table `test`.`mt_task_claim_pool` trx id 3858 lock_mode X
    Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
     0: len 8; hex 73757072656d756d; asc supremum;;
    *** (2) WAITING FOR THIS LOCK TO BE GRANTED:
    RECORD LOCKS space id 7 page no 3 n bits 72 index `PRIMARY` of table `test`.`mt_task_claim_pool` trx id 3858 lock_mode X insert intention waiting
    Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
     0: len 8; hex 73757072656d756d; asc supremum;;
    *** WE ROLL BACK TRANSACTION (2)
```

死锁日志有些杂乱无章，这里强烈推荐一下Percona的一个mysql工具：pt-deadlock-logger（其它的一些工具也超级好用，大家可以上它的官网关注一下http://www.percona.com/doc/percona-toolkit/2.2/），输出日志（不如死锁日志详细）：

localhost 2014-08-16T22:50:47 234 0 27 root localhost test mt_task_claim_pool PRIMARY RECORD X w 0 replace into mt_task_claim_pool (biz_account_id,created_time) values (1,now())

localhost 2014-08-16T22:50:47 237 0 25 root localhost test mt_task_claim_pool PRIMARY RECORD X w 1 replace into mt_task_claim_pool (biz_account_id,created_time) values (2,now())

通过这里我们可以看出，事务一二都持有一个Xlock，在等待一个插入意向锁。所以死锁。

问题来了：既然select ... for update是一个当前读，那么就是排他的，事务1操作之后，事务2应该无法执行这个sql了，为什么可以并发执行呢？

既然事务2已经有xLock了，为什么还是无法获得一个插入意向锁呢？

##死锁原因

只有当该表mt_task_claim_pool没数据的时候，才会出现上述死锁。那么在执行select ... for update时，实际上是无法锁定到具体行，无法加行锁，只能加gap锁。

gap锁是不冲突的，所以这两个事务都加了gap锁，后续事务1插入数据，发现事务2加了gap锁，那么等待。事务2插入数据，发现事务1加了gap锁，等待。这就出现死锁了。

如果是gap锁的话，lock type应该是gap，为什么显示是RECORD X（行锁）呢？

这个问题我也思考了很久，后来偶然发现：

在事务1执行插入操作后，等待，这是查看lock情况：
```
    mysql> select * from information_schema.innodb_locks;
    +------------+-------------+-----------+-----------+-----------------------------+------------+------------+-----------+----------+------------------------+
    | lock_id    | lock_trx_id | lock_mode | lock_type | lock_table                  | lock_index | lock_space | lock_page | lock_rec | lock_data              |
    +------------+-------------+-----------+-----------+-----------------------------+------------+------------+-----------+----------+------------------------+
    | 4857:7:3:1 | 4857        | X         | RECORD    | `test`.`mt_task_claim_pool` | PRIMARY    |          7 |         3 |        1 | supremum pseudo-record |
    | 4858:7:3:1 | 4858        | X         | RECORD    | `test`.`mt_task_claim_pool` | PRIMARY    |          7 |         3 |        1 | supremum pseudo-record |
    +------------+-------------+-----------+-----------+-----------------------------+------------+------------+-----------+----------+------------------------+
```

lock_data是supremum pseudo-record，这是什么呢？查看mysql 的官方doc：http://dev.mysql.com/doc/refman/5.5/en/innodb-record-level-locks.html，其中有一句：

>For the last interval, the next-key lock locks the gap above the largest value in the index and the “supremum”pseudo-record having a value higher than any value actually in the index. The supremum is not a real index record, so, in effect, this next-key lock locks only the gap following the largest index value.

可见supremum pseudo-record的X RECORD lock实际上就是gap lock的最大区间那块。当表里没有数据的时候，那么整个表的区间都是gap lock中最大区间了。

## 解决方法

解决方法很简单，只要让这个任务池永远不会空就可以了，但任务还剩1条（或者可以更多一点）的时候就往里面加任务。

至于为什么明明是个gap lock，但是在死锁日志里却显示是RECORD呢？我现在还没有研究出来，难道是bug？还是开发者有什么更深层的想法，菜鸟完全无法揣摩高手的意图嘛！
