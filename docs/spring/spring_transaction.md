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
eg:使用mysql数据库，修改user表的引擎为MyISAM,该引擎不支持事务，执行下列代码，发现事务并没有回滚，事务失效了,看到即使rollback数据还是插入到表里了。
```
@Test
@Transactional
public void contextLoads() {
    userService.save(User.builder().name("事务失效").build());
}
```
![log](/images/spring/transaction/1.png)   
* 使用try catch捕获异常，事务不会回滚。
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
![log](/images/spring/transaction/2.png)   
解决：异常不要捕获尽量抛出来，如果有捕获的需求在catch里处理完后再抛出来。 
* Spring默认对某些异常不会回滚
这里涉及到Exception的分类总结，搬运一个网上的总结,Throwable为基类，Error和Exception继承Throwable。
Error和RuntimeException及其子类称为未检查异常（unchecked即编译器不会检查出来），其它异常称为已检查异常（checked）：
![exception](/images/spring/transaction/3.jpg)   
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
![log](/images/spring/transaction/4.png)    
解决：为@Transactional注解添加rollbackFor = {Exception.class......}
* 非事务方法调用自身类事务方法，事务失效。
eg:按照如下逻辑调用，并没有开启事务。
```
@Test
public void testTransaction() {
    userService.add(User.builder().enName("yhlu").sex(0).name("小陆11").build());
}

@Override
public void add(User user) {
    addUser(user);
}

@Transactional
public void addUser(User user) {
    this.save(user);
    throw new IllegalArgumentException();
}
```
![log](/images/spring/transaction/5.jpg)   
上述操作为何最后没有在事务内？先思考一下，这里的@Transactional，事务的声明式注解是怎么实现的？Spring Aop呀！Spring Aop又是通过动态代理去实现的。
这里搬运一个网络的方法调用图，就很清晰了：    
![method call procedure](/images/spring/transaction/6.gif)    
从上面的分析可以看出，methodB没有被AopProxy通知到，导致最终结果是：被Spring的AOP增强的类，在同一个类的内部方法调用时，其被调用方法上的增强通知将不起作用。
因此可知只要利用Spring Aop机制实现的消息通知都会受到这样的限制。   
那么进一步认证上述说法，自己来做一个小测试吧：写一个切面，切特定的方法，进行同一个类内部方法调用，通知是否起作用？    
```
@Aspect
@Component
@Slf4j
public class TestAspect {
    @Pointcut("execution(public * com.example.demo..*.pointCut(..))")
    private void pointCut(){};
    
    @Around("pointCut()")
    public void around(ProceedingJoinPoint proceedingJoinPoint) {
        log.info("环绕通知的目标方法名："+proceedingJoinPoint.getSignature().getName());
        try {
            log.info("------执行方法");
            proceedingJoinPoint.proceed();
        } catch (Throwable throwable) {
            throwable.printStackTrace();
        }
    }
}
```
```
@Test
public void testAop() {
    userService.testAop();
}
```
```
@Override
public void pointCut() {
    System.out.println("--------userServiceImpl.pointCut()");
}

@Override
public void testAop() {
    System.out.println("--------userServiceImpl.testAop()");
    pointCut();
}
```
![log](/images/spring/transaction/7.jpg)    
可以看到通知确实没起作用！   
解决：   
1.为原非事务方法增加@Transactional注解   
2.暴露代理对象(@EnableAspectJAutoProxy(exposeProxy=true));原非事务方法中将this调用改成动态代理调用(AopContext.currentProxy()).method()，这样就能触发aop的通知啦~
```
@Override
public void testAop() {
    System.out.println("--------userServiceImpl.testAop()");
    ((UserServiceImpl) AopContext.currentProxy()).pointCut();
//        pointCut();
}
```
![log](/images/spring/transaction/8.jpg)  