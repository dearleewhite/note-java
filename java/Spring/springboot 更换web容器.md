# 概述

https://techlog.cn/article/list/10183221

此前我们介绍了如何通过 springboot 构建一个最简单的 web 项目

[基于 springBoot 的 Hello World](https://techlog.cn/article/list/10182886)

我们看到，通过 springboot 我们可以快速、简便地构建一个 web 项目，我们只需通过命令：

1

java -jar hello.jar

 

加载 jar 包就可以启动整个 web 项目，十分方便

 

# 原理

为什么 SpringBoot 可以不依赖外部容器加载呢？这是因为 SpringBoot 内嵌了一个 tomcat 容器

我们运行 mvn dependency:tree 可以看到：

[![img](https://techlog.cn/article/list/images/a0f5907c6ca4ba59ab63ec6dc5c435c7.png?id=3378046&v=1)](https://techlog.cn/article/list/images/a0f5907c6ca4ba59ab63ec6dc5c435c7.png)

 

 

org.springframework.boot:spring-boot-starter-tomcat 这个包就是 springboot 官方改造后的内置 tomcat

因此我们要想替换内置的 tomcat 容器，首先需要排除掉这个依赖：

1

2

3

4

5

6

7

8

9

10

11

12

<!-- spring-boot web -->

<dependency>

​    <groupId>org.springframework.boot</groupId>

​    <artifactId>spring-boot-starter-web</artifactId>

​    <!-- 排除内置容器，排除内置容器导出成war包可以让外部容器运行spring

​    -boot项目-->

​    <exclusions>

​        <exclusion>

​            <groupId>org.springframework.boot</groupId>

​            <artifactId>spring-boot-starter-tomcat</artifactId>

​        </exclusion>

​    </exclusions>

</dependency>

 

然后我们再引入所需的容器即可

 

# 将内置 tomcat 替换为内置 jetty

在排除掉内置 tomcat 容器以后，我们引入内置 jetty 依赖：

1

2

3

4

<dependency>  

​    <groupId>org.springframework.boot</groupId>  

​    <artifactId>spring-boot-starter-jetty</artifactId>  

</dependency>

 

 

官方还内置了 undertow 容器，需要注意的是他不支持 jsp

1

2

3

4

<dependency>  

​    <groupId>org.springframework.boot</groupId>  

​    <artifactId>spring-boot-starter-undertow</artifactId>  

</dependency>

 

 

# 使用外部容器

那如何通过外部容器启动 SpringBoot 项目呢？

在排除内置 tomcat 容器以后，必须让 SpringBoot 项目的入口类实现 SpringBootServletInitializer 接口的 config 方法

1

2

3

4

5

6

7

8

9

10

11

12

13

14

15

16

17

@SpringBootApplication

@ComponentScan

public class Application extends SpringBootServletInitializer {

​    /**

​     \* 实现SpringBootServletInitializer可以让spring-boot项目在web容器中运行

​     */

​    @Override

​    protected SpringApplicationBuilder configure(SpringApplicationBuilder 

​    builder) {

​        builder.sources(this.getClass());

​        return super.configure(builder);

​    }

​    public static void main(String[] args) {

​        SpringApplication.run(Application.class, args);

​    }

}

 

SpringBootServletInitializer 实现了 WebApplicationInitializer 接口，这是外部容器加载的入口，它定义了 SpringBoot 的启动加载方法

这样以后，jar 包或 war 包就可以直接放入外部容器运行了

 

# 配置 SpringBoot 项目启动端口

默认的 SpringBoot 启动端口是 8080，有三种方法可以实现自定义启动端口

 



### 入口类实现 ConfigurableEmbeddedServletContainer 接口



1

2

3

4

5

6

7

8

9

10

11

12

13

14

15

16

17

18

19

20

@RestController

@EnableAutoConfiguration

public class CustomPortController implements 

  EmbeddedServletContainerCustomizer {

​    /**

​     \* 自定义端口

​     \* @param container

​     */

​    public void customize(ConfigurableEmbeddedServletContainer container) {

​        container.setPort(8888);

​    }

​    @RequestMapping("/")

​    public String setPort(){

​        return "自定义端口:8888";

​    }

​    public static void main(String[] args) {

​        SpringApplication.run(CustomPortController.class,args);

​    }

}

 

除此之外 EmbeddedServletContainerCustomizer 还提供了很多其他功能，例如设置会话超时等，是一个非常实用的服务器设置类

除了实现 EmbeddedServletContainerCustomizer 接口，创建一个 EmbeddedServletContainerCustomizer 类型的 bean 也是同样有效的

 



### 自定义 ConfigurableEmbeddedServletContainer 类



通过创建 EmbeddedServletContainerFactory 类型的 bean，生成 ConfigurableEmbeddedServletContainer，我们就可以通过工厂方法设置其属性了，与上面的方法相同，这也是一个可以灵活配置服务器多种参数的方法

1

2

3

4

5

6

7

8

@Bean

public EmbeddedServletContainerFactory servletContainer() {

​    TomcatEmbeddedServletContainerFactory factory = new 

​    TomcatEmbeddedServletContainerFactory();

​    factory.setPort(9000);

​    factory.setSessionTimeout(10, TimeUnit.MINUTES);

​    factory.addErrorPages(new ErrorPage(HttpStatus.NOT_FOUND, "/notfound.html");

​    return factory;

}

 

 



### 配置文件



最简单的方法就是直接在 main/resources/application.properties 中添加一行：

1

server.port=8888

 

 

同样，我们也可以通过配置 main/resources/application.yml 来实现：

1

2

server:

  port: 8888

 