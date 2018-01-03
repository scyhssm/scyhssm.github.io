---
layout:    post
title:      "maven+spring+springMVC+mybatis框架搭建学习"
subtitle:   "Learn maven+spring+springMVC+mybatis"
date:       2017-12-10 12:00:00
author:     "scyhssm"
header-img: "img/ssm.jpg"
header-mask: 0.3
catalog:    true
tags:
    - j2ee
    - ssm
---

>最近实训项目要用到ssm，不写一篇博客记录感觉学到的东西会忘掉。

## spring+springMVC+mybatis运行机制
1. 首先浏览器上回生成一个请求  
2. web.xml会对请求进行拦截并分发给DispatcherServlet  
3. DispatcherServlet会根据springMVC配置把请求交给Controller类处理，要对该类进行实例化  
4. 实例化Controller类需要注入ServiceImpl。依赖注入，组件不需要以硬编码耦合在一起，由系统提供所需的实例  
5. 实例化ServiceImpl时，注入Mapper  
6. 根据ApplicationContext.xml配置信息将Mapper和mapping.xml文件关联
7. 得到实例化的controller，就可以调用其内部的方法
8. 在调用的方法中，访问service，获取数据，把数据传到jsp
9. 在jsp中显示数据
整个ssm运行机制基本如此，另外我们还用到了mysql作为测试数据库，实现在idea上，eclipse现在越来越不好用了。  
## maven项目

maven是apache下的一个项目，用于项目管理和自动构建。基于项目对象模型，利用一个中央信息片段管理项目的构建、报告和文档。现在的javaee web项目基本都是基于maven的。  
如一般的dynamic web项目需要把包引入，maven只要在pom.xml中修改，要方便不少。  
pom.xml，以下内容自动生成  
* groupId，项目或者组织者的唯一标识，配置时生成路径也是基于此
* artifactId，项目的名称
* version，项目的版本
* packaging，打包机制，javaee web一般打包为war，war是用于web项目的打包方式，tomcat会识别war包并自动部署，当packaging为pom时用于合成多个项目
* classifier分类  

在dependency中  
* groupId，artifactId，version，描述了依赖的项目jar包唯一标志
* type，依赖包形式，如war或者是jar
* scope，限制依赖范围。compile默认，用于编译。test，用于test任务使用
* systemPath，仅用于范围为system，提供相应路径
* optional，当项目自身也依赖，用于连续依赖  

maven有项目继承的功能，父项目可以定义通用的包，子类直接引用，方便大型项目的构建。maven还有聚合项目的能力，可以将独立的maven项目构成整体。比如大型项目可能会把dao层，model层，service层，controller层都分为独立的模块，这时候可以聚合各个模块。一些通用的模块可以在下次项目时使复用，独立出来便于复用。  

build用于编译设置，如指定resources位置，目标文件的文件名。reporting包含的属性对应到site阶段，特定的maven插件能够产生定义和配置在reporting元素下的报告。  

这里要说以下maven的生命周期，典型的Maven构建生命周期由prepare-resources，compile，package，install阶段的序列组成（实际有很多构建阶段）。当maven开始构建工程，会按照定义阶段序列的顺序执行每个阶段注册的目标，三个标准的生命周期clean，default，site。如clean包含多个构建阶段，pre-clean,clean,post-clean。  

另外pom.xml还包含了一些元素，如资源、build时执行的插件、插件管理和reporting输出报表等。  
列出代码
```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>mrs</groupId>
  <artifactId>ssm</artifactId>
  <packaging>war</packaging>
  <version>1.0-SNAPSHOT</version>
  <name>ssm Maven Webapp</name>
  <url>http://maven.apache.org</url>
  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>

    <!-- spring版本号 -->
    <spring.version>4.2.5.RELEASE</spring.version>

    <!-- mybatis版本号 -->
    <mybatis.version>3.2.8</mybatis.version>

    <!-- mysql驱动版本号 -->
    <mysql-driver.version>5.1.29</mysql-driver.version>

    <!-- log4j日志包版本号 -->
    <slf4j.version>1.7.18</slf4j.version>
    <log4j.version>1.2.17</log4j.version>

  </properties>
  <dependencies>
    <dependency>
      <groupId>jstl</groupId>
      <artifactId>jstl</artifactId>
      <version>1.2</version>
    </dependency>

    <dependency>
      <groupId>javax</groupId>
      <artifactId>javaee-api</artifactId>
      <version>8.0</version>
    </dependency>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>3.8.1</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-core</artifactId>
      <version>${spring.version}</version>
    </dependency>
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-web</artifactId>
      <version>${spring.version}</version>
    </dependency>
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-oxm</artifactId>
      <version>${spring.version}</version>
    </dependency>
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-tx</artifactId>
      <version>${spring.version}</version>
    </dependency>
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-jdbc</artifactId>
      <version>${spring.version}</version>
    </dependency>
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-webmvc</artifactId>
      <version>${spring.version}</version>
    </dependency>
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-context</artifactId>
      <version>${spring.version}</version>
    </dependency>
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-context-support</artifactId>
      <version>${spring.version}</version>
    </dependency>
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-aop</artifactId>
      <version>${spring.version}</version>
    </dependency>

    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-test</artifactId>
      <version>${spring.version}</version>
    </dependency>

    <!-- 添加mybatis依赖 -->
    <dependency>
      <groupId>org.mybatis</groupId>
      <artifactId>mybatis</artifactId>
      <version>${mybatis.version}</version>
    </dependency>

    <!-- 添加mybatis/spring整合包依赖 -->
    <dependency>
      <groupId>org.mybatis</groupId>
      <artifactId>mybatis-spring</artifactId>
      <version>1.2.2</version>
    </dependency>

    <!-- 添加mysql驱动依赖 -->
    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <version>${mysql-driver.version}</version>
    </dependency>
    <!-- 添加数据库连接池依赖 -->
    <dependency>
      <groupId>commons-dbcp</groupId>
      <artifactId>commons-dbcp</artifactId>
      <version>1.2.2</version>
    </dependency>

    <!-- 添加fastjson -->
    <dependency>
      <groupId>com.alibaba</groupId>
      <artifactId>fastjson</artifactId>
      <version>1.1.41</version>
    </dependency>

    <!-- 添加日志相关jar包 -->
    <dependency>
      <groupId>log4j</groupId>
      <artifactId>log4j</artifactId>
      <version>${log4j.version}</version>
    </dependency>
    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-api</artifactId>
      <version>${slf4j.version}</version>
    </dependency>
    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-log4j12</artifactId>
      <version>${slf4j.version}</version>
    </dependency>

    <!-- log end -->
    <!-- 映入JSON -->
    <dependency>
      <groupId>org.codehaus.jackson</groupId>
      <artifactId>jackson-mapper-asl</artifactId>
      <version>1.9.13</version>
    </dependency>
    <!-- https://mvnrepository.com/artifact/com.fasterxml.jackson.core/jackson-core -->
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-core</artifactId>
      <version>2.8.0</version>
    </dependency>
    <!-- https://mvnrepository.com/artifact/com.fasterxml.jackson.core/jackson-databind -->
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
      <version>2.8.0</version>
    </dependency>

    <dependency>
      <groupId>commons-fileupload</groupId>
      <artifactId>commons-fileupload</artifactId>
      <version>1.3.1</version>
    </dependency>

    <dependency>
      <groupId>commons-io</groupId>
      <artifactId>commons-io</artifactId>
      <version>2.4</version>
    </dependency>

    <dependency>
      <groupId>commons-codec</groupId>
      <artifactId>commons-codec</artifactId>
      <version>1.9</version>
    </dependency>
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-core</artifactId>
      <version>2.8.0</version>
    </dependency>
    <!-- https://mvnrepository.com/artifact/com.fasterxml.jackson.core/jackson-databind -->
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
      <version>2.8.0</version>
    </dependency>
  </dependencies>
  <build>
    <finalName>ssm</finalName>
  </build>
</project>
```
## jdbc.properties和log4j文件
jdbc.properties
```
driverClass=com.mysql.jdbc.Driver
jdbcUrl=jdbc:mysql://localhost:3306/db_mrs?useUnicode=true&characterEncoding=utf-8&zeroDateTimeBehavior=convertToNull
username=root
password=scyhssm

#定义初始连接数
initialSize=0
#定义最大连接数
maxActive=20
#定义最大空闲
maxIdle=20
#定义最小空闲
minIdle=1
#定义最长等待时间
maxWait=60000
```
log4j.properties
```
log4j.rootLogger=INFO,Console,File

#控制台日志
log4j.appender.Console=org.apache.log4j.ConsoleAppender
log4j.appender.Console.Target=System.out
log4j.appender.Console.layout=org.apache.log4j.PatternLayout
log4j.appender.Console.layout.ConversionPattern=[%p][%t][%d{yyyy-MM-dd HH\:mm\:ss}][%C] - %m%n

#普通文件日志
log4j.appender.File=org.apache.log4j.RollingFileAppender
log4j.appender.File.File=logs/ssm.log
log4j.appender.File.MaxFileSize=10MB
#输出日志，如果换成DEBUG表示输出DEBUG以上级别日志
log4j.appender.File.Threshold=ALL
log4j.appender.File.layout=org.apache.log4j.PatternLayout
log4j.appender.File.layout.ConversionPattern=[%p][%t][%d{yyyy-MM-dd HH\:mm\:ss}][%C] - %m%n
```
## web.xml
springmvc的配置文件web.xml起到至关重要的作用，可以说前端控制器起到最主要的作用。  

spring框架提供过滤器CharacterEncodingFilter，针对浏览器请求过滤，再加入了父类没有的功能处理字符编码。encoding设置编码格式，forceEncoding设置是否理会request.getCharacterEncoding()方法，若为true则强制覆盖之前的编码格式。  

项目中使用spring，spring-mybatis.xml中没有配置BeanFactory，要在业务层引用spring容器管理的bean可以在web.xml配置监听器ContextLoaderListener，部署spring-mybatis.xml文件。  

ContextLoaderListener作用是启动web容器时，自动装配spring-mybatis.xml的配置信息，它实现了ServletContextListener接口。ContextLoader创建的是XmlWebApplicationContext类，实现的接口是XmlWebApplicationContext->ConfigurableWebApplicationContext->ApplicationContext->BeanFactory。

部署spring-mybatis.xml文件，自定义文件名需要加入contextConfigLocation参数。

配置DefaultServlet拦截请求，servlet-mapping里是拦截类型，写在DispatcherServlet前可以让拦截器先拦截请求，请求就不会进入spring了。

DispatcherServlet拦截匹配请求，把拦截下来的请求依据规则分发到controller。可以设置启动顺序，指定配置文件名spring-MVC.xml，拦截规则。

最后是一些欢迎页面，404错误，error的跳转和会话超时错误的时间设置。
```
<!DOCTYPE web-app PUBLIC
 "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
 "http://java.sun.com/dtd/web-app_2_3.dtd" >

<web-app xmlns="http://java.sun.com/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://java.sun.com/xml/ns/javaee
          http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
         version="3.0">
  <display-name>ssm</display-name>
  <context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:spring-mybatis.xml</param-value>
  </context-param>

  <context-param>
    <param-name>log4jConfigLocation</param-name>
    <param-value>classpath:log4j.properties</param-value>
  </context-param>

  <!-- 编码过滤器 -->
  <filter>
    <filter-name>encodingFilter</filter-name>
    <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
    <init-param>
      <param-name>encoding</param-name>
      <param-value>UTF-8</param-value>
    </init-param>
  </filter>
  <filter-mapping>
    <filter-name>encodingFilter</filter-name>
    <url-pattern>/*</url-pattern>
  </filter-mapping>

  <!-- spring监听器 -->
  <listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>

  <!-- 防止spring内存溢出监听器，比如quartz -->
  <listener>
    <listener-class>org.springframework.web.util.IntrospectorCleanupListener</listener-class>
  </listener>


  <!-- spring mvc servlet-->
  <servlet>
    <servlet-name>SpringMVC</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
      <param-name>contextConfigLocation</param-name>
      <param-value>classpath:spring-mvc.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
    <async-supported>true</async-supported>
  </servlet>
  <servlet-mapping>
    <servlet-name>SpringMVC</servlet-name>
    <!-- 此处也可以配置成 *.do 形式 -->
    <url-pattern>/</url-pattern>
  </servlet-mapping>

  <welcome-file-list>
    <welcome-file>/index.jsp</welcome-file>
  </welcome-file-list>

  <!-- session配置 -->
  <session-config>
    <session-timeout>15</session-timeout>
  </session-config>
</web-app>
```
## spring-mvc.xml
1. 可以开启注解
2. 开启mvc的注解驱动
3. 自动扫描controller包下所有类，扫描controller的位置
4. 定义跳转文件的前后缀，视图模式配置
5. 文件上传配置
6. spring mvc拦截器
7. 异常处理类等

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                        http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
                        http://www.springframework.org/schema/context
                        http://www.springframework.org/schema/context/spring-context-4.0.xsd
                        ">

    <!-- 自动扫描  @Controller-->
    <context:component-scan base-package="ssm.controller"/>

    <!--避免IE执行AJAX时，返回JSON出现下载文件 -->
    <bean id="mappingJacksonHttpMessageConverter"
          class="org.springframework.http.converter.json.MappingJackson2HttpMessageConverter">
        <property name="supportedMediaTypes">
            <list>
                <value>text/html;charset=UTF-8</value>
            </list>
        </property>
    </bean>
    <!-- 启动SpringMVC的注解功能，完成请求和注解POJO的映射 -->
    <bean class="org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter">
        <property name="messageConverters">
            <list>
                <ref bean="mappingJacksonHttpMessageConverter"/> <!-- JSON转换器 -->
            </list>
        </property>
    </bean>



    <!-- 定义跳转的文件的前后缀 ，视图模式配置 -->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="prefix" value="/WEB-INF/jsp/" />
        <property name="suffix" value=".jsp"/>
    </bean>

    <!-- 文件上传配置 -->
    <bean id="multipartResolver" class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
        <!-- 默认编码 -->
        <property name="defaultEncoding" value="UTF-8"/>
        <!-- 上传文件大小限制为31M，31*1024*1024 -->
        <property name="maxUploadSize" value="32505856"/>
        <!-- 内存中的最大值 -->
        <property name="maxInMemorySize" value="4096"/>
    </bean>
</beans>
```
## spring-mybatis.xml
省去mybatis的配置文件
1. 配置自动扫描，将注解类自动转化为Bean，Bean注入
2. 加载数据资源属性文件
3. 配置数据源
4. 配置sqlsessionfactory
5. 装配Dao接口
6. 声明事务管理
7. 注解事务切面，事务对数据库的概念，业务lu逻辑才是实现

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context" xmlns:tx="http://www.springframework.org/schema/tx"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                        http://www.springframework.org/schema/beans/spring-beans-3.1.xsd
                        http://www.springframework.org/schema/context
                        http://www.springframework.org/schema/context/spring-context-3.1.xsd
                        http://www.springframework.org/schema/tx
                        http://www.springframework.org/schema/tx/spring-tx.xsd">

    <!-- 自动扫描 -->
    <context:component-scan base-package="ssm"/>

    <!-- 第一种方式：加载一个properties文件 -->
    <bean id="propertyConfigurer" class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
        <property name="location" value="classpath:jdbc.properties"/>
    </bean>


    <!-- 第二种方式：加载多个properties文件
    <bean id="configProperties" class="org.springframework.beans.factory.config.PropertiesFactoryBean">
        <property name="locations">
            <list>
                <value>classpath:jdbc.properties</value>
                <value>classpath:common.properties</value>
            </list>
        </property>
        <property name="fileEncoding" value="UTF-8"/>
    </bean>
    <bean id="propertyConfigurer" class="org.springframework.beans.factory.config.PreferencesPlaceholderConfigurer">
        <property name="properties" ref="configProperties"/>
    </bean>
    -->

    <!-- 配置数据源 -->
    <bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource"
          destroy-method="close">
        <property name="driverClassName" value="${driverClass}"/>
        <property name="url" value="${jdbcUrl}"/>
        <property name="username" value="${username}"/>
        <property name="password" value="${password}"/>
        <!-- 初始化连接大小 -->
        <property name="initialSize" value="${initialSize}"/>
        <!-- 连接池最大数量 -->
        <property name="maxActive" value="${maxActive}"/>
        <!-- 连接池最大空闲 -->
        <property name="maxIdle" value="${maxIdle}"/>
        <!-- 连接池最小空闲 -->
        <property name="minIdle" value="${minIdle}"/>
        <!-- 获取连接最大等待时间 -->
        <property name="maxWait" value="${maxWait}"/>
    </bean>

    <!-- mybatis和spring完美整合，不需要mybatis的配置映射文件 -->
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="dataSource"/>
        <!-- 自动扫描mapping.xml文件 -->
        <property name="mapperLocations">
            <list>
                <value>classpath:mapping/*.xml</value>
            </list>
        </property>
    </bean>

    <!-- DAO接口所在包名，Spring会自动查找其下的类 -->
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="basePackage" value="ssm.dao"/>
        <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>
    </bean>


    <!-- (事务管理)transaction manager, use JtaTransactionManager for global tx -->
    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"/>
    </bean>

    <!-- 注解事务切面 -->
    <tx:annotation-driven transaction-manager="transactionManager"/>
</beans>
```
## 参考
[ SSM框架整合 ](http://blog.csdn.net/gallenzhang/article/details/51932152),[ SSM三大框架整合详细步骤 ](http://how2j.cn/k/ssm/ssm-tutorial/1137.html?tid=77#nowhere)
