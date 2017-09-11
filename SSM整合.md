
## Spring整合Mybatis过程中遇到的问题汇总 ##

**第一步** 加载所需要的jar包 自行下载和mvn都可以

还是一样先上一段教. 整合Spring + Mybatis中用到Jar包 ( Spring 相关jar包不多描述)
```
<dependency>
  <groupId>org.mybatis</groupId>
  <artifactId>mybatis</artifactId>
  <version>3.4.1</version>
</dependency>
  <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <version>5.1.38</version>
  </dependency>
  <dependency>
      <groupId>org.mybatis</groupId>
      <artifactId>mybatis-spring</artifactId>
      <version>1.3.0</version>
  </dependency>
```

可能出现的 ==问题1==  **运行报错** 
```
java.lang.AbstractMethodError: org.mybatis.spring.transaction.SpringManagedTransaction.getTimeout()Ljava/lang/Integer; 报错解决
```

问题原因是因为 mybatis-spring 和 spring-* 和 mybatis 不兼容问题
推荐 
- spring 4.3+
- mybatis 3.4+
- mybatis-spring 1.3.0

---

**第二步整合Dao层**

首先在项目中规划好包. 如Dao层、po层 本文整合后的结构如下

![image](http://wx2.sinaimg.cn/mw690/4d662fa1gy1fjfoem8lejj209m09rq2z.jpg)

mapper中定义了dao接口 po中定义了实体类

继续在resource目录下创建一个db.properties,把一些数据库连接配置信息放到里面
```
jdbc.driver=com.mysql.jdbc.Driver
jdbc.url=jdbc:mysql://120.25.162.238:3306/mybatis001?characterEncoding=utf-8
jdbc.username=root
jdbc.password=123
```

在resource目录下建立spring目录用来保存一些上下文配置信息

先来配置spring和mybatis的整合 新建applicationContext-dao.xml

在applicationContext-dao.xml 中我们将配置

- 数据源
- SqlSessionFactory
- mapper的扫描器

完整配置如下:

```xml
<beans
    xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xmlns:mvc="http://www.springframework.org/schema/mvc"
    xsi:schemaLocation="
    http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/spring-beans.xsd
    http://www.springframework.org/schema/mvc
    http://www.springframework.org/schema/mvc/spring-mvc-4.3.xsd
    http://www.springframework.org/schema/context
    http://www.springframework.org/schema/context/spring-context-4.3.xsd" >

    <context:component-scan base-package="cn.jing.hello.web.mapper"/>
    <!-- 加载db.properties文件中的内容，db.properties文件中key命名要有一定的特殊规则 -->
    <context:property-placeholder location="classpath:db.properties" />
    <!-- 配置数据源 ，dbcp -->

    <bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
        <property name="driverClassName" value="${jdbc.driver}"/>
        <property name="url" value="${jdbc.url}"/>
        <property name="username" value="${jdbc.username}" />
        <property name="password" value="${jdbc.password}" />
        <property name="maxActive" value="30" />
        <property name="maxIdle" value="5" />
    </bean>

    <!-- 从整合包里找，org.mybatis:mybatis-spring:1.2.4 -->
    <!-- sqlSessionFactory -->
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <!-- 数据库连接池 -->
        <property name="dataSource" ref="dataSource" />
        <!-- 加载mybatis的全局配置文件 -->
        <!--<property name="configLocation" value="classpath:mybatis/sqlMapConfig.xml" />-->
        <property name="mapperLocations" value="classpath:mybatis/*.xml" />
        <!-- 巨坑: 之前把mapper.xml放到了 cn.jing.hello.web.mapper 路径下 导致mvn后 在target下没有对应的xml文件 所以转移至了 resource目录下的某个文件夹里 -->
    </bean>
    <!-- mapper扫描器 -->
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <!-- 扫描包路径，如果需要扫描多个包，中间使用半角逗号隔开 -->
        <property name="basePackage" value="cn.jing.hello.web.mapper"/>
        <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory" />
    </bean>

    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"/>
    </bean>
</beans>
```

当然生成mapper.xml 和对应的 mapper接口 可以通过 Mybatis官方提供的逆向生成类进行。请参考另一篇文章 《使用Mybatis逆向生成器生成代码》

==问题2== 报错org.apache.ibatis.binding.BindingException: Invalid bound statement (not found)

其实此处报错在mybatis中非常常见 通常可以通过以下步骤逐一检查解决

1. 检查xml文件所在的package名称是否和interface对应的package名称一一对应
2. 检查xml文件的namespace是否和xml文件的package名称一一对应
3. 检查函数名称能否对应上
4. 去掉xml文件中的中文注释
5. 随意在xml文件中加一个空格或者空行然后保存

然而有一种非常隐蔽的错误就是使用了mvn，在集成代码后在target目录下并没有生成对应的xml文件

==巨坑: 之前把mapper.xml放到了 cn.jing.hello.web.mapper 路径下 导致mvn后 在target下没有对应的xml文件 所以转移至了 resource目录下的某个文件夹里==


---

接下来就是整合Service了 此处比较简单 未遇到难题直接代码展示

在resources/spring下创建applicationContext-service.xml，文件中配置service。

```xml
<beans
        xmlns="http://www.springframework.org/schema/beans"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="
    http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/spring-beans.xsd" >

    <!-- 配置service -->
    <bean id="userService" class="cn.jing.hello.web.service.UserServiceImpl"/>

</beans>
```

在UserServiceImpl中 通过@Autowired注解去定义了一个dao，dao完成整个业务处理

因为dao的配置中指定了 mapper自动扫描, 所以此处可以完成自动的装配过程

```java
package cn.jing.hello.web.service;

import cn.jing.hello.web.mapper.RlUserMapperCustom;
import cn.jing.hello.web.po.RlUser;
import org.springframework.beans.factory.annotation.Autowired;

public class UserServiceImpl implements UserService {

    @Autowired
    private RlUserMapperCustom rlUserMapperCustom;


    public RlUser getByUserId(int userId) throws Exception {
        System.out.println("加载的资源是" + rlUserMapperCustom);
        return rlUserMapperCustom.getByUserId(userId);
    }
}

```


最后整合 springmvc 和 spring mybatis

在web.xml中加上如下关键代码 来完成spring容器的注册

```xml
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:spring/applicationContext-*.xml</param-value>
</context-param>
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

这段xml做了以下几件事
- 加载spring容器的监听 
- 使用通配符的方式加载了resources/spring/applicationContext-*.xml
