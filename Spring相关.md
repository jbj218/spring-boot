# Spring相关

## 1、IOC

> 什么是IOC?

IOC即控制反转，将创建对象的权力交给外部容器。

> IOC的好处？

1、降低对象之间的耦合程度，结构更加清晰扁平。

2、资源变得更加容易管理，比如很容易实现单例。

## 2、Bean

> 什么是Bean？

.......

> 注入Bean的三种方式？

1、按字段注入

2、构造器注入

3、setter注入

> `@Autowired`和`@Resourse`的区别？

`@Autowired`**先按类型（byType）匹配**，**若存在多个同类型 Bean，则再尝试按名称（byName）筛选**。默认 Bean 名称为类名首字母小写一致

当一个接口存在多个实现类时：

- 若属性名与某个 Bean 的名称，则注入该 Bean；
- 若属性名与所有 Bean 名称都不匹配，会抛出 `NoUniqueBeanDefinitionException`，此时需要通过 `@Qualifier` 显式指定要注入的 Bean 名称。



`@Resourse`**按名称（byName）筛选**，找不到名称**按类型匹配（byType）**

> Bean的作用域？

6个

全局端（2个）：

singleton:单例bean。==IOC容器级别，如果有多个ApplicationContext,每个容器都有自己的单例bean==

prototype:多实例bean

web端（4个）：

request:一个请求对应一个bean

session:一个会话级别一个bean

application:一个ServletContext一个bean。==可以说真正意义上的单例，整个Web应用就一个ServletContext，跨容器全局的单例。==

websocket:一个socket一个bean

> Bean是线程安全的吗？

Bean是否线程安全取决于作用域和状态

**无状态bean:**

首先无状态的bean是不会有线程安全问题的，多线程访问无状态bean根源上不会修改bean的内容。（因为都没有可修改的内容）

**有状态bean:**

~~~java
@Service
public class OrderService {
    private int orderCount = 0;  // 危险！可变实例变量
    
    public void createOrder() {
        orderCount++;  // 多线程下会丢数据
    }
}

~~~

针对大多数情况：即单例bean

1、从作用域上处理：改成`prototype`作用域，每次getBean都新建一个实例，天然线程隔离。

2、ThreadLocal:每个线程维护自己的副本

3、字段上：用线程安全的数据结构，如果ConcurrentHashMap代替HashMap,用原子类或者上锁。

<h3>Tip：</h3>

我用final修饰不就保证不可变了吗？

不行，首先从功能上来说，用final修饰就代表你希望这个字段是不可变得。其次final只能保证基本类型不可变以及引用不可变。

有意思的点：
使用集合的角度来说
如果希望一个集合不可变，用final就是一个集合是不可行的，因为只能保证引用不变，集合该怎么操作怎么操作。解决办法用**不可变集合**，如 `List.of(1,2,3)`。就像无状态bean一样，天然线程安全。

如果希望一个集合可变且线程安全，用**线程安全集合**

<h3>总结</h3>

希望一个bean是线程安全的方法

![image-20261015221146443](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/image-20261015221146443.png)

> bean的生命周期？

<h3>5步骤</h3>

1. 实例化
2. 依赖注入，字段注入
3. 初始化
4. 使用bean
5. 销毁bean

<h3>7步骤</h3>

1. 实例化
2. 依赖注入、字段注入
3. ==初始化前，BeanPostProcessor的before方法：对 Bean 做“使用前的前置加工“==
4. 初始化
5. ==初始化后，BeanPosetProcessor的After方法（AOP...就在这儿做的）：对 Bean 做“成品后的加工/包装”==
6. 使用bean
7. 销毁bean

<h3>8步骤（了解到此）</h3>

1. 实例化
2. 依赖注入、字段注入
3. ==实现Aware接口，让 Bean 感知/知道自己所处的环境==
4. 初始化前，BeanPostProcessor的before方法：对 Bean 做“使用前的前置加工“
5. 初始化
6. 初始化后，BeanPosetProcessor的After方法（AOP...就在这儿做的）：对 Bean 做“成品后的加工/包装”
7. 使用bean
8. 销毁bean

![image-20261015230220650](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/image-20261015230220650.png)

<h3>完全体</h3>

颜色对应细节扩展

![image-20261015231535192](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/image-20261015231535192.png)

展开：

![image-20261015231321647](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/image-20261015231321647.png)

<h3>tip:</h3>

`@PostConstrcut`事实上是被BeanPostProcessor的**before方法包裹住执行**的,而**after不包裹**，只会定义自己的逻辑

<h4>为什么 `@PostConstruct` 特殊？</h4>

因为它是“**外部标准**”，Spring 得“**借壳**”执行它：

```
Spring 自己的初始化机制：
    afterPropertiesSet()  → 直接安排进生命周期
    init-method           → 直接安排进生命周期

外部标准（JSR-250）：
    @PostConstruct        → 借 before 这个钩子执行
```

而 after 不承担这种“借壳”任务，所以它**纯粹是个“时机点”**——用来在“初始化全部完成后”做加工（生成代理等）。

<h2>图对应快速查阅</h2>

<h3>一、初始化阶段（bean 诞生）</h3>

| #    | 图中名称                   | 实际调用的方法                                               | 出处 / 驱动者                                               | 作用                                                         |
| :--- | :------------------------- | :----------------------------------------------------------- | :---------------------------------------------------------- | :----------------------------------------------------------- |
| 1    | 构造函数                   | 反射 `Constructor.newInstance()`                             | 容器反射调用                                                | 创建对象实例。此时依赖还没注入，字段是 null                  |
| 2    | 依赖注入                   | 属性填充 `populateBean()`                                    | `@Autowired` / `@Value` / `@Resource`                       | 给字段和 setter 塞依赖，解决"我用谁"                         |
| 3    | aware 注入                 | `setBeanName()` / `setBeanClassLoader()` / `setBeanFactory()` | `BeanNameAware`、`BeanClassLoaderAware`、`BeanFactoryAware` | 把容器自身资源交给 bean，解决"我在哪、我能拿到啥"。**三者是按需实现的并列关系，不是依次执行** |
| 4    | BeanPostProcessor 前置方法 | `postProcessBeforeInitialization(bean, beanName)`            | `BeanPostProcessor` 接口                                    | **容器级横切**，所有 bean 都过一遍。`@PostConstruct` 就是在这被 `CommonAnnotationBeanPostProcessor` 调起来的 |
| 5    | @PostConstruct             | 标了该注解的方法                                             | JSR-250 注解                                                | 自定义初始化：校验、预热、建连接                             |
| 6    | InitializingBean           | `afterPropertiesSet()`                                       | 实现 `InitializingBean` 接口                                | Spring 原生钩子，语义就是"属性填完了"                        |
| 7    | initMethod                 | `@Bean(initMethod="xxx")` 指定的方法                         | 配置声明                                                    | 不改源码就能指定初始化方法                                   |
| 8    | BeanPostProcessor 后置方法 | `postProcessAfterInitialization(bean, beanName)`             | `BeanPostProcessor` 接口                                    | 统一后置处理。**AOP 代理就在这里生成**（`AbstractAutoProxyCreator`） |
| 9    | 使用                       | —                                                            | —                                                           | bean 就绪。注意你拿到的大概率是第 8 步产出的代理对象，不是原始实例 |

<h3>二、销毁阶段（容器关闭时）</h3>

| #    | 图中名称       | 实际调用的方法                          | 出处                       | 作用                                                         |
| :--- | :------------- | :-------------------------------------- | :------------------------- | :----------------------------------------------------------- |
| 10   | @PreDestroy    | 标了该注解的方法                        | JSR-250 注解               | 释放资源、清理线程                                           |
| 11   | DisposableBean | `destroy()`                             | 实现 `DisposableBean` 接口 | Spring 原生销毁钩子                                          |
| 12   | destroyMethod  | `@Bean(destroyMethod="xxx")` 指定的方法 | 配置声明                   | 第三方对象的销毁。`@Bean` 默认会自动推断 `close()` / `shutdown()` |

**销毁回调只对单例 bean 生效。** prototype 的 bean 出了容器就不管了，销毁由你自己负责。



## 3、AOP

> 什么是AOP？

AOP即面向切面编程，用于封装于业务无关，为业务所共用的逻辑（日志、事务、权限控制....）

AOP基于动态代理：

1、JDK代理，需要实现某个接口

2、CGLIB代理，无需实现接口，基于继承。（spring2.x后默认都是这个）

> Spring AOP 和AspectJ AOP的区别

Spring AOP 支持运行时织入，构造函数，静态方法，字段访问都做不到

AspectJ AOP 支持运行、类加载、编译时（字节码层面的操作），功能更加的强大

> Spring AOP 常见通知类型

5种
前置通知before

后置通知after

返回通知

异常返回通知

环绕（around）最强大，其他通知只能“在方法前后做点事”，@Around 能“完全接管方法的执行”。

<h3>Tip:</h3>

当多个通知同时存在时，执行顺序是：

~~~text
@Around 前半部分
    ↓
@Before
    ↓
目标方法执行
    ↓
@AfterReturning / @AfterThrowing
    ↓
@After
    ↓
@Around 后半部分
~~~

![image-20261016101004550](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/image-20261016101004550.png)

## 4、Spring MVC

> Spring MVC工作流程

1、DispatcherServlet 根据URL查询能够处理的Handler（即`Controller` 控制器）

2、Hanlder处理请求

3、返回视图或JSON(前后端分离)

更细节的流程：

![image-20261016101902704](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/image-20261016101902704.png)

## 5、循环依赖

> 什么是循环依赖?

循环依赖指两个及以上的bean互相持有对方引用导致无法获取bean

Tip：循环依赖是一种设计缺陷，应该在最开始就避免。spring 2.6.x后默认禁止循环依赖，当然像开启也可以手动开启

>如何解决循环依赖？

两个方法：

1、三级缓存（解决部分单例bean通过字段注入或Setter注入时的循环依赖），出现循环时，先从三级缓存中获取A的前期暴露对象，此时虽然bean不完整但已近有地址了，可以根据A去获取B了

2、`@Lazy`懒加载，即用时加载。非懒加载时A,B在容器启动时就要加载了。开启懒加载后，比如A调用B时，在B上标注`@Lazy`，此时会先加载A，知道需要传入B时在加载B

<h3>Tip</h3>

三级缓存会有额外的内存开销。

懒加载虽然会让整个容器启动更快，但运行过程中需要第一次使用bean时又会变慢，复杂依赖关系下也可能产生更隐蔽的初始化问题。

## 6、事务

> spring中事务的传播行为？

更具体查看[Spring 有哪几种事务传播行为? - 面试鸭 | 2026最新面试题+详细答案解析](https://www.mianshiya.com/question/1780933295417094146#heading-9)

红框为常用的三个：

- REQUIRED(required),占绝大多数场景。比如订单创建调库存扣减，两个操作必须在同一个事务里，要么一起成功要么一起回滚。
- REQUIRES_NEW(requires_new)，==开启新事务，原事务挂起==。用在"不管主流程成败，我这步必须独立提交"的场景。典型的就是审计日志、操作记录。用户转账失败了没关系，但"转账失败"这条日志必须写进去，不能因为主事务回滚把日志也吞了。还有发送通知、积分变更这类"尽力而为"的操作也适合用它。
- NESTED(nested)==开启嵌套子事务，不影响父事务执行==,适合"部分失败可以接受"的批量处理。比如导入 1000 条数据，每条数据开一个嵌套子事务，某条失败就回滚那条，其他的继续。外层事务负责整体控制，发现失败太多可以整体回滚。

![image-20261016105827796](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20261016105827796.png)

> 事务隔离级别

Spring 定义了 5 种事务隔离级别，本质上是对数据库隔离级别的封装：

1. **DEFAULT**（默认）：使用底层数据库的默认隔离级别。如Mysql默认隔离级别是可重复读。
2. **READ_UNCOMMITTED**（读未提交）：最低的隔离级别，允许事务读取尚未提交的数据，可能会导致脏读、不可重复读和幻读。
3. **READ_COMMITTED**（读已提交）：仅允许读取已经提交的数据，避免了脏读，但可能会出现不可重复读和幻读问题。
4. **REPEATABLE_READ**（可重复读）：确保在同一个事务内的多次读取结果一致，避免脏读和不可重复读，但可能会有幻读问题。
5. **SERIALIZABLE**（可串行化）：最高的隔离级别，通过强制事务按顺序执行，完全避免脏读、不可重复读和幻读，代价是性能显著下降。

## 7、权限控制

### SpringSecurity

> 有哪些控制访问请求权限的方法？

- `permitAll()`：无条件允许任何形式访问，不管你登录还是没有登录。
- `anonymous()`：允许匿名访问，也就是没有登录才可以访问。
- `denyAll()`：无条件决绝任何形式的访问。
- `authenticated()`：只允许已认证的用户访问。
- `fullyAuthenticated()`：只允许完整认证的用户访问，不接受匿名认证或 remember-me 认证。
- `hasRole(String)` : 只允许指定的角色访问。
- `hasAnyRole(String)` : 指定一个或者多个角色，满足其一的用户即可访问。
- `hasAuthority(String)`：只允许具有指定权限的用户访问
- `hasAnyAuthority(String)`：指定一个或者多个权限，满足其一的用户即可访问。
- `hasIpAddress(String)` : 只允许指定 ip 的用户访问

> 如何对密码进行加密？

Spring Security 提供了多种密码编码算法的实现，开箱即用。这些实现类的接口是 `PasswordEncoder`；如果需要自定义密码编码方案，也需要实现 `PasswordEncoder` 接口。

`PasswordEncoder` 接口有 `encode()` 和 `matches()` 两个必须实现的抽象方法，以及一个可以按需覆盖的默认方法 `upgradeEncoding()`。

`encode()`:定义加密规则；`matches()`验证密码是否和输入的一样

~~~java
public interface PasswordEncoder {
    // 对原始密码进行单向编码
    String encode(CharSequence var1);
    // 比对原始密码和数据库中保存的密码
    boolean matches(CharSequence var1, String var2);
    // 判断已编码密码是否需要升级编码，默认返回 false
    default boolean upgradeEncoding(String encodedPassword) {
        return false;
    }
}
~~~

## 8、日志

