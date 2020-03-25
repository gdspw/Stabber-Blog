---
layout: post
title:  "Mybatis与Mapper4结合，提高开发效率!"
categories: [java,mybatis,mysql]
tags: [mybatis,mysql,java]
---
>**背景**：项目的日常迭代，总会涉及到对于已有表的结构调整，每次表结构的变更都会涉及到model对应xml的变更，每次的变更都是小心翼翼，以求不影响已有的功能，着实令人头疼；直到遇见Mapper4，才解决之一难题。

---
通用Mapper4是一个可以实现任意MyBatis通用方法的框架，项目提供了常规的增删盖茶操作以及Example相关的单表操作，是为了解决Mybati使用的90%的基本操作，使用它可以很方便的进行开发，可以节省开发人员大量的时间。
本文主要是对通用Mapper做下样例的介绍及依赖引入、配置等，希望能够对于各位阅读的同学有些许帮助。

## 1. 先看下样例
数据库如下表：

```
CREATE TABLE `country` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `countryname` varchar(255) DEFAULT NULL COMMENT '名称',
  `countrycode` varchar(255) DEFAULT NULL COMMENT '代码',
  PRIMARY KEY (`Id`)
) ENGINE=InnoDB AUTO_INCREMENT=10011 DEFAULT CHARSET=utf8 COMMENT='国家信息';
```

对应的Java实体类型如下：
```
public class Country {
    @Id
    private Integer id;
    private String  countryname;
    private String  countrycode;

    //省略 getter 和 setter
}
```
最简单的情况下，只需要一个@Id 标记为主键即可，数据库中的字段名和实体类的字段名是完全相同的，这中情况下实体和表可以直接映射。
**提醒：**如果实体类中没有一个标记 @Id 的字段，当你使用带有 ByPrimaryKey 的方法时，所有的字段会作为联合主键来使用，也就会出现类似 
```
where id = ? and countryname = ? and countrycode = ? 的情况
```
通用Mapper提供了大量的通用接口，这里以最常用的Mapper接口为例
该实体类对应的dao层接口如下：
```
import tk.mybatis.mapper.common.Mapper;

public interface CountryMapper extends Mapper<Country> {}

```
该接口默认集成的方法如下：
- selectOne
- select
- selectAll
- selectCount
- selectByPrimaryKey
- 等等......
从 MyBatis 中获取该接口后就可以直接使用：
```
//从 MyBatis 或者 Spring 中获取 countryMapper，然后调用 selectAll 方法
List<Country> countries = countryMapper.selectAll();
//根据主键查询
Country country = countryMapper.selectByPrimaryKey(1);
//或者使用对象传参，适用于1个字段或者多个字段联合主键使用
Country query = new Country();
query.setId(1);
country = countryMapper.selectByPrimaryKey(query);
```

如果想要增加自己写的方法，可以直接在ContryMapper中增加。
### 1.1 使用纯接口注解方式
```
import org.apache.ibatis.annotations.Select;
import tk.mybatis.mapper.common.Mapper;

public interface CountryMapper extends Mapper<Country> {
    @Select("select * from country where countryname = #{countryname}")
    Country selectByCountryName(String countryname);
}
```
复杂的sql也是可以的，这里只是一个简单的举例

### 1.2 如果使用XML方式，需要提供接口对应的XML文件
对，你没有看错，Mapper可以跟xml文件共存，看到这里是不是觉得很意外，哈哈哈！

例如提供了 CountryMapper.xml 文件，内容如下：
```
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="tk.mybatis.sample.mapper.CountryMapper">
    <select id="selectByCountryName" resultType="tk.mybatis.model.Country">
        select * from country where countryname = #{countryname}
    </select>
</mapper>
```
在接口中添加对应的方法：
```
import tk.mybatis.mapper.common.Mapper;

public interface CountryMapper extends Mapper<Country> {
    Country selectByCountryName(String countryname);
}
```
*在接口中添加其他方法的时候和只用 MyBatis 是完全一样的，但是需要注意，在对应的 XML 中，不能出现和继承接口中同名的方法！*
多态！

在接口中，只要不是通过注解来实现接口方法，接口是允许重名的，真正调用会使用通用 Mapper 提供的方法。

例如在上面 CountryMapper 中提供一个带分页的 selectAll 方法：

```
  public interface CountryMapper extends Mapper<Country> {
    List<Country> selectAll(RowBounds rowBounds);
}
```
在 Java 8 的接口中通过默认方法还能增加一些简单的间接调用方>法，例如：
```
public interface CountryMapper extends Mapper<Country> {
    //这个示例适合参考实现对乐观锁方法封装
    default void updateSuccess(Country country){
        Assert.assertEquals(1, updateByPrimaryKey(country));
    }
}
```

## 2. 集成方式
### 2.1 Spring集成
---
#### 2.1.1 添加依赖
通用Mapper支持Mybatis 3.2.4+，使用该通用Mapper的同学注意当前项目的Mybatis版本号
```
<dependency>
    <groupId>tk.mybatis</groupId>
    <artifactId>mapper</artifactId>
    <version>最新版本</version>
</dependency>
```
#### 2.1.2 XML配置-MapperScannerConfigurer
引入通用Mapper4依赖后，只需要替换XML原有的*org.mybatis.spring.mapper.MapperScannerConfigurer* 为*tk.mybatis.spring.mapper.MapperScannerConfigurer*，两者的唯一区别一个是 **tk** 一个是 **org**。
```
<bean class="tk.mybatis.spring.mapper.MapperScannerConfigurer">
    <property name="basePackage" value="扫描包名"/>
</bean>
```
如果需要对Mapper进行特殊配置，可以按照以下方式：
```
<bean class="tk.mybatis.spring.mapper.MapperScannerConfigurer">
    <property name="basePackage" value="tk.mybatis.mapper.mapper"/>
    <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>
    <property name="properties">
        <value>
            参数名=值
            参数名2=值2
            ...
        </value>
    </property>
</bean>
```

Spring引入更多的配置相关可以移步官方介绍：https://gitee.com/free/Mapper/wikis/1.2-spring?sort_id=208197

### 2.2 Spring Boot 集成
Spring Boot 在微服务领域中已经成为主流。

这里介绍通用 Mapper 如何同 Spring Boot 进行集成。

为了能适应各种情况的用法，这里也提供了多种集成方式，基本上分为两大类。

- 基于 starter 的自动配置
- 基于 @MapperScan 注解的手工配置

#### 2.2.1 mapper-spring-boot-starter
在 starter 的逻辑中，如果你没有使用 @MapperScan 注解，你就需要在你的接口上增加 @Mapper 注解，否则 MyBatis 无法判断扫描哪些接口。

这里的第一种用法没有用 @MapperScan 注解，所以你需要在所有接口上增加 @Mapper 注解。

以后会考虑增加其他方式。

你只需要添加通用 Mapper 提供的 starter 就完成了最基本的集成，依赖如下：
```
<dependency>
  <groupId>tk.mybatis</groupId>
  <artifactId>mapper-spring-boot-starter</artifactId>
  <version>版本号</version>
</dependency>
```
如果你需要对通用 Mapper 进行配置，你可以在 Spring Boot 的配置文件中配置 mapper. 前缀的配置。

例如在 yml 格式中配置：
```
mapper:
  mappers:
    - tk.mybatis.mapper.common.Mapper
    - tk.mybatis.mapper.common.Mapper2
  notEmpty: true
```
在 properties 配置中：
```
mapper.mappers=tk.mybatis.mapper.common.Mapper,tk.mybatis.mapper.common.Mapper2
mapper.notEmpty=true
```
#### 2.2.2 @MapperScan 注解配置
你可以给带有 @Configuration 的类配置该注解，或者直接配置到 Spring Boot 的启动类上，如下：
```
@tk.mybatis.spring.annotation.MapperScan(basePackages = "扫描包")
@SpringBootApplication
public class SampleMapperApplication implements CommandLineRunner {
```
**注意：这里使用的 tk.mybatis.spring.annotation.MapperScan !**


以上就是对于Mapper4的简要介绍，更多功能可参考官网：https://gitee.com/free/Mapper ,希望能够帮助到大家，减轻繁琐的重复的工作，提高coding效率，感谢阅读本文！
