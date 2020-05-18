---
title: 事务采坑记录01
date: 2019-09-24 17:29:03
tags: [transaction, 事务, 记录, 原创]
categories: spring
---

[transaction] spring事务踩坑记录

<!-- more --> 

# 故事原由

最近有个同事在使用开放平台透出去自己方法的时候，开放平台返回出来了异常。一开始问我的时候，因为用到了RPC的泛化调用，我和他还扯了一会的泛化的异常处理逻辑。等忙完自己的事情后，我仔细查跟他的代码走了下，发现事务有嵌套关系，突然我就意识到了事务的一个常识，在这里记录下。 

# 问题说明

伪代码如下：

```java
class CService {

    @Autowired
    private BService bService;

    @Autowired
    private AService aService;

    @Override
    public String execute(String value) {
        String result = null;
        try {
            // ......
            result = bService.do();
            aService.do();
            // ......
        } catch (Exception e) {
            e.printStackTrace();
        }

        return result;
    }

}
```

疑惑的点在于我都  ` try...catch `  住了，为何还会出现异常？这个问题通过阅读事务的源码可以让人豁然开朗。

# 复现问题的Demo

定义一个3个Service类，AService、BService、CService，在controller调用CService的方法execute，然后CService里面对数据库做一次更新，再调用BService和AService的方法，其中某个方法里面发生异常。

```java
public class DemoController {

    @Autowired
    private CService cService;

    @GetMapping(value = "transaction-demo")
    public String result(@RequestParam("name") String name) {
        return cService.execute(name);
    }

}
```

```java
@Transactional
@Service
public class CServiceImpl implements CService {

    @Autowired
    private DruidDemoMapper druidDemoMapper;

    @Autowired
    private BService bService;

    @Autowired
    private AService aService;

    @Override
    public String execute(String value) {
        String result = null;
        try {
            DruidDemo druidDemo = new DruidDemo();
            druidDemo.setId(401L);
            druidDemo.setName("tie");
            druidDemo.setAge(18);
            druidDemoMapper.updateByPrimaryKey(druidDemo);
            result = aService.execute(value);
            result = bService.executeError(value);
        } catch (Exception e) {
            e.printStackTrace();
        }

        return result;
    }
```

```java
@Transactional
@Service
public class AServiceImpl implements AService {

    @Override
    public String execute(String value) {
        return "A" + value;
    }
}
```

```java
@Transactional
@Service
public class BServiceImpl implements BService {

    @Override
    public String executeError(String value) {
        int i = 1 / 0;
        return "";
    }

}
```

# 核心源码说明

前置说明
- @Service等注解，意味着会产生一个代理类（目前的版本基于cglib）
- 只分析本次问题的核心代码（补充一次完整的事务调用流程）
- 事务管理器会单独剖析

## status对象

```java
public class DefaultTransactionStatus extends AbstractTransactionStatus {

    // self

	private final Object transaction;

	private final boolean newTransaction;

	private final boolean newSynchronization;

	private final boolean readOnly;

	private final boolean debug;

	private final Object suspendedResources;

    // AbstractTransactionStatus

    private boolean rollbackOnly = false;

	private boolean completed = false;

	private Object savepoint;

    // ......
}    
```

实际运行时如下：
![DefaultTransactionStatus](https://cdn.nlark.com/yuque/0/2019/png/327432/1569381660979-c364f0c7-4a84-46e8-bbd7-2d4d81207f52.png)

## 一个状态的设置

### rollbackOnly

- org.springframework.transaction.support.DefaultTransactionStatus#isGlobalRollbackOnly
- org.springframework.transaction.support.ResourceHolderSupport#isRollbackOnly

```java
public abstract class ResourceHolderSupport implements ResourceHolder {

	private boolean synchronizedWithTransaction = false;

	private boolean rollbackOnly = false;

    // ......

	public boolean isRollbackOnly() {
		return this.rollbackOnly;
	}

}
```

### 设置的地方

- org.springframework.transaction.support.AbstractPlatformTransactionManager#processRollback
- org.springframework.transaction.support.AbstractPlatformTransactionManager#doSetRollbackOnly
- org.springframework.jdbc.datasource.DataSourceTransactionManager.DataSourceTransactionObject#setRollbackOnly
- org.springframework.transaction.support.ResourceHolderSupport#setRollbackOnly

```java
private void processRollback(DefaultTransactionStatus status, boolean unexpected) {
		try {
			boolean unexpectedRollback = unexpected; // 回滚进来的时候是 false

			try {
				// ......
				if (status.hasSavepoint()) {
                    // 这个在内嵌事务的时候有一种使用场景
					// ......
				}
				else if (status.isNewTransaction()) {
					// ......
				}
				else {
					// Participating in larger transaction
					if (status.hasTransaction()) {
						if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
							if (status.isDebug()) {
								logger.debug("Participating transaction failed - marking existing transaction as rollback-only");
							}
							doSetRollbackOnly(status);
						}
						else {
							// ......
						}
					}
					else {
						logger.debug("Should roll back transaction but cannot - no transaction available");
					}
					// Unexpected rollback only matters here if we're asked to fail early
					if (!isFailEarlyOnGlobalRollbackOnly()) {
						unexpectedRollback = false;
					}
				}
			}
			catch (RuntimeException | Error ex) {
				// ......
			}

			// ......

			// 最终事务回滚会走到这里，抛出这个异常
			if (unexpectedRollback) {
				throw new UnexpectedRollbackException(
						"Transaction rolled back because it has been marked as rollback-only");
			}
		}
		finally {
			// ......
		}
	}
```

- 21行 进入回滚逻辑，会设置rollbackOnly=true

## 异常触发的地方

```java
	public final void commit(TransactionStatus status) throws TransactionException {
		if (status.isCompleted()) {
			throw new IllegalTransactionStateException(
					"Transaction is already completed - do not call commit or rollback more than once per transaction");
		}

		DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
		if (defStatus.isLocalRollbackOnly()) {
			if (defStatus.isDebug()) {
				logger.debug("Transactional code has requested rollback");
			}
			processRollback(defStatus, false);
			return;
		}

		if (!shouldCommitOnGlobalRollbackOnly() && defStatus.isGlobalRollbackOnly()) {
			if (defStatus.isDebug()) {
				logger.debug("Global transaction is marked as rollback-only but transactional code requested commit");
			}
			processRollback(defStatus, true);
			return;
		}

		processCommit(defStatus);
	}
```

- 16行 shouldCommitOnGlobalRollbackOnly 一直是false，所以第一个条件是true，所以只要第二个条件满足就会进入processRollback
这个地方比较骚气的地方是，commit里面会走真正的rollback逻辑，
```java
	public boolean isGlobalRollbackOnly() {
		return ((this.transaction instanceof SmartTransactionObject) &&
				((SmartTransactionObject) this.transaction).isRollbackOnly());
	}
```
结合上面的DefaultTransactionStatus对象可以清楚的看到属性变化

## 调用链路

一些细节不在本文展开

> 逻辑起始 
controller中调用 ` com.funnycode.dashboard.service.CService#execute `

> 核心代码
```java
	public Object proceed() throws Throwable {
		//	We start with an index of -1 and increment early.
		if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
			return invokeJoinpoint();
		}

		Object interceptorOrInterceptionAdvice =
				this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
		if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
			// Evaluate dynamic method matcher here: static part will already have
			// been evaluated and found to match.
			InterceptorAndDynamicMethodMatcher dm =
					(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
			Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());
			if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {
				return dm.interceptor.invoke(this);
			}
			else {
				// Dynamic matching failed.
				// Skip this interceptor and invoke the next in the chain.
				return proceed();
			}
		}
		else {
			// It's an interceptor, so we just invoke it: The pointcut will have
			// been evaluated statically before this object was constructed.
			return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
		}
	}
```

> 进入cglib
- org.sporg.springframework.aop.framework.CglibAopProxy.DynamicAdvisedInterceptor#intercept 这里会获取到事务切面
    - org.springframework.aop.framework.ReflectiveMethodInvocation#proceed 21行 继续调用自己方法 proceed()
        - org.springframework.transaction.interceptor.TransactionInterceptor#invoke 27行 再次进来会走Interceptor
            - org.springframework.transaction.interceptor.TransactionAspectSupport#invokeWithinTransaction
                - org.springframework.transaction.interceptor.TransactionAspectSupport.InvocationCallback#proceedWithInvocation
                - org.springframework.aop.framework.ReflectiveMethodInvocation#proceed
                    - org.springframework.aop.framework.CglibAopProxy.CglibMethodInvocation#invokeJoinpoint
                        - org.springframework.cglib.proxy.MethodProxy#invoke
                            - org.springframework.cglib.reflect.FastClass#invoke(int, java.lang.Object, java.lang.Object[])

> 执行
进入` com.funnycode.dashboard.service.CService#execute `真正执行代码


# 总结

Spring 事务默认级别是 REQUIRED，这样在外部有事务的情况下会共用一个事务，只要有一个地方出现异常，整个事务就会出错回滚了。如果你想要 A->B->C B 异常不影响 A，那么可以给 B 配置 PROPAGATION_REQUIRES_NEW。

## 概念归档

|     事务传播行为类型      |                             说明                             |
| :-----------------------: | :----------------------------------------------------------: |
|   PROPAGATION_REQUIRED    | 如果当前没有事务，就新建一个事务，如果已经存在一个事务中，加入到这个事务中。这是最常见的选择 |
|   PROPAGATION_SUPPORTS    |      支持当前事务，如果当前没有事务，就以非事务方式执行      |
|   PROPAGATION_MANDATORY   |         使用当前的事务，如果当前没有事务，就抛出异常         |
| PROPAGATION_REQUIRES_NEW  |          新建事务，如果当前存在事务，把当前事务挂起          |
| PROPAGATION_NOT_SUPPORTED |   以非事务方式执行操作，如果当前存在事务，就把当前事务挂起   |
|     PROPAGATION_NEVER     |        以非事务方式执行，如果当前存在事务，则抛出异常        |
|    PROPAGATION_NESTED     | 如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则执行与PROPAGATION_REQUIRED类似的操作 |

> PROPAGATION_NESTED 是所谓的子事务，PROPAGATION_REQUIRES_NEW 是挂起当前线程，是个完全独立当事务了

可以查看官方文件说明: https://docs.spring.io/spring/docs/3.0.x/spring-framework-reference/html/transaction.html#tx-propagation