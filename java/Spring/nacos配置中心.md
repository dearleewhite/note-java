# 一、什么是Nacos

英文全称Dynamic Naming and Configuration Service，Na为naming/nameServer即注册中心,co为configuration即注册中心，service是指该注册/配置中心都是以服务为核心。服务在nacos是一等公民

# 二、Nacos原理

![img](https:////upload-images.jianshu.io/upload_images/18465434-6e52eea204e1dd68.png?imageMogr2/auto-orient/strip|imageView2/2/w/804/format/webp)

nacos简单介绍



Nacos注册中心分为server与client，server采用Java编写，为client提供注册发现服务与配置服务。而client可以用多语言实现，client与微服务嵌套在一起，nacos提供sdk和openApi，如果没有sdk也可以根据openApi手动写服务注册与发现和配置拉取的逻辑



![img](https:////upload-images.jianshu.io/upload_images/18465434-4969f32b90966ec1.png?imageMogr2/auto-orient/strip|imageView2/2/w/432/format/webp)

nacos服务领域模型



Nacos服务领域模型主要分为命名空间、集群、服务。在下图的分级存储模型可以看到，在服务级别，保存了健康检查开关、元数据、路由机制、保护阈值等设置，而集群保存了健康检查模式、元数据、同步机制等数据，实例保存了该实例的ip、端口、权重、健康检查状态、下线状态、元数据、响应时间。这些数据的作用会在第三章讲到

![img](https:////upload-images.jianshu.io/upload_images/18465434-755eec44672473e9.png?imageMogr2/auto-orient/strip|imageView2/2/w/632/format/webp)

image.png

## 2.1注册中心原理

![img](https:////upload-images.jianshu.io/upload_images/18465434-d17f9ad8f6655a0a.png?imageMogr2/auto-orient/strip|imageView2/2/w/814/format/webp)

服务注册原理

服务注册方法：以Java nacos client v1.0.1  为例子，服务注册的策略的是每5秒向nacos server发送一次心跳，心跳带上了服务名，服务ip，服务端口等信息。同时 nacos server也会向client 主动发起健康检查，支持tcp/http检查。如果15秒内无心跳且健康检查失败则认为实例不健康，如果30秒内健康检查失败则剔除实例。

## 2.2 配置中心原理

![img](https:////upload-images.jianshu.io/upload_images/18465434-28cf0f9e370f5b64.png?imageMogr2/auto-orient/strip|imageView2/2/w/575/format/webp)

image.png

# 三、 Nacos使用方法

## 3.1创建命名空间

不同的命名空间逻辑上是隔离的，不特殊设置的情况下，服务不会跨命名空间请求，命名空间主要的作用是区分服务使用的范围，比如开发、测试、生产、灰度可以分别设置四个命名空间来互相隔离。

![img](https:////upload-images.jianshu.io/upload_images/18465434-64026cf9401b2b68.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

新建命名空间


 如图所示，在控制台的 **服务管理-命名空间-新建命名空间**按钮可以创建新的命名空间，命名空间创建后，会在列表显示**命名空间ID**，这个ID后面会用在服务的配置文件中



## 3.2在服务上配置注册、配置中心

以springcloud为例，首先用maven导入nacos clinet的依赖：



```xml
     <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
            <version>0.2.1.RELEASE</version>
            <exclusions>
                <exclusion>
                    <groupId>com.alibaba.nacos</groupId>
                    <artifactId>nacos-client</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
            <version>0.2.1.RELEASE</version>
            <exclusions>
                <exclusion>
                    <groupId>com.alibaba.nacos</groupId>
                    <artifactId>nacos-client</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
     
```

先导入springcloud的alibaba-nacos-config和alibaba-nacos-discovery两个依赖，这两个依赖是用于nacos clinet与cloud结合的工具，0.2.x对应springboot 2.x.x ,0.1.x对应springboot 1.x.x。这两个组件可以和各种版本的nacos-client结合。把其中的nacos-clinet依赖给排除，引入想要引入的nacosclinet版本，如下：



```xml
   <!-- https://mvnrepository.com/artifact/com.alibaba.nacos/nacos-client -->
        <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-client</artifactId>
            <version>1.0.1</version>
        </dependency>
```

在bootstrap.properties上添加配置中心的配置



```csharp
spring.cloud.nacos.config.server-addr=nacos.e.189.cn:80
spring.cloud.nacos.config.namespace=命名空间id
```

在application-xxx.properties新增如下配置



```csharp
spring.cloud.nacos.discovery.server-addr=nacos.e.189.cn:80
spring.cloud.nacos.discovery.namespace=命名空间id
```

如果springboot启动类没有`@EnableDiscover`注解则加上
 完成如上更改，即可使用Nacos注册/配置服务

## 3.3服务间使用feign调用

演示：
 使用Feign、Ribbon均可，在这不做过多介绍

## 3.4通过配置更改动态刷新参数

普通application参数在配置中心直接配置皆可，如果需要可以动态刷新的配置，需要在相应类上加上`@RefreshScope`注解,示例如下，当在nacos配置中心更改配置后，方法getId的值也会刷新。



```java
@RefreshScope
public class IdEntity {
    @Value("${id}")
    private int id;
    public int getId(){
        return this.id;
    }
}
```

**配置中心参数修改/设置**
 如下两张图：在nacos控制台的**配置管理-配置列表**中顶部选择相应的命名空间，点击列表右上角的加号新增配置，Data ID 为   项目名-{spring.profiles.active}.properties,Group如果在bootstrap.properties中不指定则填默认的DEFAULT_GROUP，描述写该配置的描述，配置内容填写Properties格式或者Yaml格式。

![img](https:////upload-images.jianshu.io/upload_images/18465434-3c95ba344dd092d2.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

nacos控制台-配置管理



![img](https:////upload-images.jianshu.io/upload_images/18465434-1b6b4184210e5267.png?imageMogr2/auto-orient/strip|imageView2/2/w/694/format/webp)

nacos控制台-新建配置



## 3.5 nacos其他功能使用与介绍

### 控制台手动上下线实例

在控制台的**服务管理-服务列表**选择一个服务点击详情，在下方的集群列表可以看到有上线/下线按钮，点击即可以对该实例执行上线/下线操作，下线后的实例不会被请求

![img](https:////upload-images.jianshu.io/upload_images/18465434-d62393c18aa3d520.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

控制台手动上下线



### 配置实例权重

可以通过手动配置权重来控制流量，当一个集群内两个实例，权重越高，到达该实例的请求比例越多。



![img](https:////upload-images.jianshu.io/upload_images/18465434-768bde0776881a3a.png?imageMogr2/auto-orient/strip|imageView2/2/w/592/format/webp)

image.png



权重的初始值是1

### 配置保护阈值

保护阈值的范围是0~1
 服务的健康比例=服务的健康实例/总实例个数
 当服务健康比例<=保护阈值时候，无论实例健不健康都会返回给调用方
 当服务健康比例>保护阈值的时候，只会返回健康实例给调用方
 在**服务管理-服务列表**选择一个服务点击详情可以配置

![img](https:////upload-images.jianshu.io/upload_images/18465434-7e9c126ca65274aa.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

image.png



作者：一个没有感情的程序员
链接：https://www.jianshu.com/p/39ade28c150d
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。