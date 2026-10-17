# Mybatis

## Mybatis的执行流程

初始化阶段：

1、解析全局配置文件**mybatis-config.xml**,加载每个xml文件（mapper.xml和mapper接口），获取到数据源、注册信息等，将sql语句解析成对应的MappedStatement，将这些信息存入Configuration对象中

2、根据Configuration构建出sqlSessionFactory

执行阶段：

1、通过sqlSessionFactory获取一个sqlSession，再调用getMapper()获取mapper接口对应的动态代理对象。
2、执行mapper动态代理对象，通过MappedStatement获取sql语句。之后就走JDBC那一套了，预编译sql传入参数，获取结果集。将结果集映射成对应的对象输出即可。

> Mybatis执行一条查询的流程？

上面就是

> ${}和#{}有啥区别？

`${}`走字符串拼接，容易被sql注入，底层使用`Statement`

`#{}` 也就是`？`走预编译，更加安全，底层使用`PrepareStatement`。但是涉及表结构的sql（需要动态拼接表名、列名、排序字段这些没法用占位符），`#{}`使用不了,只能使用${},此时为了保证安全会用白名单校验以保证安全。

## 动态sql

> 什么是动态sql

动态sql，就是mybatis再xml中提供的一类标签，根据传入的条件不同动态的拼接sql语句。不用再java代码中写一堆if-else判断来拼接sql了。

> 原理是什么？

mybatis解析xml时，遇见动态标签时会会构建出一个SqlNode树，执行时遍历这颗树，条件符合即拼接

> 常见的动态sql标签？

1）`if` 标签，最常用的条件判断，test 条件成立就拼进去，不成立就跳过：

```xml
<select id="findUsers" resultType="User">
    SELECT * FROM users
    <where>
        <if test="name != null">
            AND name = #{name}
        </if>
        <if test="age != null">
            AND age = #{age}
        </if>
    </where>
</select>
```

2）`where` 标签，自动帮你加 WHERE 关键字，还能把多余的 AND、OR 干掉，不用自己写 `WHERE 1=1` 了。

3）`foreach` 标签，专门处理集合遍历，批量插入、IN 查询全靠它：

```xml
<select id="findUsersInIds" resultType="User">
    SELECT * FROM users
    WHERE id IN
    <foreach item="id" collection="idList" open="(" separator="," close=")">
        #{id}
    </foreach>
</select>
```

4）`choose` + `when` + `otherwise`，相当于 Java 的 switch-case，==多个条件只命中一个==：

```xml
<select id="findUsers" resultType="User">
    SELECT * FROM users
    <where>
        <choose>
            <when test="status != null">
                status = #{status}
            </when>
            <otherwise>
                status = 'ACTIVE'
            </otherwise>
        </choose>
    </where>
</select>
```

5）`set` 标签，专门给 UPDATE 用的，自动去掉末尾多余的逗号：

```xml
<update id="updateUser">
    UPDATE users
    <set>
        <if test="name != null">name = #{name},</if>
        <if test="email != null">email = #{email},</if>
    </set>
    WHERE id = #{id}
</update>
```

## 嵌套查询和关联查询

- **关联查询（嵌套结果映射）**：通过 Join 一次查出主对象和关联对象，再根据 `<id>` 等映射合并重复行。SQL 次数少，但结果集可能膨胀。
- **嵌套查询**：先查主对象，再为关联属性执行另一个映射语句，相当于拆成多次查询。结构简单，也**支持延迟加载**，但**容易产生 N+1 查询**。

![image-20261017153532774](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/image-20261017153532774.png)

### 关联查询案例



### 嵌套查询案例



### `<association>`和`<collection>`的区别

#### ⚠️ 容易混淆的地方

很多人容易把 `<association>` 和 `<collection>` 搞混：

| 标签                | 对应关系            | 举例                             | 代码表现                         |
| :------------------ | :------------------ | :------------------------------- | :------------------------------- |
| **`<association>`** | **一对一 / 多对一** | 学生 → 班级，学生 → 学生证       | 实体类属性是**单个对象**         |
| **`<collection>`**  | **一对多**          | 班级 → 多个学生，订单 → 多个商品 | 实体类属性是**集合（List/Set）** |

**判断口诀**：看你的**实体类属性**是**单个对象**（用 association）还是**集合**（用 collection）。如果是单个对象，那它就是一对一/多对一。



#### MySQL层面和Mybatis(Java)层面看待一对多和多对一

>  在**mysql层面一对多和多对一是一个意思**，**在mybatis中一对多和多对一处理方式不同！！！！**

这是一个理解数据库关系映射非常关键的认知分水岭。我们把它彻底拆开来看：

##### 1. MySQL 层面：物理结构上是“同一个东西”

在关系型数据库中，**多对一**和**一对多**描述的其实是**同一条外键约束**，只是看问题的方向不同。

假设有两张表：

```sql
-- 班级表
CREATE TABLE class (id INT PRIMARY KEY, name VARCHAR(20));
-- 学生表（带有外键 class_id）
CREATE TABLE student (id INT PRIMARY KEY, name VARCHAR(20), class_id INT);
```



- **从学生往班级看**：多个学生对应一个班级，这是**多对一**。
- **从班级往学生看**：一个班级包含多个学生，这是**一对多**。

**在底层，它们就是靠 `student.class_id` 这同一根线连起来的，没有任何区别。** 数据库不关心你的叫法，它只知道 `student` 引用了 `class`。

------

##### 2. MyBatis 层面：业务代码中必须“分开处理”

因为 Java 是面向对象的，一个对象里面装什么，决定了 MyBatis 必须怎么映射。

还是上面那个例子，在 Java 里我们会定义两个实体类：

```java
// 学生类（站在学生的角度看，班级只有一个）
public class Student {
    private Integer id;
    private String name;
    private Class clazz;      // 单个对象！
}

// 班级类（站在班级的角度看，学生有一群）
public class Class {
    private Integer id;
    private String name;
    private List<Student> students; // 集合！
}
```



MyBatis 看到这两个类，就必须用完全不同的方式来处理：

| 场景                       | 实体类属性类型         | MyBatis 映射标签 | SQL 查询思路                                         |
| :------------------------- | :--------------------- | :--------------- | :--------------------------------------------------- |
| **多对一**（查学生带班级） | 单个对象 (`Class`)     | `<association>`  | 通常用 `LEFT JOIN` 查出来，或者额外查一次(嵌套查询)  |
| **一对多**（查班级带学生） | 集合 (`List<Student>`) | `<collection>`   | 通常用 `LEFT JOIN` 查出多行，MyBatis 帮你合并成 List |

### N加1问题

嵌套查询容易出现N＋1问题：假设查 100 个用户的订单信息，主查询 1 次查出 100 个用户，每个用户再触发 1 次子查询去查订单，总共 101 次 SQL。如果关联层级更深，比如订单里还关联了商品详情，那 SQL 次数还会继续翻倍，数据库直接被打爆。

嵌套结果映射就没这个问题，一条 JOIN SQL 全搞定。

**但 JOIN 也有代价**：如果关联表数据量很大，JOIN 出来的中间结果集会膨胀得很厉害，一个用户有 50 个订单，用户数据就重复 50 行，网络传输和内存消耗都大。

#### 解决办法：

- 换成嵌套结果映射，即JOIN查询
- 开启懒加载==缓解==

### 懒加载

MyBatis 给嵌套查询提供了懒加载来缓解 N+1 问题。

全局配置 `lazyLoadingEnabled=true` 后，**子查询不会在主查询执行时就触发，只有真正访问关联属性的 getter 方法时才会去查**。













