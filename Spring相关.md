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