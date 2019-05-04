跑测试用例可以看出aliSkcService这个类已经被CGLIB代理了  
![](https://raw.githubusercontent.com/zhaoww/picbed/master/%E5%9B%BE%E5%BA%8A/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A120190504142421.png?token=AC6AR6RSYXYO72HWJYYZ4QS4ZU5LA)
继续追踪，进入了org.springframework.aop.framework.CglibAopProxy.DynamicAdvisedInterceptor#intercept
```
public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
	Object oldProxy = null;
	boolean setProxyContext = false;
	Class<?> targetClass = null;
	Object target = null;
	try {
		if (this.advised.exposeProxy) {
			// Make invocation available if necessary.
			oldProxy = AopContext.setCurrentProxy(proxy);
			setProxyContext = true;
		}
		// May be null. Get as late as possible to minimize the time we
		// "own" the target, in case it comes from a pool...
		target = getTarget();
		if (target != null) {
			targetClass = target.getClass();
		}
		List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
		Object retVal;
		// Check whether we only have one InvokerInterceptor: that is,
		// no real advice, but just reflective invocation of the target.
		if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
			// We can skip creating a MethodInvocation: just invoke the target directly.
			// Note that the final invoker must be an InvokerInterceptor, so we know
			// it does nothing but a reflective operation on the target, and no hot
			// swapping or fancy proxying.
			Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
			retVal = methodProxy.invoke(target, argsToUse);
		}
		else {
			// We need to create a method invocation...
			retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
		}
		retVal = processReturnType(proxy, target, method, retVal);
		return retVal;
	}
	finally {
		if (target != null) {
			releaseTarget(target);
		}
		if (setProxyContext) {
			// Restore old proxy.
			AopContext.setCurrentProxy(oldProxy);
		}
	}
}
```
重点在于proceed
```
// We need to create a method invocation...
retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
```
跟踪org.springframework.aop.framework.ReflectiveMethodInvocation#proceed
```
return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
```
跟踪org.springframework.transaction.interceptor.TransactionInterceptor#invoke
```
public Object invoke(final MethodInvocation invocation) throws Throwable {
		// Work out the target class: may be {@code null}.
		// The TransactionAttributeSource should be passed the target class
		// as well as the method, which may be from an interface.
		Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

		// Adapt to TransactionAspectSupport's invokeWithinTransaction...
		return invokeWithinTransaction(invocation.getMethod(), targetClass, new InvocationCallback() {
			@Override
			public Object proceedWithInvocation() throws Throwable {
				return invocation.proceed();
			}
		});
	}
```
在此方法里面有一个事务的核心方法 invokeWithinTransaction
```
/**
 * General delegate for around-advice-based subclasses, delegating to several other template
 * methods on this class. Able to handle {@link CallbackPreferringPlatformTransactionManager}
 * as well as regular {@link PlatformTransactionManager} implementations.
 * @param method the Method being invoked
 * @param targetClass the target class that we're invoking the method on
 * @param invocation the callback to use for proceeding with the target invocation
 * @return the return value of the method, if any
 * @throws Throwable propagated from the target invocation
 */
protected Object invokeWithinTransaction(Method method, Class<?> targetClass, final InvocationCallback invocation)
			throws Throwable {

	// If the transaction attribute is null, the method is non-transactional.
	final TransactionAttribute txAttr = getTransactionAttributeSource().getTransactionAttribute(method, targetClass);
	final PlatformTransactionManager tm = determineTransactionManager(txAttr);
	final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

	if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
		// Standard transaction demarcation with getTransaction and commit/rollback calls.
		TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
		Object retVal = null;
		try {
			// This is an around advice: Invoke the next interceptor in the chain.
			// This will normally result in a target object being invoked.
			// 执行实际的业务操作
			retVal = invocation.proceedWithInvocation();
		}
		catch (Throwable ex) {
			// target invocation exception
			// 业务回滚
			completeTransactionAfterThrowing(txInfo, ex);
			throw ex;
		}
		finally {
		    // 还原成之前的事务状态
			cleanupTransactionInfo(txInfo);
		}
		commitTransactionAfterReturning(txInfo);
		return retVal;
	}

	else {
		final ThrowableHolder throwableHolder = new ThrowableHolder();

		// It's a CallbackPreferringPlatformTransactionManager: pass a TransactionCallback in.
		try {
			Object result = ((CallbackPreferringPlatformTransactionManager) tm).execute(txAttr,
					new TransactionCallback<Object>() {
						@Override
						public Object doInTransaction(TransactionStatus status) {
							TransactionInfo txInfo = prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
							try {
								return invocation.proceedWithInvocation();
							}
							catch (Throwable ex) {
								if (txAttr.rollbackOn(ex)) {
									// A RuntimeException: will lead to a rollback.
									if (ex instanceof RuntimeException) {
										throw (RuntimeException) ex;
									}
									else {
										throw new ThrowableHolderException(ex);
									}
								}
								else {
									// A normal return value: will lead to a commit.
									throwableHolder.throwable = ex;
									return null;
								}
							}
							finally {
								cleanupTransactionInfo(txInfo);
							}
						}
					});

			// Check result state: It might indicate a Throwable to rethrow.
			if (throwableHolder.throwable != null) {
				throw throwableHolder.throwable;
			}
			return result;
		}
		catch (ThrowableHolderException ex) {
			throw ex.getCause();
		}
		catch (TransactionSystemException ex2) {
			if (throwableHolder.throwable != null) {
				logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
				ex2.initApplicationException(throwableHolder.throwable);
			}
			throw ex2;
		}
		catch (Throwable ex2) {
			if (throwableHolder.throwable != null) {
				logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
			}
			throw ex2;
		}
	}
}
```
跟踪createTransactionIfNecessary方法

```
/**
 * Create a transaction if necessary based on the given TransactionAttribute.
 * <p>Allows callers to perform custom TransactionAttribute lookups through
 * the TransactionAttributeSource.
 * @param txAttr the TransactionAttribute (may be {@code null})
 * @param joinpointIdentification the fully qualified method name
 * (used for monitoring and logging purposes)
 * @return a TransactionInfo object, whether or not a transaction was created.
 * The {@code hasTransaction()} method on TransactionInfo can be used to
 * tell if there was a transaction created.
 * @see #getTransactionAttributeSource()
 */
protected TransactionInfo createTransactionIfNecessary(
			PlatformTransactionManager tm, TransactionAttribute txAttr, final String joinpointIdentification) {

	// If no name specified, apply method identification as transaction name.
	if (txAttr != null && txAttr.getName() == null) {
		txAttr = new DelegatingTransactionAttribute(txAttr) {
			@Override
			public String getName() {
				return joinpointIdentification;
			}
		};
	}

	TransactionStatus status = null;
	if (txAttr != null) {
		if (tm != null) {
			status = tm.getTransaction(txAttr);
		}
		else {
			if (logger.isDebugEnabled()) {
				logger.debug("Skipping transactional joinpoint [" + joinpointIdentification +
						"] because no transaction manager has been configured");
			}
		}
	}
	return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
}


public final TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException {
	Object transaction = doGetTransaction();

	// Cache debug flag to avoid repeated checks.
	boolean debugEnabled = logger.isDebugEnabled();

	if (definition == null) {
		// Use defaults if no transaction definition given.
		definition = new DefaultTransactionDefinition();
	}
	// 存在事务
	if (isExistingTransaction(transaction)) {
		// Existing transaction found -> check propagation behavior to find out how to behave.
		return handleExistingTransaction(definition, transaction, debugEnabled);
	}

	// Check definition settings for new transaction.
	if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
		throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
	}

	// No existing transaction found -> check propagation behavior to find out how to proceed.
	if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
		throw new IllegalTransactionStateException(
				"No existing transaction found for transaction marked with propagation 'mandatory'");
	}
	else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
			definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
			definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
		SuspendedResourcesHolder suspendedResources = suspend(null);
		if (debugEnabled) {
			logger.debug("Creating new transaction with name [" + definition.getName() + "]: " + definition);
		}
		try {
			boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
			DefaultTransactionStatus status = newTransactionStatus(
					definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
			doBegin(transaction, definition);
			prepareSynchronization(status, definition);
			return status;
		}
		catch (RuntimeException ex) {
			resume(null, suspendedResources);
			throw ex;
		}
		catch (Error err) {
			resume(null, suspendedResources);
			throw err;
		}
	}
	else {
		// Create "empty" transaction: no actual transaction, but potentially synchronization.
		if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
			logger.warn("Custom isolation level specified but no actual transaction initiated; " +
					"isolation level will effectively be ignored: " + definition);
		}
		boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
		return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
	}
}
```
跟踪org.springframework.transaction.support.AbstractPlatformTransactionManager#handleExistingTransaction 在此方法内根据不同的事务传播行为进行处理
```
/**
 * Create a TransactionStatus for an existing transaction.
 */
private TransactionStatus handleExistingTransaction(
			TransactionDefinition definition, Object transaction, boolean debugEnabled)
			throws TransactionException {

	if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NEVER) {
		throw new IllegalTransactionStateException(
				"Existing transaction found for transaction marked with propagation 'never'");
	}

	if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NOT_SUPPORTED) {
		if (debugEnabled) {
			logger.debug("Suspending current transaction");
		}
		// 挂起当前事务
		Object suspendedResources = suspend(transaction);
		boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
		return prepareTransactionStatus(
				definition, null, false, newSynchronization, debugEnabled, suspendedResources);
	}

	if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
		if (debugEnabled) {
			logger.debug("Suspending current transaction, creating new transaction with name [" +
					definition.getName() + "]");
		}
		// 挂起当前事务
		SuspendedResourcesHolder suspendedResources = suspend(transaction);
		try {
			boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
			// 创建新的事务
			DefaultTransactionStatus status = newTransactionStatus(
					definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
			// 开启事务
			doBegin(transaction, definition);
			prepareSynchronization(status, definition);
			return status;
		}
		catch (RuntimeException beginEx) {
			resumeAfterBeginException(transaction, suspendedResources, beginEx);
			throw beginEx;
		}
		catch (Error beginErr) {
			resumeAfterBeginException(transaction, suspendedResources, beginErr);
			throw beginErr;
		}
	}

	if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
		if (!isNestedTransactionAllowed()) {
			throw new NestedTransactionNotSupportedException(
					"Transaction manager does not allow nested transactions by default - " +
					"specify 'nestedTransactionAllowed' property with value 'true'");
		}
		if (debugEnabled) {
			logger.debug("Creating nested transaction with name [" + definition.getName() + "]");
		}
		if (useSavepointForNestedTransaction()) {
			// Create savepoint within existing Spring-managed transaction,
			// through the SavepointManager API implemented by TransactionStatus.
			// Usually uses JDBC 3.0 savepoints. Never activates Spring synchronization.
			DefaultTransactionStatus status =
					prepareTransactionStatus(definition, transaction, false, false, debugEnabled, null);
			status.createAndHoldSavepoint();
			return status;
		}
		else {
			// Nested transaction through nested begin and commit/rollback calls.
			// Usually only for JTA: Spring synchronization might get activated here
			// in case of a pre-existing JTA transaction.
			boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
			DefaultTransactionStatus status = newTransactionStatus(
					definition, transaction, true, newSynchronization, debugEnabled, null);
			doBegin(transaction, definition);
			prepareSynchronization(status, definition);
			return status;
		}
	}

	// Assumably PROPAGATION_SUPPORTS or PROPAGATION_REQUIRED.
	if (debugEnabled) {
		logger.debug("Participating in existing transaction");
	}
	if (isValidateExistingTransaction()) {
		if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT) {
			Integer currentIsolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
			if (currentIsolationLevel == null || currentIsolationLevel != definition.getIsolationLevel()) {
				Constants isoConstants = DefaultTransactionDefinition.constants;
				throw new IllegalTransactionStateException("Participating transaction with definition [" +
						definition + "] specifies isolation level which is incompatible with existing transaction: " +
						(currentIsolationLevel != null ?
								isoConstants.toCode(currentIsolationLevel, DefaultTransactionDefinition.PREFIX_ISOLATION) :
								"(unknown)"));
			}
		}
		if (!definition.isReadOnly()) {
			if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
				throw new IllegalTransactionStateException("Participating transaction with definition [" +
						definition + "] is not marked as read-only but existing transaction is");
			}
		}
	}
	boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
	return prepareTransactionStatus(definition, transaction, false, newSynchronization, debugEnabled, null);
}
```
出现异常则调用org.springframework.transaction.interceptor.TransactionAspectSupport#completeTransactionAfterThrowing
```
/**
 * Handle a throwable, completing the transaction.
 * We may commit or roll back, depending on the configuration.
 * @param txInfo information about the current transaction
 * @param ex throwable encountered
 */
protected void completeTransactionAfterThrowing(TransactionInfo txInfo, Throwable ex) {
	if (txInfo != null && txInfo.hasTransaction()) {
		if (logger.isTraceEnabled()) {
			logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() +
					"] after exception: " + ex);
		}
		if (txInfo.transactionAttribute.rollbackOn(ex)) {
			try {
				txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
			}
			catch (TransactionSystemException ex2) {
				logger.error("Application exception overridden by rollback exception", ex);
				ex2.initApplicationException(ex);
				throw ex2;
			}
			catch (RuntimeException ex2) {
				logger.error("Application exception overridden by rollback exception", ex);
				throw ex2;
			}
			catch (Error err) {
				logger.error("Application exception overridden by rollback error", ex);
				throw err;
			}
		}
		else {
			// We don't roll back on this exception.
			// Will still roll back if TransactionStatus.isRollbackOnly() is true.
			try {
				txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
			}
			catch (TransactionSystemException ex2) {
				logger.error("Application exception overridden by commit exception", ex);
				ex2.initApplicationException(ex);
				throw ex2;
			}
			catch (RuntimeException ex2) {
				logger.error("Application exception overridden by commit exception", ex);
				throw ex2;
			}
			catch (Error err) {
				logger.error("Application exception overridden by commit error", ex);
				throw err;
			}
		}
	}
}

```
进行回滚操作主体流程
```
/**
 * Reset the TransactionInfo ThreadLocal.
 * <p>Call this in all cases: exception or normal return!
 * @param txInfo information about the current transaction (may be {@code null})
 */
private void processRollback(DefaultTransactionStatus status) {
	try {
		try {
			triggerBeforeCompletion(status);
			if (status.hasSavepoint()) {
				if (status.isDebug()) {
					logger.debug("Rolling back transaction to savepoint");
				}
				status.rollbackToHeldSavepoint();
			}
			else if (status.isNewTransaction()) {
				if (status.isDebug()) {
					logger.debug("Initiating transaction rollback");
				}
				doRollback(status);
			}
			else if (status.hasTransaction()) {
				if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
					if (status.isDebug()) {
						logger.debug("Participating transaction failed - marking existing transaction as rollback-only");
					}
					doSetRollbackOnly(status);
				}
				else {
					if (status.isDebug()) {
						logger.debug("Participating transaction failed - letting transaction originator decide on rollback");
					}
				}
			}
			else {
				logger.debug("Should roll back transaction but cannot - no transaction available");
			}
		}
		catch (RuntimeException ex) {
			triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
			throw ex;
		}
		catch (Error err) {
			triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
			throw err;
		}
		triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
	}
	finally {
		cleanupAfterCompletion(status);
	}
}
```
最终进行还原事务org.springframework.transaction.interceptor.TransactionAspectSupport#cleanupTransactionInfo
```
private void restoreThreadLocalStatus() {
	// Use stack to restore old transaction TransactionInfo.
	// Will be null if none was set.
	transactionInfoHolder.set(this.oldTransactionInfo);
}
```
如果没有异常则调用org.springframework.transaction.interceptor.TransactionAspectSupport#commitTransactionAfterReturning 提交事务
```
/**
 * Execute after successful completion of call, but not after an exception was handled.
 * Do nothing if we didn't create a transaction.
 * @param txInfo information about the current transaction
 */
protected void commitTransactionAfterReturning(TransactionInfo txInfo) {
	if (txInfo != null && txInfo.hasTransaction()) {
		if (logger.isTraceEnabled()) {
			logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() + "]");
		}
		// 提交事务
		txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
	}
}
```