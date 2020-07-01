---
layout: post
title:  "记一次PageHelper5.x的踩坑记录！"
categories: [mybatis,it]
excerpt: PageHelper5.x相对于4.x版本在拦截器的实现上进行了改变，4.x是对于4个参数的query方法做了拦截，5.x版本后同时对4个参数、6个参数的方法做了拦截，接下来我们具体看下

---

> 起因：公司内部对于数据安全有要求，要求所有敏感数据入库都需进行脱敏处理，然后提供了统一的加密组件，在我们开发项目过程中，小组成员突然告知，入库数据能够正常加密，查询插入后的数据没有解密，经过不断的排查才发现是因为PageHelper拦截器的问题导致的。

对于Mybatis集成PageHelper有过经验的人应该知道，pageHelper主要是拦截Executor的query方法 ，而Executor的query方法有两个：

```java
  <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey cacheKey, BoundSql boundSql) throws SQLException;

  <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException;
```

两个方法参数不一样，如果有过查看源码经验的朋友应该知道，第一个方法是被第二个方法内部调用的，所以一般实现mybatis插件都会拦截第二个方法，包括我前面提到的我们公司的加密组件及PageHelper5.x之前的版本。我们先看下一般情况下mybatis拦截器的执行数据，如下我们配置了三个拦截器及拦截器的加载顺序：

```java
//Interceptor1：
@Intercepts({
	@Signature(type = Executor.class, method = "query", args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class})
})
public class Interceptor1 implements Interceptor {}
//Interceptor2：
@Intercepts({
	@Signature(type = Executor.class, method = "query", args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class})
})
public class Interceptor2 implements Interceptor {}
//Interceptor3：
@Intercepts({
	@Signature(type = Executor.class, method = "query", args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class})
})
public class Interceptor2 implements Interceptor {}

//加载顺序
@Bean
public Interceptor1 getPageInterceptor(){return new Interceptor1();}
@Bean
public Interceptor2 getPageInterceptor(){return new Interceptor2();}
@Bean
public Interceptor3 getPageInterceptor(){return new Interceptor3();}
```

上面三个拦截器都是对Executor类四个参数query方法做拦截，三个拦截器的加载顺序是：Interceptor1>Interceptor2>Interceptor3

但是经过InterceptorChain.pluginAll()处理之后，拦截器的执行顺序变成了：

**前置处理Interceptor3>Interceptor2>Interceptor1>executor.query(4个参数方法)>后置处理Interceptor1>Interceptor2>Interceptor3**

用代码验证下，

我在拦截器的的intercrept方法内添加：

```java
// x对应拦截器类型的1，2，3
log.info("InterceptorX执行intercept>>>>");
```

并在plugin方法内添加：

```java
// x对应拦截器类型的1，2，3
log.info("InterceptorX执行plugin>>>>");
```

最终执行结果为：

```java
2020-03-27 00:13:23.646  INFO 33491 --- [           main] c.i.b.example.interceptor.Interceptor1   : Interceptor1执行plugin>>>>
2020-03-27 00:13:23.651  INFO 33491 --- [           main] c.i.b.example.interceptor.Interceptor2   : Interceptor2执行plugin>>>>
2020-03-27 00:13:23.651  INFO 33491 --- [           main] c.i.b.example.interceptor.Interceptor3   : Interceptor3执行plugin>>>>
2020-03-27 00:13:23.654  INFO 33491 --- [           main] c.i.b.example.interceptor.Interceptor3   : Interceptor3执行intercept>>>>
2020-03-27 00:13:23.654  INFO 33491 --- [           main] c.i.b.example.interceptor.Interceptor2   : Interceptor2执行intercept>>>>
2020-03-27 00:13:23.654  INFO 33491 --- [           main] c.i.b.example.interceptor.Interceptor1   : Interceptor1执行intercept>>>>

```

通过代码可以看到打印的日志结果跟上面的描述一样。

有兴趣的可以根据文末提供的源码地址看下我的代码，断点跟进去看下InterceptorChain类的pluginAll方法，源码中的单测类：UserQueryForInterceptorTest.testQuery 用于测试拦截器的加载及最终执行顺序。

好了，还是说回我们的PageHelper，上面说到了正常情况下，三个插件按照注入时的相反顺序执行拦截操作，我们在拦截器Interceptor2的下方注入分页插件的拦截器：

```java
 @Bean
    public Interceptor1 getInterceptor1(){
        return  new Interceptor1();
    }

    @Bean
    public Interceptor2 getInterceptor2(){
        return  new Interceptor2();
    }

    @Bean
    public PageInterceptor getPageInterceptor(){
        Properties properties=new Properties();
        properties.setProperty("helperDialect","mysql");
        properties.setProperty("reasonable","true");
        properties.setProperty("supportMethodsArguments","true");
        properties.setProperty("params","count=countSql");
        PageInterceptor pageInterceptor = new PageInterceptor();
        pageInterceptor.setProperties(properties);
        return pageInterceptor;
    }

    @Bean
    public Interceptor3 getInterceptor3(){
        return  new Interceptor3();
    }
```

然后再次执行代码看下日志打印：

```java
2020-03-27 00:24:43.371  INFO 33685 --- [           main] c.i.b.example.interceptor.Interceptor1   : Interceptor1执行plugin>>>>
2020-03-27 00:24:43.374  INFO 33685 --- [           main] c.i.b.example.interceptor.Interceptor2   : Interceptor2执行plugin>>>>
2020-03-27 00:24:43.374  INFO 33685 --- [           main] c.i.b.example.interceptor.Interceptor3   : Interceptor3执行plugin>>>>
2020-03-27 00:24:43.376  INFO 33685 --- [           main] c.i.b.example.interceptor.Interceptor3   : Interceptor3执行intercept>>>>
```

你会发现，只有Interceptor3执行完intercept的日志打印，Interceptor1，Interceptor2没有执行intercept方法，这两个拦截器失效了...**WTF？？?这是为啥！！！** 我当时也是很懵逼，这个问题还要从5.x的PageHelper的插件改变说起，看下分页插件的拦截器代码，下面代码简化了，只列出拦截器注释中拦截的executor的方法及关键代码执行的部分：

```java
@Intercepts(
    {
        @Signature(type = Executor.class, method = "query", args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}),
        @Signature(type = Executor.class, method = "query", args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class, CacheKey.class, BoundSql.class}),
    }
)
public class PageInterceptor implements Interceptor {
  ...
     @Override
    public Object intercept(Invocation invocation) throws Throwable {
        try {
          ...
    //由于逻辑关系，只会进入一次
    if(args.length == 4){
           //4 个参数时
           boundSql = ms.getBoundSql(parameter);
           cacheKey = executor.createCacheKey(ms, parameter, rowBounds, boundSql);
     } else {
                //6 个参数时
                cacheKey = (CacheKey) args[4];
                boundSql = (BoundSql) args[5];
      }
  ...
    if (!dialect.skip(ms, parameter, rowBounds)) {
                //反射获取动态参数
                String msId = ms.getId();
                Configuration configuration = ms.getConfiguration();
                Map<String, Object> additionalParameters = (Map<String, Object>) additionalParametersField.get(boundSql);
                //判断是否需要进行 count 查询
                if (dialect.beforeCount(ms, parameter, rowBounds)) {
                    ...
                //判断是否需要进行分页查询
                if (dialect.beforePage(ms, parameter, rowBounds)) {
                    ...
                    //执行分页查询
                    resultList = executor.query(ms, parameter, RowBounds.DEFAULT, resultHandler, pageKey, pageBoundSql);
                } else {
                    //不执行分页的情况下，也不执行内存分页
                    resultList = executor.query(ms, parameter, RowBounds.DEFAULT, resultHandler, cacheKey, boundSql);
                }
            } else {
                //rowBounds用参数值，不使用分页插件处理时，仍然支持默认的内存分页
                resultList = executor.query(ms, parameter, rowBounds, resultHandler, cacheKey, boundSql);
            }
}
```



从源码里面可以看到，PageInterceptor里面这一段代码：

```java
//由于逻辑关系，只会进入一次
    if(args.length == 4){
           //4 个参数时
           boundSql = ms.getBoundSql(parameter);
           cacheKey = executor.createCacheKey(ms, parameter, rowBounds, boundSql);
     } else {
                //6 个参数时
                cacheKey = (CacheKey) args[4];
                boundSql = (BoundSql) args[5];
      }
```

如果这个插件配置的靠后，是通过 4 个参数方法进来的，我们就获取这两个对象。如果这个插件配置的靠前，已经被别的拦截器处理成 6 个参数的方法了， 那么我们直接从 args 中取出这两个参数直接使用即可。取出这两个参数就保证了当其他拦截器对这两个参数做过处理时，这两个参数在这里会继续生效。

那么问题就来了，当我们的分页拦截器加载两个拦截四个参数的query方法的拦截器中间时，也就是注入顺序：

**Interceptor1>Interceptor2>PageInterceptor>Interceptor3**

这时，调用顺序就变了，Interceptor3 执行顺序如下：

```java
Interceptor3 前置处理      
Object result = QueryInterceptor.query(4个参数方法);     
Interceptor3 后续处理   
return result;
```

PageInterceptor 执行逻辑如下：

```java
PageInterceptor 前置处理
//根据源码分析只会进入一次4个参数拦截方法，所以执行的是6个参数的方法
Object result = executor.query(6个参数方法);     
PageInterceptor 后续处理   
return result;
```

由于此时PageInterceptor最终执行了6个参数的query方法，后续的Interceptor1、Interceptor2两个针对于4个参数的query方法的拦截就无法生效了，这也是为什么没有日志打印。

鉴于这种情况，有两种方案解决：

1.如果使用的Interceptor1、2、3这种插件是个人编写的，建议跟分页拦截器保持一致，同时拦截4个参数和6个参数的query方法并做对应的判断处理。

2.如果使用的拦截器是别人提供的组件内部的，这个时候就要调整拦截器注入的顺序了，保证在分页插件之后执行的拦截器是对6个参数的拦截，否则，就将分页插件放到最后一步执行，换句话说，就是注入的顺序保证分页插件在第一位。

源码可以访问我的github项目地址：[源码地址](https://github.com/gdspw/blog-demo)

如有疑问，可以在下方评论区留言，我收到留言会及时回复，多谢！

