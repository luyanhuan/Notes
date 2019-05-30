# Spring-事务
面试的时候基本事务问题肯定会问到，比如事务的传播性，事务的隔离型，事务失效问题等。大部分我都是停留在理论阶段，现在就来整理实践一下吧！
## 事务的传播性
找到@Transaction的注解定义：
```
Propagation propagation() default Propagation.REQUIRED;
```
```
REQUIRED(0),
SUPPORTS(1),
MANDATORY(2),
REQUIRES_NEW(3),
NOT_SUPPORTED(4),
NEVER(5),
NESTED(6);
```
可以看到，默认是使用REQUIRED的，这也是我们最常用的传播性。
## 事务的隔离性
```
Isolation isolation() default Isolation.DEFAULT;
```
```
DEFAULT(-1),
READ_UNCOMMITTED(1),
READ_COMMITTED(2),
REPEATABLE_READ(4),
SERIALIZABLE(8);
```
默认是default,也就是用数据库默认的隔离性。
## 事务失效
在用spring Test测试的过程中，发现即使正确执行，最后也会看到一句：
```
o.s.t.c.transaction.TransactionContext   : Rolled back transaction for test:XXXXXX
```
原来是在使用spring Test进行单元测试的时候，默认会对事务进行回滚，要想测试不回滚数据，就得加上@Rollback(value = false),
或者老老实实自己把service写好，在service里加上事务。
* 数据库不支持事务的情况，spring的事务会失效。   
eg:使用mysql数据库，修改user表的引擎为MyISAM,该引擎不支持事务，执行下列代码，发现事务并没有回滚，事务失效了。
```
@Test
@Transactional
public void contextLoads() {
    userService.save(User.builder().name("事务失效").build());
}
```
![log](/images/spring/1.png)   
* 使用try catch捕获异常，事务不会回滚
```
@Test
public void testTransaction() {
    userService.add(User.builder().sex(1).enName("yhlu").name("事务失效111").build());
}

@Override
@Transactional
public void add(User user) {
    this.save(user);
    try {
        int a = 1/0;
    } catch (Exception e) {
        //e.printStackTrace();
    }
}
```
![log](/images/spring/2.png)   
* Spring默认对某些异常不会回滚
这里涉及到Exception的分类总结，搬运一个网上大神的总结,Throwable为基类，Error和Exception继承Throwable。
Error和RuntimeException及其子类成为未检查异常（unchecked即编译器不会检查出来），其它异常成为已检查异常（checked）：
![exception](/images/spring/3.jpg)   
Spring默认只对unchecked异常回滚，且必须是抛出异常。
eg:测试抛出IOException，事务并不会回滚。
```
@Test
public void testTransaction() {
    try {
        userService.add(User.builder().sex(1).enName("yhlu").name("事务失效111").build());
    } catch (Exception e) {
//            e.printStackTrace();
    }
}

@Transactional
public void add(User user) throws IOException {
    this.save(user);
    throw new IOException();
}
``` 
![exception](/images/spring/4.png)
