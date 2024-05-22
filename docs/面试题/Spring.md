## IoC和AOP
1. IoC 控制反转：指将对象的管理的权利交给上层容器
   1. DI 依赖注入：IoC的具体实现，通过容器动态的将依赖注入到组件中
2. AOP 面相切面编程：通过定义切面实现不同业务模块间的解耦
## Spring中常见的注解

1. @service，用在服务层
2. @Component 将对应类声明为Spring的一个组件
3. @resposity 用在dao层
4. @condition 在自动装配时可以用来判断是否加载类
## AOP的底层实现

1. 基于动态代理实现
   1. JDK基于反射实现，只能代理接口实现的方法
   2. cglib基于继承实现
## spring的事务

1. spring的传播行为
   1. ![](https://raw.githubusercontent.com/danmuking/image/main/aad93f54f97f3ee6b28d123935ce2dae.png)
   2. REQUIRED类型
      1. 如果当前没有开启事务，那么就新建一个事务
      2. 如果当前已经存在事务，则加入当前事务
      3. 当前事务和新建事务属于同一个事务，任何一个失败，两个都会回滚
   3. REQUIREDS_NEW类型
      1. 如果当前没有开启事务，那么就新建一个事务
      2. 如果当期已经存在事务，则创建一个新事务
      3. 内部事务的失败可以被外部事务捕获，如果异常被捕获，内部事务失败，外部事务将不会回滚。如果外部事务失败，内部事务将正常执行
   4. NESTED类型
      1. 如果当前没有开启事务，那么就新建一个事务
      2. 如果当前存在事务，则创建一个嵌套事务
      3. 内部事务失败同样可以被外部事务捕获，但是如果外部事务失败，内部事务会被回滚
2. spring的隔离级别
   1. default
   2. read uncommit
   3. read commited
   4. repeated read
   5. serializable
3. spring事务失效的几种情况
   1. 抛出未检查的异常
      1. 原因：spring事务将会捕获指定的异常，如果抛出的异常不是指定异常事务就会失效
      2. 解决方法：将检查的异常设置为Expection
   2. 业务方法本身捕获了异常
      1. 原因：spring的事务是基于异常实现的，如果没有收到异常，事务自然失效
      2. 解决方法：需要注意Transactional的优先级是最低的，要避免异常被其他注解处理
   3. 在同一类中的方法调用
      1. 原因：spring事务默认情况下是基于JDK通过反射实现的，如果在同类中调用，将不会走代理
      2. 解决方法：设置通过cglib来实现事务
   4. 方法使用final或static关键字
      1. 原因：在cglib实现事务的情况下，事务是基于继承实现的，static和final关键字修饰的方法无法被继承
      2. 解决方法：想办法去掉关键字
   5. 方法不是public的
      1. 原因：在spring事务中会判断当前方法是否是public，如果没有将会去掉事务
      2. 解决方法：改成public
   6. 事务在多线程中执行
      1. 原因：在多线程中执行是，获得的mysql连接和当前线程不是同一个，事务自然失效。
      2. 解决方法：在一个线程中处理
## Bootstrap.yml和Application.yml
加载顺序：先加载Bootstrap.yml然后加载Application.yml
配置区别：

- bootstrap.yml 一般用来存放系统级的参数
- application.yml 可以用来定义应用级别的，如果存在和bootstrap相似的参数，会将其覆盖
## SpringBoot自动装配
原理：通过SPI扫描引入包下的`spring.factories`将其中的类加载到spring中。
相关问题：

1. 是否需要将`spring.factories`中所有的类都加载进来？
## @Autowared注解

1. 作用：自动装配
2. 使用：
   1. 在构造函数上
   2. 在set函数上
   3. 在变量上
   4. 在参数上
3. 规则：
   1. 先找到匹配的类，如果存在多个可用类，则按照名字匹配
   2. 如果连名字也无法区分，就需要配合`@Qulifier`一起使用
## @Resource和@Autowired的区别

1. 相同：
   1. 两者都可以用于自动装配
2. 区别
   1. Autowired是spring提供的，Resource是java提供的
   2. Autowired是根据type注入，如果要按照名称注入需要配合qualifier一起使用。Resource默认按照name注入，也可以按照type注入
