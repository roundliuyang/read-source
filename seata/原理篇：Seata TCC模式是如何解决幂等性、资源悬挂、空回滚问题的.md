# 原理篇：Seata TCC模式是如何解决幂等性、资源悬挂、空回滚问题的



摘自：https://developer.aliyun.com/article/1116100

> **简介：** 原理篇：Seata TCC模式是如何解决幂等性、资源悬挂、空回滚问题的



## 前言



在之前的博客中，我们讲到TCC模式中，如果在@TwoPhaseBusinessAction注解中设置useTCCFence=true，那么Seata会帮助开发人员处理幂等性、资源悬挂、空回滚等问题，那么我们这篇文章来看看Seata TCC是如何解决这三个问题的。



## 什么时候触发TCCFence？



**一阶段**

**ActionInterceptorHandler.proceed():**

```java
// 如果@TwoPhaseBusinessAction注解设置了useTCCFence=true
if (businessAction.useTCCFence()) {
    try {
        // 调用TCCFenceHandler.prepareFence()
        return TCCFenceHandler.prepareFence(xid, Long.valueOf(branchId), actionName, targetCallback);
    } catch (SkipCallbackWrapperException | UndeclaredThrowableException e) {
        Throwable originException = e.getCause();
        if (originException instanceof FrameworkException) {
            LOGGER.error("[{}] prepare TCC fence error: {}", xid, originException.getMessage());
        }
        throw originException;
    }
}
```

在Seata准备执行一阶段资源预留前，会判断是否设置了`useTCCFence=true`，如果设置了`useTCCFence=true`，那么需要交给`TCCFenceHandler.prepareFence()`来处理；



**二阶段**

**TCCResourceManager.branchCommit():**

```java
// 判断是否useTCCFence=true
if (Boolean.TRUE.equals(businessActionContext.getActionContext(Constants.USE_TCC_FENCE))) {
    try {
        // 调用TCCFenceHandler.commitFence()执行提交
        result = TCCFenceHandler.commitFence(commitMethod, targetTCCBean, xid, branchId, args);
    } catch (SkipCallbackWrapperException | UndeclaredThrowableException e) {
        throw e.getCause();
    }
}
```



**TCCResourceManager.branchRollback():**

```java
// 判断是否useTCCFence=true
if (Boolean.TRUE.equals(businessActionContext.getActionContext(Constants.USE_TCC_FENCE))) {
    try {
        // 调用TCCFenceHandler.rollbackFence()执行回滚
        result = TCCFenceHandler.rollbackFence(rollbackMethod, targetTCCBean, xid, branchId,
                args, tccResource.getActionName());
    } catch (SkipCallbackWrapperException | UndeclaredThrowableException e) {
        throw e.getCause();
    }
}
```

在RM调用分支事务提交或回滚前，都会检查是否设置了`useTCCFence=true`，如果`useTCCFence=true`，那么就要交给`TCCFenceHandler`来增强提交或回滚逻辑；



## TCCFenceHandler如何实现幂等性



**一阶段**
TCCFenceHandler.prepareFence():

```java
Connection conn = DataSourceUtils.getConnection(dataSource);
// 数据库添加一条有状态的记录，状态为`TCCFenceConstant.STATUS_TRIED`;
boolean result = insertTCCFenceLog(conn, xid, branchId, actionName, TCCFenceConstant.STATUS_TRIED);
LOGGER.info("TCC fence prepare result: {}. xid: {}, branchId: {}", result, xid, branchId);
if (result) {
    // 调用资源预留逻辑
    return targetCallback.execute();
} else {
    throw new TCCFenceException(String.format("Insert tcc fence record error, prepare fence failed. xid= %s, branchId= %s", xid, branchId),
            FrameworkErrorCode.InsertRecordError);
}
```



**二阶段**

**TCCFenceHandler.commitFence():**

```java
public static boolean commitFence(Method commitMethod, Object targetTCCBean,
                                      String xid, Long branchId, Object[] args) {
        // 模版模式
        return transactionTemplate.execute(status -> {
            try {
                Connection conn = DataSourceUtils.getConnection(dataSource);
                // 查询一阶段提交的记录
                TCCFenceDO tccFenceDO = TCC_FENCE_DAO.queryTCCFenceDO(conn, xid, branchId);
                // 如果没查到一阶段的记录，说明还不能调用提交逻辑
                if (tccFenceDO == null) {
                    throw new TCCFenceException(String.format("TCC fence record not exists, commit fence method failed. xid= %s, branchId= %s", xid, branchId),
                            FrameworkErrorCode.RecordAlreadyExists);
                }
                // 如果记录的状态为`TCCFenceConstant.STATUS_COMMITTED`，说明已经提交过了，直接返回true；这就满足了幂等性；
                if (TCCFenceConstant.STATUS_COMMITTED == tccFenceDO.getStatus()) {
                    LOGGER.info("Branch transaction has already committed before. idempotency rejected. xid: {}, branchId: {}, status: {}", xid, branchId, tccFenceDO.getStatus());
                    return true;
                }
                // 如果状态是`TCCFenceConstant.STATUS_ROLLBACKED`，说明已经调用了回滚逻辑，那就不能执行提交；如果是`TCCFenceConstant.STATUS_SUSPENDED`，那就说明该分布式事务暂时挂起了，还不能执行提交；
                if (TCCFenceConstant.STATUS_ROLLBACKED == tccFenceDO.getStatus() || TCCFenceConstant.STATUS_SUSPENDED == tccFenceDO.getStatus()) {
                    if (LOGGER.isWarnEnabled()) {
                        LOGGER.warn("Branch transaction status is unexpected. xid: {}, branchId: {}, status: {}", xid, branchId, tccFenceDO.getStatus());
                    }
                    return false;
                }
                // 这里就可以执行提交逻辑，并修改状态为`TCCFenceConstant.STATUS_COMMITTED`;
                return updateStatusAndInvokeTargetMethod(conn, commitMethod, targetTCCBean, xid, branchId, TCCFenceConstant.STATUS_COMMITTED, status, args);
            } catch (Throwable t) {
                status.setRollbackOnly();
                throw new SkipCallbackWrapperException(t);
            }
        });
    }
```

>1.在提交前会先查询出一阶段插入的记录，如果没有查询到记录，说明一阶段还没有完成，那么就不能执行提交逻辑；
>
>2.如果查询到记录状态是`TCCFenceConstant.STATUS_COMMITTED`，说明已经执行过提交逻辑了，那么就不会再次执行提交逻辑，也就保证了提交的幂等性；
>
>3.如果是其他状态，比如已经回滚了，或者被挂起了，那么就不能执行提交逻辑；
>
>4.所有不可提交的状态都排除后，开始正式执行提交逻辑，并更新状态为`TCCFenceConstant.STATUS_COMMITTED`，表示已经提交；



**TCCFenceHandler.rollbackFence():**

```java
public static boolean rollbackFence(Method rollbackMethod, Object targetTCCBean,
                                        String xid, Long branchId, Object[] args, String actionName) {
        return transactionTemplate.execute(status -> {
            try {
                Connection conn = DataSourceUtils.getConnection(dataSource);
                // 查询一阶段插入的记录
                TCCFenceDO tccFenceDO = TCC_FENCE_DAO.queryTCCFenceDO(conn, xid, branchId);
                // 如果没查询到记录，说明一阶段还没执行
                if (tccFenceDO == null) {
                    // 如果一阶段没执行，就来执行回滚逻辑，说明出现空回滚情况，那么就插入一条记录，防止一阶段在回滚后再执行；
                    boolean result = insertTCCFenceLog(conn, xid, branchId, actionName, TCCFenceConstant.STATUS_SUSPENDED);
                    LOGGER.info("Insert tcc fence record result: {}. xid: {}, branchId: {}", result, xid, branchId);
                    // 如果插入失败，说明一阶段刚刚执行成功了，那么抛出异常，等待下次重试；
                    if (!result) {
                        throw new TCCFenceException(String.format("Insert tcc fence record error, rollback fence method failed. xid= %s, branchId= %s", xid, branchId),
                                FrameworkErrorCode.InsertRecordError);
                    }
                    // 如果插入成功，直接返回，一阶段也不会执行成功；
                    return true;
                } else {
                    // 如果是`TCCFenceConstant.STATUS_ROLLBACKED`状态，说明已经回滚过了；如果是`TCCFenceConstant.STATUS_SUSPENDED`，也不允许执行回滚逻辑；
                    if (TCCFenceConstant.STATUS_ROLLBACKED == tccFenceDO.getStatus() || TCCFenceConstant.STATUS_SUSPENDED == tccFenceDO.getStatus()) {
                        LOGGER.info("Branch transaction had already rollbacked before, idempotency rejected. xid: {}, branchId: {}, status: {}", xid, branchId, tccFenceDO.getStatus());
                        return true;
                    }
                    // 如果是`TCCFenceConstant.STATUS_COMMITTED`状态，说明已经提交了，那么也不能回滚；
                    if (TCCFenceConstant.STATUS_COMMITTED == tccFenceDO.getStatus()) {
                        if (LOGGER.isWarnEnabled()) {
                            LOGGER.warn("Branch transaction status is unexpected. xid: {}, branchId: {}, status: {}", xid, branchId, tccFenceDO.getStatus());
                        }
                        return false;
                    }
                }
                // 执行回滚逻辑，并修改状态为`TCCFenceConstant.STATUS_ROLLBACKED`；
                return updateStatusAndInvokeTargetMethod(conn, rollbackMethod, targetTCCBean, xid, branchId, TCCFenceConstant.STATUS_ROLLBACKED, status, args);
            } catch (Throwable t) {
                status.setRollbackOnly();
                throw new SkipCallbackWrapperException(t);
            }
        });
    }
```

>1.回滚前会查询一阶段保存的记录，如果没有查询到一阶段保存的记录，那么说明出现了空回滚的情况；
>
>2.出现空回滚情况就立马插入一条记录，防止一阶段插入记录成功，也就预防了资源悬挂；
>
>3.如果空回滚能够插入记录，那么一阶段必然执行失败，不会出现资源悬挂问题，也就说明什么都没做，回滚成功；如果空回滚插入记录失败，说明一阶段刚刚执行成功，那么我们直接抛异常，回滚失败，等待下次重试；
>
>4.如果能够查到一阶段的记录，并且状态是`TCCFenceConstant.STATUS_ROLLBACKED`或`TCCFenceConstant.STATUS_SUSPENDED`，说明已经执行过回滚动作了，那么就不能再执行了，直接返回，满足幂等性；
>
>5.如果一阶段记录的状态是`TCCFenceConstant.STATUS_COMMITTED`，说明已经执行过提交逻辑了，那么就不能再回滚了；
>
>6.如果满足执行回滚的条件，那么调用回滚逻辑，并修改状态为`TCCFenceConstant.STATUS_ROLLBACKED`，完成回滚动作；



## 小结



通过以上源码分析，我们可以大概总结为以下几点：

1.开发人员需要在使用TCC模式的时候，在`@TwoPhaseBusinessAction`注解中设置`useTCCFence=true`才能使用Seata提供的功能处理幂等性、资源悬挂、空回滚等问题；

2.Seata解决这三个问题的方式是建立一张记录表，通过记录一阶段、二阶段处理的状态让各处理逻辑感知事务所处的状态，从而达到接口幂等性、避免资源悬挂、空回滚等问题；

3.开发人员需要额外再补充一张记录表`tcc_fence_log`，该数据表建表语句如下：

```sql
-- ----------------------------
-- Table structure for tcc_fence_log
-- ----------------------------
DROP TABLE IF EXISTS `tcc_fence_log`;
CREATE TABLE `tcc_fence_log` (
  `branch_id` bigint NOT NULL COMMENT 'branch transaction id',
  `xid` varchar(128) NOT NULL COMMENT 'global transaction id',
  `action_name` varchar(128) NOT NULL COMMENT 'TCC Action name',
  `status` int NOT NULL COMMENT '1:STATUS_TRIED,2:STATUS_COMMITTED,3:STATUS_ROLLBACKED,4:STATUS_SUSPENDED',
  `gmt_create` datetime(6) NOT NULL COMMENT 'create datetime',
  `gmt_modified` datetime(6) NOT NULL COMMENT 'modify datetime',
  UNIQUE KEY `ux_tcc_fence_log` (`xid`,`branch_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='TCC transaction mode tcc fence log table';
```









