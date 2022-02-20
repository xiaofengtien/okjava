## spring事务的传播机制列表

spring事务的传播机制共7中,可以分为3组+1个特殊来分析或者记忆

1).REQUIRE组

   1.REQUIRED:当前存在事务则使用当前的事务, 当前不存在事务则创建一个新的事务

   2.REQUIRES_NEW:创建新事务, 如果已经存在事务, 则把已存在的事务挂起

2).SUPPORT组

   1.SUPPORTS:支持事务. 如果当前存在事务则加入该事务, 如果不存在事务则以无事务状态执行

   2.NOT_SUPPORTED:不支持事务. 在无事务状态下执行,如果已经存在事务则挂起已存在的事务

3).Exception组

   1.MANDATORY:必须在事务中执行, 如果当前不存在事务, 则抛出异常

​    2.NEVER: 不可在事务中执行, 如果当前存在事务, 则抛出异常

4).NESTED:嵌套事务. 如果当前存在事务, 则嵌套执行, 如果当前不存在事务, 则开启新事务

3.关于NESTED的理解

对于以上7中传播机制, 大部分大家根据解释说明就可以比较好的理解, 这里就不在详细说明了. 唯一比较晦涩的可能就是关于NESTED的理解. 现在说一下对于这部分的理解

首先贴出对于NESTED机制的注释说明:

> /**
>
> ​      \* Execute within a nested transaction if a current transaction exists,
>
> ​      \* behave like {@link #PROPAGATION_REQUIRED} else. There is no analogous
>
> ​       \* feature in EJB.
>
> ​       \* <p><b>NOTE:</b> Actual creation of a nested transaction will only work on
>
> ​      \* specific transaction managers. Out of the box, this only applies to the JDBC
>
> ​      \* {@link org.springframework.jdbc.datasource.DataSourceTransactionManager}
>
> ​       \* when working on a JDBC 3.0 driver. Some JTA providers might support
>
> ​      \* nested transactions as well.
>
> ​      \* @see org.springframework.jdbc.datasource.DataSourceTransactionManager
>
> ​      */

由注释可以看出, 如果当前不存在事务的话, NESTED机制的操作其实是和REQUIRED机制是一样的.但是如果当前存在事务的话,NESTED机制会创建一个事务, 新创建的事务嵌套在当前事务中执行. 说到这里, 大家可能会想到REQUIRES_NEW机制, 在当前存在事务的情况下这两个机制都会创建事务, 不同的是REQUIRES_NEW会把已存在的事务挂起, 而NESTED机制会把新创建的事务嵌套在原事务中执行.那么这两者的区别是什么呢?

首相需要明确的是, REQUIRES_NEW机制是真正意义上的独立的新事务, 新事务与原事务没有一毛钱关系, 他拥有自己的隔离区,锁等等事务的属性. 而NESTED则是真正意义上的原事务的子事务. 他所创建的事务可以理解为在原事务中设置了一个savepoint, 新事务相当于内部事务, 原事务相当于外部事务, 内部事务的提交依赖于外部事务的提交. 内部事务的回滚其实是把操作回到savepoint点的状态

使用NESTED机制需要设置transactionManager的nestedTransactionAllowed为true