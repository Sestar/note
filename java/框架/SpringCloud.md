# SpringCloud
</br>

## BootStrap配置文件

### BootStrapApplicationListener#onApplicationEnvent

```text
  1. 判断'spring.cloud.bootstrap.enable'是否关闭

     ‘spring.cloud.bootstrap.enable’代表是否允许读取自定义的配置文件, 默认true为打开状态。若需要读取默认配置文件,
     想直接在配置文件(application.yml)中配置enable属性为false也是没有效果。因为加载配置文件监听器(ConfigFileApplicationListener)
     的优先级比BootStrapApplicatonListener低(优先级看Listener.order, 越小越优先)。如果要BootStrapApplication读取enable属性,
     就必须让enable在BootStrapApplicationListener之前就加载进环境配置中, 最简单的方法就是将enable直接在program Arguments
     中配置, 即启动类的String[] args入参(加载时间是spring.factories的第二步Run Listeners, 早于
     第四步Application Listeners(BootStrapApplication, ConfigFileApplicationListener), args入参存放在
     CommandLinePropertySource的数据源中)


  2. 修改bootstrap配置文件

         第一步是加载classpath:路径下的bootstrap文件(文件名由${spring.cloud.bootstrap.name}决定, 默认"bootstrap")
     从源码可以看出environment.resolvePlaceholders("${spring.cloud.bootstrap.name:bootstrap}")可以看出默认
     bootstrap配置文件名为bootstrap, 如果有修改bootstrap配置文件名, 通过name属性修改(修改name属性和enable属性一样,
     需要在BootStrapApplicationListener之前加载)。比如在Args中添加--spring.cloud.bootstrap.name=spring-cloud,
     BootStrapApplicationContext就会加载classpath:和classpath:config/路径下的spring-cloud.properties文件,
     文件加载顺序是由外到里, 先加载classpath:再加载classpath:config/, 所以classpath:config/路径下的配置文件
     会覆盖classpath:文件内容(MutablePropertySources#addFirst), 当读取配置内容时, 将会读取加载顺序最后的配置文件内容,
     即配置内容以最后加载的文件内容为主, 亦可以看成读取配置优先级是由里到外, 优先级逐渐降低。

         第二步是加载${spring.cloud.bootstrap.location}指定的文件, (默认为空字符串, 即不再加载另外bootstrap文件)
     在通过BootStrapApplicationListener#bootstrapServiceContext加载${spring.cloud.bootstrap.location}指定的文件,
     样例: 修改location属性(与修改name属性和enable属性一样, 需要在BootStrapApplicationListener之前加载),
     修改成: ··.location=classpath:configDemo/spring-cloud.yml, 会加载resources/configDemo/spring-cloud.yml文件。
     由于该文件相较于之前通过${spring.cloud.bootstrap.name}加载的配置文件, 最后加载, 所以该文件的优先级最高。

  3. 配置文件配置属性生效注意

     当application.yml和bootstrap.yml文件都存在同一个属性, 则application.yml的属性生效。相对于bootstrap.yml文件,
     application.yml文件后加载, 所以application.yml文件配置优先级比bootstrap.yml高。
     (bootstrap.yml可以理解成系统级别的一些参数配置，这些参数一般是不会变动的。application.yml可以用来定义应用级别的。)
```

### 源码

```Java
public void onApplicationEvent(ApplicationEnvironmentPreparedEvent event) {
    ConfigurableEnvironment environment = event.getEnvironment();
    // 判断spring.cloud.bootstrap配置开关是否打开
    if (!environment.getProperty("spring.cloud.bootstrap.enabled", Boolean.class, true)) {
        return;
    }
    // 这里暂时不知道作用到底是什么？？？
    // don't listen to events in a bootstrap context
    if (environment.getPropertySources().contains(BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
        return;
    }
    ConfigurableApplicationContext context = null;
    // 读取spring.cloud.bootstrap.name属性, 默认返回bootstrap, 如果有修改配置名则返回修改的名字
    String configName = environment.resolvePlaceholders("${spring.cloud.bootstrap.name:bootstrap}");
    // 初始化ApplicationContext, 这里暂时不知道作用到底是什么？？？
    for (ApplicationContextInitializer<?> initializer : event.getSpringApplication().getInitializers()) {
        if (initializer instanceof ParentContextApplicationContextInitializer) {
            context = findBootstrapContext((ParentContextApplicationContextInitializer) initializer, configName);
        }
    }
    if (context == null) {
        // 定位bootstrap配置文件, spring.cloud.bootstrap.location返回BootStrap的ApplicationContext
    	    context = bootstrapServiceContext(environment, event.getSpringApplication(), configName);
    }
    apply(context, event.getSpringApplication(), environment);
}
```
</br>

## Actuator
</br>

### 路径鉴权

HTTP方法 | 路径 | 描述 | 鉴权
-- | -- | -- | --
GET | /autoconfig | 查看自动配置的使用情况 | true
GET | /configprops | 查看配置属性，包括默认配置 | true
GET | /beans | 查看bean及其关系列表 | true
GET | /dump | 打印线程栈 | true
GET | /env | 查看所有环境变量 | true
GET | /env/{name} | 查看具体变量值 | true
GET | /health | 查看应用健康指标 | false
GET | /info | 查看应用信息 | false
GET | /mappings | 查看所有url映射 | true
GET | /metrics | 查看应用基本指标 | true
GET | /metrics/{name} | 查看具体指标 | true
POST | /shutdown | 关闭应用 | true
GET | /trace | 查看基本追踪信息 | true
</br>

### 鉴权配置
</br>

#### 鉴权安全配置(SecurityProperties)

```pom
    <!-- Actuator 保证安全性 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
```

```yml
security:
  basic:
    enabled: true        # 开启安全监测, 默认开启安全监测
  user:                  # 账号(用于网站登录)
    name: sestar         # 用户名, 默认user
    password: 123        # 密码,   默认随机
```

#### 鉴权url配置(ManagementServerProperties)

```pom
    <!-- Actuator 监控插件 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
```

```yml
management:
  context-path: /spring-actuator  # 默认路径是 '/', 如果配置路径没有配置端口, 无法访问; 如果配置路径和端口则配置可用
  port: 3772                      # 默认端口是同server一致
  security:                       # 每个鉴权不同, 具体可以查看actuator路径鉴权
    enabled: true                 # 默认开启, false则将所有鉴权关闭, 所有路径都可以访问
```

</br>

#### 关闭鉴权

* 统一关闭(ManagementServerProperties):

```properties
### 默认开启, false则将所有鉴权关闭, 所有路径都可以访问
management.security.enabled = true
```
* 关闭某一个鉴权(sensitive):

```properties
### restart Http重启服务
endpoints.restart.sensitive = false
endpoints.restart.enabled = true
### 关闭env鉴权
endpoints.env.sensitive = false
```
</br>

### 查看环境配置

```text
* /env                     数据来源        <- EnvironmentEndPoint
* /env/endpoints.env.*     Controller来源 <- EnvironmentMvcEndpoint
* /env?endpoints.env.*=*   POST访问, 修改属性值,修改的值会放在`manager`节点中, 且根据修改时间排序,最新则放在第一位
                          (EnvironmentManager#addFirst)
```

</br>

## 搭建简单的Spring Cloud

<br />

[参考源码](https://github.com/Sestar/firstLearn/tree/master/spring-cloud/simple-cloud)

<br />

### 配置服务器
</br>

> 以下方式都是使用Git获取服务器配置，当git仓库的配置发生了改变, 无需重启项目, 配置内容自动重新加载。

<br />

#### Maven依赖

```xml
<!-- spring-cloud-server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```
</br>

#### 基于本地 Git 不同环境配置管控
</br>

##### 在引导类添加注释@EnableConfigServer

```Java
package com.sestar.springcloudfirstlearn;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;

@SpringBootApplication
@EnableConfigServer
public class SpringCloudConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudConfigServerApplication.class, args);
    }
}
```
</br>

##### 创建不同环境的服务器配置文件

```text
- resources
-- config
--- segments
---- segment.properties          ## 默认环境配置, 使用默认环境(访问segment或者segment-default配置文件)
---- segment-test.properties     ## 测试环境配置
---- segment-prod.properties     ## 上线环境配置
```

```properties
## segment-test.properties
## 配置服务器应用名称, 测试环境(如果只写segment, 会识别为segment-default, 视为默认环境)
spring.cloud.config.name = segment-test

## 配置服务器端口
## 客户端的访问接口,客户端server.port和manager.port将会沿用这个端口
## 客户端可以覆盖manager.port端口,但是server.port无法覆盖
server.port = 3782
```
</br>

##### application.properties文件绑定服务器配置文件

```properties
## ${user.dir}定位项目根路径, 可以适用于不同操作系统。
## 如果不确定${user.dir},System.out.println(System.getProperty("user.dir"))
spring.cloud.config.server.git.uri = ${user.dir}\src\main\resources\config\segments
```
</br>

##### 配置文件改装成本地git

```git
> git init   ## 将目录改装成git仓库

> git add .  ## 将现有路径添加到git仓库

> git commit -m "Add Segment.properties"  ## 提交本次增加的三个properties文件
[master (root-commit) 16d057d] Add segment.properties
 3 files changed, 15 insertions(+)
 create mode 100644 segment-prod.properties
 create mode 100644 segment-test.properties
 create mode 100644 segment.properties
```
</br>

##### 测试配置是否正确

```text
http://localhost:3771/spring-cloud-server/segment/default  ## 使用默认环境

http://localhost:3771/spring-cloud-server/segment/test     ## 使用测试环境

http://localhost:3771/spring-cloud-server/segment/prod     ## 使用上线环境
```

测试环境中, 会出现两个properties,  一个是测试环境的，一个是默认环境的。  
默认环境配置是一直会读取, 如果指定环境没有默认环境的一些配置, 将会使用默认配置中的属性
```json
{
    name: "segment",
    profiles: [
        "test"
    ],
    label: null,
    version: "16d057d892495640256141bd43a0d54587e611d7",  ## git版本号
    state: null,
    propertySources: [{
        name: "...\spring-cloud-first-learn\src\main\resources\config\segments\segment-test.properties",
        source: {
            server.port: "3782",
            spring.cloud.config.name: "segments-test"
        }
    	}, {
        name: "...\spring-cloud-first-learn\src\main\resources\config\segments\segment.properties",
        source: {
            server.port: "3781",
            spring.cloud.config.name: "segments"
        }
    }]
}
```
<br />

##### 配置文件解析
```text
如果将segment-prod文件名改成segments-prod,
那么访问路径http://localhost:3771/spring-cloud-server/segment/prod -> 
http://localhost:3771/spring-cloud-server/segments/prod,
多了一个s, segments/prod其实就是segments-prod文件名,
而且加载的默认配置文件也是加载文件segments或者segments-default。
所以单独修改prod文件, 那么加载默认文件就会为空, 因为segments文件不存在
```

```json
{
    name: "segments",  ## 发生改变
    profiles: [
        "prod"
    ],
    label: null,
    version: "16d057d892495640256141bd43a0d54587e611d7",
    state: null,
    propertySources: [{
        name: "...\src\main\resources\config\segments\segments-prod.properties", ## 发生改变
        source: {
            server.port: "3782",
            ## 没有发生改变, 文件名改变, 但是文件内容不变, 'segment-prod'是文件内容
            spring.cloud.config.name: "segment-prod"
        }
    	}]
}
```
</br>

#### 基于远程 Git 不同环境配置管控
</br>

##### 在引导类添加注释@EnableConfigServer

```Java
package com.sestar.springcloudfirstlearn;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;

@SpringBootApplication
@EnableConfigServer
public class SpringCloudConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudConfigServerApplication.class, args);
    }
}
```
</br>

##### Git Clone 获取不同环境的服务器配置文件

```text
- segments
-- segment.properties          ## 默认环境配置
-- segment-test.properties     ## 测试环境配置
-- segment-prod.properties     ## 上线环境配置
```

```properties
## segment-test.properties
## 配置服务器应用名称, 测试环境(如果只写segment, 会识别为segment-default, 视为默认环境)
spring.cloud.config.name = segment-test

## 配置服务器端口
## 客户端的访问接口,客户端server.port和manager.port将会沿用这个端口
## 客户端可以覆盖manager.port端口,但是server.port无法覆盖
server.port = 3782
```
</br>

##### application.properties文件绑定服务器配置文件

```properties
## 基于远程git的服务器配置
### 连接git的用户名(如果是私有的git, 必须要配置账号信息, 若是公开的, 不用配置)
spring.cloud.config.server.git.username = userName
### 连接git的用户名密码
spring.cloud.config.server.git.password = 123
### git远程仓库配置
spring.cloud.config.server.git.uri = https://github.com/Sestar/cloud-config.git
### 强制拉取git内容, 如果本地副本是脏数据, 将强制从远程仓库拉取配置
spring.cloud.config.server.git.force-pull = true
```
</br>


##### 测试配置是否正确

```text
http://localhost:3771/spring-cloud-server/segment/default  ## 使用默认环境(访问segment或者segment-default配置文件)

http://localhost:3771/spring-cloud-server/segment/test     ## 使用测试环境

http://localhost:3771/spring-cloud-server/segment/prod     ## 使用上线环境
```

测试环境中, 会出现两个properties,  一个是测试环境的，一个是默认环境的。  
默认环境配置是一直会读取, 如果指定环境没有默认环境的一些配置, 将会使用默认配置中的属性
```json
{
    name: "segment",
    profiles: [
        "test"
    ],
    label: null,
    version: "ac9c220bdcc2528e694da6ed24f3df95f2eb2245",  ## git版本号
    state: null,
    propertySources: [{
        name: "...\spring-cloud-first-learn\src\main\resources\config\segments\segment-test.properties",
        source: {
            server.port: "3782",
            spring.cloud.config.name: "segments-test"
        }
    	}, {
        name: "...\spring-cloud-first-learn\src\main\resources\config\segments\segment.properties",
        source: {
            server.port: "3781",
            spring.cloud.config.name: "segments"
        }
    }]
}
```
</br>

##### 配置文件解析
```text
如果将segment-prod文件名改成segments-prod,
那么访问路径http://localhost:3771/spring-cloud-server/segment/prod -> 
http://localhost:3771/spring-cloud-server/segments/prod
多了一个s, segments/prod其实就是segments-prod文件名,
而且加载的默认配置文件也是加载文件segments或者segments-default。
所以单独修改prod文件, 那么加载默认文件就会为空, 因为segments文件不存在
```

```json
{
    name: "segments",  ## 发生改变
    profiles: [
        "prod"
    ],
    label: null,
    version: "16d057d892495640256141bd43a0d54587e611d7",
    state: null,
    propertySources: [{
        name: "...\src\main\resources\config\segments\segments-prod.properties", ## 发生改变
        source: {
            server.port: "3782",
            ## 没有发生改变, 文件名改变, 但是文件内容不变, 'segment-prod'是文件内容
            spring.cloud.config.name: "segment-prod"
        }
    	}]
}
```
<br />

### 配置客户端
<br />

#### Maven依赖

```xml
<!-- Spring-Cloud-Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```
<br />

#### application.properties

```properties
## 配置客户端应用名称
spring.application.name = spring-cloud-client-application

## 配置客户端应用端口
## 服务端spring.cloud.config.server的配置文件如果有server.port,
## 则application配置文件中的server.port无法覆盖配置
server.port = 3791
server.context-path = /spring-cloud-client

## 关闭管理端actuator的安全鉴权
management.security.enabled = false
#management.context-path = /spring-client-actuator
### 服务端spring.cloud.config.server的配置文件(如上服务端segment.properties文件)如果有server.port,
### application配置文件中的management.prot可以覆盖配置
management.port = 3792
```
<br />

#### bootstrap.properties

``` properties
## 配置客户端关联应用
## ‘segment’是服务端的spring.cloud.config.server.git.uri获取配置文件‘segment-test’的前缀‘segment’
## 如果没有配置spring.cloud.config.name,将会使用application.properties的${spring.application.name}
spring.cloud.config.name = segment

## 关联profile
## ‘test’是服务端的spring.cloud.config.server.git.uri获取配置文件‘segment-test’的后缀‘test’
## 客户端可以从服务器的segment-test配置文件中读取配置
spring.cloud.config.profile = test

## 关联label
## ‘master’是服务端的spring.cloud.config.server.git.uri的分支名称
spring.cloud.config.label = master

## 配置服务器URI
## ‘127.0.0.1:3771’是服务端IP加上服务端的application.properties配置server.port
## ‘spring-cloud-server’是服务端的application.properties配置server.context-path
spring.cloud.config.uri = http://127.0.0.1:3771/spring-cloud-server
```
<br />

#### 启动日志

```text
日志可以看出,
‘Fetching config from server at: http://127.0.0.1:3771/spring-cloud-server’说明可以从服务端抓取配置文件

‘Located environment: name=segment, profiles=[test]’定位到配置文件‘segment-test’

‘propertySources=[MapPropertySource {name='configClient'}, MapPropertySource {name='E:\coding\firstLearn\spring-cloud-config-server/src/main/resources/config/segments/segment-test.properties'}, MapPropertySource {name='E:\coding\firstLearn\spring-cloud-config-server/src/main/resources/config/segments/segment.properties'}]’
说明抓取到两个配置文件, segment-test.properties和segment.properties
```

```text
[           main] c.c.c.ConfigServicePropertySourceLocator : Fetching config from server at: http://127.0.0.1:3771/spring-cloud-server
[           main] c.c.c.ConfigServicePropertySourceLocator : Located environment: name=segment, profiles=[test], label=master, version=a70a20c1cd94559923bd608342d4a158d6ff4596, state=null
[           main] b.c.PropertySourceBootstrapConfiguration : Located property source: CompositePropertySource [name='configService', propertySources=[MapPropertySource {name='configClient'}, MapPropertySource {name='E:\coding\firstLearn\spring-cloud-config-server/src/main/resources/config/segments/segment-test.properties'}, MapPropertySource {name='E:\coding\firstLearn\spring-cloud-config-server/src/main/resources/config/segments/segment.properties'}]]
```
<br />

#### 测试配置是否正确

> 访问 http://localhost:3771/spring-client-actuator/env 查看环境配置

```json
configService:configClient: {
    config.client.version: "a70a20c1cd94559923bd608342d4a158d6ff4596"  ## git版本号
},
configService:.../src/main/resources/config/segments/segment-test.properties: {
    ## 抓取服务端测试环境配置, 因为客户端bootstrap.properties配置spring.cloud.config.profile = test
    spring.cloud.config.name: "segments-test"
},
configService:.../src/main/resources/config/segments/segment.properties: {
    ## 抓取服务端默认环境配置
    spring.cloud.config.name: "segments-test"
},
```

> 获取服务端segment.test的配置属性(比如segment-test文件中有配置segment.prop)
> 
> 访问 http://localhost:3792/spring-client-actuator/env/segment.*

```json
{
    segment.prop.name: "test",
    segment.prop.id: "002"
}
```

<br />

#### 客户端更新服务端配置
<br />

##### 手动更新服务端配置
<br />

> 启动服务端和客户端, 客户端查看 /env/segment.prop.id
> segment.prop.id属性是服务端spring.cloud.config.server.git.uri配置文件配置属性

```json
{
    segment.prop.id = 003
}
```

> 修改服务端配置  
> 具体文件需要看客户端使用服务端的哪个配置文件  
> 如下客户端的配置, 读取的就是服务端的 git.uri文件夹下的 'segment.test'文件  
> 修改segment.prop.id -> 110  

```properties
spring.cloud.config.name = segment
spring.cloud.config.profile = test
```

```properties
segment.prop.id = 110
```

> 再次访问客户端 /env/segment.prop.id  
> 发现还是没有修改, 是因为客户端不知道服务端已经修改了配置, 需要手动重新获取服务端配置  
> 使用客户端POST访问 /refresh, 发现返回一个数组，表示这两个属性是重新获取服务端配置后发现属性值发生改变  
> 可以查看 RefreshEndpoint#invoke  

```json
[
    "config.client.version",
    "segment.prop.id"
]
```
<br />

##### 定时更新服务端配置
<br />

> 由于服务端配置修改, 客户端不会感知, 可以通过 ContextRefresher#refresh 进行重新加载服务端配置  
> 可以在引导类中增加定时任务， 让contextRefrsher定时执行refresh从服务端拉取配置

```Java
package com.sestar.springcloudconfigclient;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.context.refresh.ContextRefresher;
import org.springframework.scheduling.annotation.Scheduled;

import java.util.Set;

@SpringBootApplication
public class SpringCloudConfigClientApplication {

    /**
     * 用以定时更新服务器端的配置
     **/
    private final ContextRefresher contextRefresher;

    @Autowired
    public SpringCloudConfigClientApplication(ContextRefresher contextRefresher) {
        this.contextRefresher = contextRefresher;
    }

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudConfigClientApplication.class, args);
    }

    /**
     * @description 定时更新服务端的配置(1秒)
     * @author zhangxinxin
     * @date 2018/12/28 15:05
     **/
    @Scheduled(fixedRate = 1000L)
    private void refresh() {
        Set<String> changeProp = contextRefresher.refresh();
        if (null != changeProp && changeProp.size() > 0) {
            System.out.println("=================change prop start====================");
            for (String prop : changeProp) {
                System.out.println(prop);
            }
            System.out.println("=================end prop start=======================");
        }
    }

}
```
<br />

## Eureka
<br />

### 搭建单个Eureka服务端, 并配有Cloud服务配置端

<br />

[参考源码](https://github.com/Sestar/firstLearn/tree/master/spring-cloud/spring-eureka)

<br />

#### 搭建 Eureka 服务端
<br />

##### Maven依赖

```xml
<!-- Actuator插件 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<!-- Eureka Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
```
<br />

##### 启动类注册为Eureka服务端

```Java
package com.sestar.springeurekaserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer
public class SpringEurekaServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringEurekaServerApplication.class, args);
    }

}
```
<br />

##### application.properties

```properties
spring.application.name = spring-eureka-server-application

## 访问配置
server.context-path = /spring-eureka-server
server.port = 3771

## Actuator
### Actuator 安全关闭
management.security.enabled = false
management.context-path = /eureka-server-config
management.port = 3772

## Eureka
### 取消向注册中心注册
eureka.client.register-with-eureka = false
### 取消向注册中心获取注册信息(服务, 实例信息)
eureka.client.fetch-registry = false
## 将应用注册到 Eureka 服务器, 解决 peer 问题
### 当*.hostname 和 *.serviceUrl.defaultZone的 protocol一致时, Eureka服务端将不会再去寻找其他Eureka服务端
eureka.instance.hostname = localhost
eureka.client.serviceUrl.defaultZone = http://${eureka.instance.hostname}:${server.port}/${server.context-path}/eureka
```
<br />

##### 测试 Eureka 服务端是否搭建成功

> 访问 http://127.0.0.1:3771/spring-eureka-server  
> 如果能够进入 Eureka 页面说明 Erueka服务端已经搭建成功

<br />

<img width="100%" src="/note/_v_images/java/框架/SpringCloud/Eureka服务端搭建成功.png">

<br /><br />

#### 搭建 Eureka 服务配置端
<br />

##### Maven依赖

```xml
<!-- Actuator插件 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<!-- Spring Cloud Server 应用作为 Spring Cloud 配置端 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
<!-- Eureka Client 用于应用被Eureka服务端发现 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```
<br />

##### 启动类注册为Spring Cloud配置端, Eureka客户端

```Java
package com.sestar.springconfigserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.config.server.EnableConfigServer;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;

@SpringBootApplication
@EnableConfigServer  // 注册为Spring Cloud配置端
@EnableEurekaClient  // 注册为Eureka客户端
public class SpringConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringConfigServerApplication.class, args);
    }

}
```
<br />

##### application.properties

```properties
spring.application.name = spring-config-server-application

server.context-path = /spring-config-server
server.port = 3791

## Actuator 安全关闭
management.security.enabled = false
management.context-path = /config-server-actuator
management.port = 3792

### git远程仓库配置
spring.cloud.config.server.git.uri = https://github.com/Sestar/cloud-config.git
## 强制拉取git内容, 如果本地副本是脏数据, 将强制从远程仓库拉取配置
spring.cloud.config.server.git.force-pull = true

## 应用注册到 Eureka 服务器
eureka.client.service-url.defaultZone = http://127.0.0.1:3771/spring-eureka-server/eureka
## 应用注册方式从默认模式(电脑名称)注册改成IP注册
## 启动项目可以查看控制台c.c.c.ConfigServicePropertySourceLocator :
## Fetching config from server at: 后面到底是电脑名还是IP
#eureka.instance.prefer-ip-address = true

## Banner
banner.location = config/banner.txt
```
<br />

##### 测试 Eureka 服务配置端是否搭建成功

> 再次访问 http://127.0.0.1:3771/spring-eureka-server  
> 如果 Instances currently registered with Eureka 如果有 Eureka 服务配置端的 spring.application.name  
> 则表示 Eureka 服务配置端已经在 Eureka 服务端注册成功

<br />

<img width="30%" src="/note/_v_images/java/框架/SpringCloud/Eureka服务配置端注册成功.png">

<br />

##### 测试 Eureka 服务配置端能否获取配置文件

> 访问http://127.0.0.1:3791/spring-config-server/segment/prod/master  
> 能够获取两个propertySource, 一个是prod(上线环境), 一个是default(默认环境)

```json
{
    name: "segment",
    profiles: [
        "prod"
    ],
    label: "master",
    version: "6b1013feb372c55994c5f4f9a20c24de6386a4c2",
    state: null,
    propertySources: [{
        name: "https://github.com/Sestar/spring-cloud-segment-config/segment-prod.properties",
        source: {
            server.port: "3783",
            spring.cloud.config.name: "segment-prod"
        }
    },
    {
        name: "https://github.com/Sestar/spring-cloud-segment-config/segment.properties",
        source: {
            server.port: "3781",
            spring.cloud.config.name: "segment"
        }
    }]
}
```

<br /><br />

#### 搭建 Eureka 客户端
<br />

##### Maven依赖

```xml
<!-- Actuator插件 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<!-- Spring Cloud Client 接收服务配置端的配置文件 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
<!-- Eureka Client 用于应用被Eureka服务端发现 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```
<br />

##### 启动类注册为 Eureka 客户端

```Java
package com.sestar.springeurekaclient;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;

@SpringBootApplication
@EnableEurekaClient
public class SpringEurekaClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringEurekaClientApplication.class, args);
    }

}
```
<br />

##### application.properties

```properties
spring.application.name = spring-eureka-client-application

## 访问配置
server.context-path = /spring-eureka-client
server.port = 3781

## Actuator
management.security.enabled = false
management.context-path = /eureka-client-config
management.port = 3782

##  Eureka
### spring-eureka-server-application注册到 Eureka 服务器中
### ${}都为服务器的配置内容 http://服务器地址:${server.port}/${server.context-path}/eureka
### 如果 bootstrap 有配置 spring.cloud.config.discovery.service-id, 则需要把该配置移到 bootstrap 中
### 因为*.discovery.service-id属性是查找 Eureka 服务配置端, 而查找范围就是eureka.client.service-url.defaultZone
### 需要最先加载 Eureka 注册信息, eureka.client.service-url.defaultZone才能确定范围,
### 而 *.discovery.service-id属性配置在 bootstrap 中, 比 application 配置先加载,
### 所以当有配置Eureka服务配置端, 则需要把 eureka.client.service-url.defaultZone必须配置在 bootstrap 中
eureka.client.service-url.defaultZone = http://127.0.0.1:3771/spring-eureka-server/eureka
```
<br />

##### bootstrap.properties

```properties
##  Eureka
### spring-eureka-server-application注册到 Eureka 服务器中
### ${}都为服务器的配置内容 http://服务器地址:${server.port}/${server.context-path}/eureka
### 如果 bootstrap 有配置 spring.cloud.config.discovery.service-id, 则需要把该配置移到 bootstrap 中
### 因为*.discovery.service-id属性是查找 Eureka 服务配置端, 而查找范围就是eureka.client.service-url.defaultZone
### 需要最先加载 Eureka 注册信息, eureka.client.service-url.defaultZone才能确定范围,
### 而 *.discovery.service-id属性配置在 bootstrap 中, 比 application 配置先加载,
### 所以当有配置Eureka服务配置端, 则需要把 eureka.client.service-url.defaultZone必须配置在 bootstrap 中
eureka.client.service-url.defaultZone = http://127.0.0.1:3771/spring-eureka-server/eureka
### 修改 http://localhost:3771/spring-eureka-server页面 Instances currently registered with Eureka的Status的访问路径
#eureka.instance.status-page-url-path = /info
### Instances currently registered with Eureka 的Status的状态为 /health 的status节点状态("UP", "DOWN"等)
#eureka.instance.health-check-url-path = /health

## 配置客户端关联应用
### ‘segment’是服务端的spring.cloud.config.server.git.uri获取配置文件‘segment-test’的前缀‘segment’
### Eureka 客户端加载配置文件是 *.service-id对应protocol:/ip:port + *.name + *.profile + *.label
### 本来git有segment-prod.properties文件, 但是因为Eureka服务端有设置server.context-path=spring-eureka-server,
### 所以访问配置文件路径为http:/localhost:3771/‘spring-eureka-server’/segment/prod/master
### *.name如果单写segment不够,只会访问http:/localhost:3771/segment/prod/master, 明显路径不对, 将会获取不到配置文件
spring.cloud.config.name = spring-config-server/segment
## 关联profile
## ‘test’是服务端的spring.cloud.config.server.git.uri获取配置文件‘segment-test’的后缀‘test’
spring.cloud.config.profile = prod
### 关联label
## ‘master’是服务端的spring.cloud.config.server.git.uri的分支名称
spring.cloud.config.label = master
## 允许本应用被服务器发现, *.server-id才会生效
spring.cloud.config.discovery.enabled = true
## 配置本应用的服务器名称, serverId为 Eureka服务配置端的spring.application.name
spring.cloud.config.discovery.service-id = spring-config-server-application
```
<br />

##### 测试客户端是否注册到服务端

> 访问客户端 http://localhost:3771/spring-eureka-server/
> 如果 Instances currently registered with Eureka 如果有 Eureka 客户端的 spring.application.name  
> 则表示 Eureka 客户端已经在 Eureka 服务端注册成功

<img width="100%" src="/note/_v_images/java/框架/SpringCloud/Eureka客户端注册成功.png">

<br /><br />

##### 查看是否获取到 Eureka 服务配置端的配置文件

> 从IDE的控制台中可以看出 Eureka Client 已经定位到了两个 propertySource文件, segment-prod和segment.properties

```text
c.c.c.ConfigServicePropertySourceLocator : Fetching config from server at: http://10.4.31.23:3791/
c.c.c.ConfigServicePropertySourceLocator : Located environment: name=segment, profiles=[prod], label=master, version=6b1013feb372c55994c5f4f9a20c24de6386a4c2, state=null
b.c.PropertySourceBootstrapConfiguration : Located property source: CompositePropertySource [name='configService', propertySources=[MapPropertySource {name='configClient'}, MapPropertySource {name='https://github.com/Sestar/spring-cloud-segment-config/segment-prod.properties'}, MapPropertySource {name='https://github.com/Sestar/spring-cloud-segment-config/segment.properties'}]]
```

<br />

### 搭建高可用的服务治理
<br />

[参考源码](https://github.com/Sestar/firstLearn/tree/master/spring-cloud/eureka-high-available)

<br />

#### 搭建 Eureka 服务端集群
<br />

##### Maven依赖

```xml
<!-- Actuator 插件 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<!-- Eureka Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
```
<br />

##### 启动类注册为Eureka服务端

```Java
package com.sestar.springeurekaserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer
public class SpringEurekaServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringEurekaServerApplication.class, args);
    }

}
```
<br />

##### application.properties

```properties
## 服务配置
server.context-path = /spring-eureka-server

## Actuator 安全管理关闭
management.security.enabled = false

## Eureka 公用配置
### 向注册中心注册
eureka.client.register-with-eureka = true
### 向注册中心获取注册信息
eureka.client.fetch-registry = true
```
<br />

##### 创建服务端集群(两个服务端)
<br />

> application-peer1.properties

```properties
## Run 应用的时候, 在 Args 添加 --spring.profiles.active=peer1, 
## 相当于启动 peer1 服务端, 加载application.properties 和 application-peer1.properties

## 应用名称
spring.application.name = spring-eureka-server-application-1

## peer 1 服务配置
server.port = 3771

## peer 2 服务配置
peer2.server.host = 127.0.0.1
peer2.server.port = 3772
peer2.spring.application.name = spring-eureka-server-application-2

## peer 1 向 peer 2 注册
eureka.client.serviceUrl.defaultZone = http://${peer2.server.host}:${peer2.server.port}${server.context-path}/eureka
```
<br />

> application-peer2.properties

```properties
## Run 应用的时候, 在 Args 添加 --spring.profiles.active=peer2, 
## 相当于启动 peer2 服务端, 加载application.properties 和 application-peer2.properties

## 应用名称
spring.application.name = spring-eureka-server-application-2

## peer 2 服务配置
server.port = 3772

## peer 1 服务配置
peer1.server.host = 127.0.0.1
peer1.server.port = 3771
peer1.spring.application.name = spring-eureka-server-application-1

## peer 2 向 peer 1 注册
eureka.client.serviceUrl.defaultZone = http://${peer1.server.host}:${peer1.server.port}${server.context-path}/eureka
```

<br />

##### 测试集群服务端

> 单独启动一个服务端, 肯定会出现Connection refused: connect错误, 毕竟另一个服务端访问不了, 就无法注册。   
> 当两个服务都启动起来后, 查看控制台是否有绑定注册中心的信息  
> 访问 http://127.0.0.1:3771/spring-eureka-server 和 http://127.0.0.1:3772/spring-eureka-server 的 Eureka页面   
> Instances currently registered with Eureka 实例都有两个, spring-eureka-server-application-1和*.application-2   
> 则说明服务端集群可以相互注册

```text
// 如果有下述两条日志信息, 说明两个Eureka 服务端已经布置好集群
// peer1 console
c.n.eureka.cluster.PeerEurekaNodes       : Adding new peer nodes [http://127.0.0.1:3771/spring-eureka-server/eureka/]
c.n.eureka.cluster.PeerEurekaNodes       : Replica node URL:  http://127.0.0.1:3771/spring-eureka-server/eureka/
// peer2 console
c.n.eureka.cluster.PeerEurekaNodes       : Adding new peer nodes [http://127.0.0.1:3772/spring-eureka-server/eureka/]
c.n.eureka.cluster.PeerEurekaNodes       : Replica node URL:  http://127.0.0.1:3772/spring-eureka-server/eureka/
```
<br />

#### 搭建Eureka客户端
<br />

##### Maven依赖

```xml
<!-- Actuator插件 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<!-- Eureka Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```
<br />

##### 启动类注册为 Eureka 客户端

```Java
package com.sestar.springeurekaclient;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;

@SpringBootApplication
@EnableEurekaClient
public class SpringEurekaClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringEurekaClientApplication.class, args);
    }

}
```
<br />

##### application.properties

```properties
## 应用名称
spring.application.name = spring-eureka-client-application

## 服务端口
server.context-path = /spring-eureka-client
server.port = 3781

## Actuator 安全管理关闭
management.security.enabled = false

## 配置连接多个Eureka服务端
## 配置多个 Eureka 注册服务中心， 以','隔开
eureka.client.service-url.defaultZone = \
  http://127.0.0.1:3771/spring-eureka-server/eureka,\
  http://127.0.0.1:3772/spring-eureka-server/eureka
```
<br />

#### 测试高可用性

> 高可用性就是一个客户端在服务端集群(多台服务端)都有记录  
> 客户端对其中一个服务端注册, 在一段时间后, 集群内所有服务端都会有这台客户端的注册信息  
> 实现即使有服务端不可用, 有另外的服务端可以顶替使用   
>  
> 访问 http://127.0.0.1:3771/spring-eureka-server 和 http://127.0.0.1:3772/spring-eureka-server 的 Eureka页面
> Instances currently registered with Eureka 实例都有三个, *.server-application-1和*.application-2, *.client-application  
> 则证明 Eureka 集群服务端搭建成功



### 状态页面
<br />

#### 状态页面配置

```properties
## 状态页面的地址 默认为 /info
## http://localhost:3771/spring-eureka-server页面 Instances currently registered with Eureka的Status的访问路径
eureka.instance.status-page-url-path = /info
## /health 的status节点状态为 Instances currently registered with Eureka 的Status的状态 ("UP", "DOWN"等)
eureka.instance.health-check-url-path = /health
```
<br />

#### 修改状态页面地址
<br />

##### bootstrap.properties

```properties
### 修改状态页面的地址 默认为 /info
### 修改 http://localhost:3771/spring-eureka-server页面 Instances currently registered with Eureka的Status的访问路径
eureka.instance.status-page-url-path = /health
```
<br />

##### 修改成功校验

<img width="100%" src="/note/_v_images/java/框架/SpringCloud/Eureka修改状态页面地址.png" />
<br /><br />

## Consul

[参考源码](https://github.com/Sestar/firstLearn/tree/master/spring-cloud/spring-consul)

<br />

### Consul使用步骤
<br />

#### Consul安装

> 访问 https://www.consul.io/downloads.html 进行下载 zip 包  
> 解压 consul.exe 程序, 直接使用命令行即可进行简单使用  

#### Consul指令

```text
consul agent -dev -ui: 单机启动附带网页版, 相当于consul agent -dev -ui -node 'LAPTOP-I5HBMU6M' -datacenter dc1
  默认启动参数：node为电脑名(启动控制台输出 consul: member 'LAPTOP-I5HBMU6M' joined, marking health alive),LAPTOP-I5HBMU6M就是电脑名, 被注册为node
  默认启动参数：datacenter为dc1
  访问 http://127.0.0.1:8500/ui(默认端口8500), 查看 Consul 页面
```

<br />

| 指令 | 功能 |
|:--|:--|
| consul -v | 显示consul版本 |
| consul agent -dev -ui | 单机启动附带网页版 |
| consul members | 显示consul的参数, node, Address, Status, Type, DC |
| consul kv put key value | consul的kv增加Map元素 |
| consul kv get key | 通过key获取consul中kv的value |

<br />

### 搭建简易 Consul
<br />

#### Consul 服务端

<br />

##### 启动 Consul Server

> 直接使用命令行启动Consul服务端: consul agent -dev -ui

<br />

##### 测试 Consul Server 启动是否成功

```text
访问 http://127.0.0.1:8500/ui, 
可以发现Service 节点有刚刚启动 Consul 的 Node实例(实例名consul),
并且实例的Health Nodes已经有一个客户端(实例名为本机名,其实就是本身)
进入实例的Health Checks界面可以看到健康检查正常
```

<img width="100%" src="/note/_v_images/java/框架/SpringCloud/Consul服务端搭建成功-1.png" />
<img width="100%" src="/note/_v_images/java/框架/SpringCloud/Consul服务端搭建成功-2.png" />

<br /><br />

#### Consul 客户端
<br />

##### Maven依赖

```xml
<!-- Consul Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
<!-- Actuator 插件 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

<br />

##### 启动类注册为 Consul 客户端

```Java
package com.sestar.springconsulclient;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

@SpringBootApplication
@EnableDiscoveryClient
public class SpringConsulClientApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringConsulClientApplication.class, args);
	}

}
```

<br />

##### application.properties

```properties
# 应用名称
spring.application.name = spring-consul-client-application

# 应用服务
server.context-path = /spring-consul-client
server.port = 3781

# Actuator安全管理关闭
management.security.enabled = false
management.context-path = /consul-client-actuator
management.port = 3782

# Consul服务配置
spring.cloud.consul.host = 127.0.0.1
spring.cloud.consul.port = 8500
## 默认的安全检查地址为 http://本机IP: ${server.port}/health
## 由于有配置server.context-path和management-context-path, 所以需要自定义healthCheckUrl
spring.cloud.consul.discovery.healthCheckUrl = http://${spring.cloud.consul.host}:${management.port}${management.context-path}/health
```

<br />

##### 测试 Consul 服务端是否注册成功

<br />

###### Consul Client Console

> 可以看到注册客户端信息, NewService节点内容

```text
o.s.c.c.s.ConsulServiceRegistry          : Registering service with consul: NewService{id='spring-consul-client-application-3781', name='spring-consul-client-application', tags=[contextPath=/spring-consul-client], address='LAPTOP-I5HBMU6M', port=3781, enableTagOverride=null, check=Check{script='null', interval='10s', ttl='null', http='http://127.0.0.1:3782/consul-client-actuator/health', tcp='null', timeout='null', deregisterCriticalServiceAfter='null', tlsSkipVerify=null, status='null'}, checks=null}
```

<br />

###### Consul Server Web

> 访问 http://127.0.0.1:8500/ui
> 发现 Services 节点多出了两个Service, 因为Client中server.port和management.port的端口不一致, Consul将会把这两个端口解析成两个
> 不同的Service Node, 但是两个Service Node 的 health check 的地址是同一个

<img width="80%" src="/note/_v_images/java/框架/SpringCloud/Consul客户端注册成功-1.png" />
<img width="80%" src="/note/_v_images/java/框架/SpringCloud/Consul客户端注册成功-2.png" />

<br /><br />

## 负载均衡(Eureka)

[参考源码](https://github.com/Sestar/firstLearn/tree/master/spring-cloud/eureka-load-balancing)

<br />

### ReadMeFirst

```text
源码包含两个项目:
    一个是使用Ribbon.listOfServers方式寻找服务提供端(server-provider),
    一个是使用Eureka服务发现寻找服务提供端

查看Ribbon自带方式寻找服务提供端项目:
    1. 无需搭建Eureka Server
    2. 服务提供端启动类无需注册Eureka Client(取消注释@EnableDiscoveryClient)
    3. 服务提供端application.properties启用对应配置
    4. Ribbon Client启动类无需注册Eureka Client(取消注释@EnableDiscoveryClient)
    5. Ribbon Client的application.properties启用对应配置

查看Eureka服务发现方式寻找服务提供端
    1. 搭建Eureka Server
    2. 服务提供端启动类注册Eureka Client(添加注释@EnableDiscoveryClient)
    3. 服务提供端application.properties启用对应配置
    4. Ribbon Client启动类注册Eureka Client(添加注释@EnableDiscoveryClient)
    5. Ribbon Client的application.properties启用对应配置
```

<br />

### Eureka Server

<br />

#### Maven依赖

```
<!-- Eureka Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
```

<br />

#### 启动类注册服务端

```Java
package com.sestar.springeurekaserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer
public class SpringEurekaServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringEurekaServerApplication.class, args);
	}

}
```

<br />

#### application.properties

```properties
# 应用名称
spring.application.name = spring-eureka-server-application

# 应用服务
server.context-path = /spring-eureka-server
server.port = 3781

# Actuator 安全管理
management.security.enabled = false
management.context-path = /eureka-server-actuator
management.port = 3782

## Eureka
### 取消向注册中心注册
eureka.client.register-with-eureka = false
### 取消向注册中心获取注册信息(服务, 实例信息)
eureka.client.fetch-registry = false
## 将应用注册到 Eureka 服务器, 解决 peer 问题
### 当*.hostname 和 *.serviceUrl.defaultZone的 protocol一致时, Eureka服务端将不会再去寻找其他Eureka服务端
eureka.instance.hostname = 127.0.0.1
eureka.client.serviceUrl.defaultZone = http://${eureka.instance.hostname}:${server.port}${server.context-path}/eureka
## 关闭Eureka Server自我保护， 以保证关闭的客户端能够过期
eureka.server.enable-self-preservation = false

# Banner
banner.location = config/banner.txt
```

<br />

#### 测试Eureka Server搭建成功

> 访问 http://127.0.0.1:3781/spring-eureka-server 能进入Eureka页面, 即Eureka Server搭建成功

<br />

### 搭建服务提供端

<br />

#### Maven依赖

<br />

```xml
<!-- Eureka Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
<!-- Actuator 插件 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

<br />

#### 启动类注册Eureka Client

```Java
package com.sestar.cloudserverprovider;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

@SpringBootApplication
//@EnableDiscoveryClient // 使用Eureka服务发现方式发现服务提供端
public class CloudServerProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(CloudServerProviderApplication.class, args);
    }

}
```

<br />

#### application.properties

```propertie
### 通用配置
## 应用名称
spring.application.name = cloud-server-provider-application

## 应用服务
server.context-path = /cloud-server-provider
server.port = 3771

## Actuator 安全管理
management.security.enabled = false
management.context-path = /server-provider-actuator
management.port = 3772

## Banner
banner.location = config/banner.txt



### 使用ribbon.listOfServers发现服务提供端
eureka.client.enabled = false



### 使用Eureka发现服务提供端
## Eureka注册
#eureka.client.serviceUrl.defaultZone = http://127.0.0.1:3781/spring-eureka-server/eureka
## 上传注册信息时间间隔
#eureka.client.instance-info-replication-interval-seconds = 5
## 获取注册信息时间间隔
#eureka.client.registry-fetch-interval-seconds = 5
## IP注册(Eureka默认为主机名注册)
#eureka.instance.prefer-ip-address = true
```

<br />

#### 新建测试服务

```Java
package com.sestar.cloudserverprovider.web.controller;

import com.sestar.cloudserverprovider.domain.User;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

/**
 * @description Server Provider Controller
 * @author zhangxinxin
 * @date 2019/1/10 14:30
 **/
@RestController
public class ServerProviderController {

    @Value("${server.port}")
    private String port;

    @PostMapping("/greeting")
    public String greeting(@RequestBody User user) {
        return "greeting:" + user.toString() + " port:" + port;
    }

}
```

<br />

#### 测试Controller正常

> 使用 POST 访问 http://127.0.0.1:3771/cloud-server-provider/greeting  
> json传参: {"name": "sestar", "id": 123}  
> 返回字符串为: greeting:User{name='sestar', id=123}  port: 3771 即为正常  

<br />

### Ribbon Client

<br />

#### Maven依赖

```xml
<!-- Eureka Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
<!-- Ribbon -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-ribbon</artifactId>
</dependency>
```

<br />

#### 启动类注册为Ribbon Client, Eureka Client(附带生成RestTemplate的Bean和加载LoadBalanced)

```Java
package com.sestar.springcloudribbonclient;

import com.netflix.loadbalancer.IRule;
import com.sestar.springcloudribbonclient.ribbon.MyRule;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.cloud.netflix.ribbon.RibbonClient;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

@SpringBootApplication
@RibbonClient(name = "cloud-server-provider-application")
//@EnableDiscoveryClient // 使用Eureka服务发现方式发现服务提供端
public class SpringCloudRibbonClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudRibbonClientApplication.class, args);
    }

    /**
     * @description 获取RestTemplate的Bean对象
     * @author zhangxinxin
     * @date 2019/1/10 14:43
     * @return org.springframework.web.client.RestTemplate
     **/
    @Bean
    @LoadBalanced
    public RestTemplate getRestTemplate() {
        return new RestTemplate();
    }

    /**
     * @description 使用自定义负载均衡规则(永远选择可达服务器的最后一台)
     *              使用Ribbon自带寻找服务提供端时, 使用自定义负载均衡规则
     * @author zhangxinxin
     * @date 2019/1/16 13:46
     * @return com.netflix.loadbalancer.IRule
     **/
    @Bean
    public IRule getRule () {
        return new MyRule();
    }

}
```

<br />

#### 自定义负载均衡规则(IRule)

```java
package com.sestar.springcloudribbonclient.ribbon;

import com.netflix.client.config.IClientConfig;
import com.netflix.loadbalancer.AbstractLoadBalancerRule;
import com.netflix.loadbalancer.Server;

import java.util.List;

/**
 * @description 自定义负载均衡规则，永远返回可达服务器的最后一台
 * @author zhangxinxin
 * @date 2019/1/16 11:04
 **/
public class MyRule extends AbstractLoadBalancerRule {

    @Override
    public void initWithNiwsConfig(IClientConfig clientConfig) {

    }

    /**
     * 永远返回可达服务器的最后一台
     **/
    @Override
    public Server choose(Object key) {
        List<Server> reachableServers = getLoadBalancer().getReachableServers();
        if (reachableServers.isEmpty()) {
            return null;
        }
        return reachableServers.get(reachableServers.size() - 1);
    }
}
```

<br />

#### 自定义负载均衡Ping(IPing)

```Java
package com.sestar.springcloudribbonclient.ribbon;

import com.netflix.loadbalancer.IPing;
import com.netflix.loadbalancer.Server;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.client.RestTemplate;
import org.springframework.web.util.UriComponentsBuilder;

import java.net.URI;

/**
 * @description 自定义负载均衡的Ping
 * @author zhangxinxin
 * @date 2019/1/23 10:12
 **/
public class MyPing implements IPing {

    @Autowired
    @Qualifier("simpleRestTemplate")
    private RestTemplate restTemplate;

    @Value("${service-provider.manager.context-path}")
    private String managerContextPath;

    /**
     * @description 通过判断Server的health情况来决定Server是否存活, 默认isAlive方法都是直接return true
     * @author zhangxinxin
     * @date 2019/1/23 10:13
     * @param server 服务对象
     * @return boolean
     **/
    @Override
    public boolean isAlive(Server server) {
        int port = server.getPort() + 10;
        URI uri = UriComponentsBuilder.newInstance().scheme("http").host(server.getHost())
                .port(port).path(managerContextPath + "/health").build().toUri();

        ResponseEntity entity = restTemplate.getForEntity(uri, String.class);
        return HttpStatus.OK.equals(entity.getStatusCode());
    }

}
```

<br />

#### application.properties

```properties
### 通用配置
# 应用名称
spring.application.name = cloud-ribbon-client-application

# 应用服务
server.context-path = /cloud-ribbon-client
server.port = 3791

# Actuator 安全管理
management.security.enabled = false
management.context-path = /ribbon-client-actuator
management.port = 3792

# Service-provider.properties
service-provider.server.host = 127.0.0.1
service-provider.server.context-path = /cloud-server-provider
service-provider.manager.context-path = /server-provider-actuator
#service-provider.manager.context-path = /cloud-server-provider
service-provider.application.name = cloud-server-provider-application

# Ribbon
## 配置名称可以看 org.springframework.cloud.netflix.ribbon.PropertiesFactory
cloud-server-provider-application.ribbon.NFLoadBalancerRuleClassName = com.sestar.springcloudribbonclient.ribbon.MyRule
cloud-server-provider-application.ribbon.NFLoadBalancerPingClassName = com.sestar.springcloudribbonclient.ribbon.MyPing

# Banner
banner.location = config/banner.txt



### 使用ribbon.listOfServers发现服务提供端
# 使用ribbon.listOfServers服务时, 不需要开启Eureka
#eureka.client.enabled = false

# Service-provider.properties
service-provider1.port = 3871
service-provider2.port = 3872
service-provider3.port = 3873

## 配置 Ribbon 服务提供方
## listOfServers会注入到LoadBalancerInterceptor中, protocol,host,port会被解析, 但是context-path不会解析
## LoadBalancerContext#reconstructURIWithServer,可以发现只会解析 scheme(protocol), host, port
## 简单来说就是即使将ribbon.listOfServers写成
## http://${service-provider.host}:${service-provider.port}${service-provider.context-path}
## 但是最后解析只会解析成http://${service-provider.host}:${service-provider.port},不会解析${service-provider.context-path}.
## 使用ribbon.listOfServers需要LoadBalancerInterceptor实例, 这里是用启动类在注册RestTemplate时,使用@LoadBalanced创建
#cloud-server-provider-application.ribbon.listOfServers = \
#  http://${service-provider.server.host}:${service-provider1.port},\
#  http://${service-provider.server.host}:${service-provider2.port},\
#  http://${service-provider.server.host}:${service-provider3.port}



### 使用Eureka发现服务提供端
# Eureka
# eureka注册
eureka.client.serviceUrl.defaultZone = http://127.0.0.1:3781/spring-eureka-server/eureka
```
<br />

#### 新建对服务提供端的Controller进行访问的服务

```Java
package com.sestar.springcloudribbonclient.web.controller;

import com.sestar.springcloudribbonclient.domain.User;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.loadbalancer.LoadBalancerClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

import javax.annotation.Resource;
import java.io.IOException;

/**
 * @description 访问服务提供端Controller的服务
 * @author zhangxinxin
 * @date 2019/1/10 14:39
 **/
@RestController
public class RibbonClientController {

    @Autowired
    @Qualifier("loadBalanceRestTemplate")
    private RestTemplate restTemplate;

    @Value("${service-provider.server.host}")
    private String serverHost;

    @Value("${service-provider.server.context-path")
    private String serverContextPath;

    @Value("${service-provider.application.name}")
    private String applicationName;

    @Value("${service-provider1.port}")
    private String port;

    @Value("${service-provider.application.name}" + "${service-provider.server.context-path}")
    private String urlTogether;

    @GetMapping("/greeting")
    public String greeting() {
        User user = new User();
        user.setId(123L);
        user.setName("sestar");

        // 使用拼接方式(非Eureka Client)
//        String url = "http://" + host + ":" + port + contextPath + "/greeting";
        // 使用ribbon.listOfServers方式, 解析url(非Eureka Client)
//        String url = "http://" + urlTogether + "/greeting";
        // 使用 Eureka 服务发现访问 server-provider
        String url = "http://" + urlTogether + "/greeting";

        return restTemplate.postForObject(url, user, String.class);
    }

}
```

<br />

#### 测试能否访问服务提供端Controller

> 访问 http://127.0.0.1:3791/cloud-ribbon-client/greeting  
> 返回字符串 greeting:User{name='sestar', id=123} port: 3771 即为可以正常访问

<br />

### 测试负载均衡

```text

1. 服务提供端创建三个(通过Args修改server.port, 假设启动3771,3772, 3773端口)

2. (使用Ribbon.listOfServers方式请忽略此步)
查看Eureka Server的Instance currently registered with eureka是否有三个application.name为
cloud-server-provider-application实例,若存在则说明已经提供了三个服务提供端.

3. 访问 http://127.0.0.1:3791/cloud-ribbon-client/index, 访问结果查看port端口

3.1 使用默认负载均衡方式, 轮训使用端口
不停地访问, 会发现一个规律, port后面的端口其实是按一定规律, 三个端口一次循环.
说明了三个端口的访问数量基本被控制相同, 不会有其中一个端口相比其他端口频繁访问, 有效地利用了所有的端口

3.2 使用自定义的负载均衡, 永远返回可达服务器的最后一台
不停地访问, 会发现port后面的端口都是最先注册的服务提供端(ServerList的存储结构是列表(后进先出), 
最后注册的就是第一位服务提供端), 可以尝试将服务提供端都重启, 再次查看是否是最先注册的服务提供端的端口
(最好把Eureka Server的application.properties关闭Eureka自我保护设置, 会把无法访问的
Eureka Client进行剔除)
```

<br />

## 服务熔断器(Netflix Hystrix)

[参考源码](https://github.com/Sestar/firstLearn/tree/master/spring-cloud/cloud-multiple)

<br />

### Netflix Hystrix 简介

[原理解析](https://blog.csdn.net/onlyou930/article/details/77368962)

<br />

#### Netflix Hystrix使用原因

```text
当有一个服务请求时间过长, 导致其他服务无法调用底层, 会极大影响其他服务, 造成"雪崩"现象,
所以需要一个管理控件进行统一管理服务。

而 Netflix Hystrix 可以帮助隔离每个服务, 能使单个服务访问失败, 不会影响整个系统服务,
这种机制也称作对某个服务进行熔断, 这种熔断机制, 极大提升整个服务的可靠性, 可用性。

在大中型分布式系统中，通常系统很多依赖(HTTP,hession,Netty,Dubbo等)，在高并发访问下,这些依赖的稳定性与否对系统的影响非常大,但是依赖有很多不可控问题:如网络连接缓慢，资源繁忙，暂时不可用，服务脱机等。

当依赖阻塞时,大多数服务器的线程池就出现阻塞(BLOCK),影响整个线上服务的稳定性，在复杂的分布式架构的应用程序有很多的依赖，都会不可避免地在某些时候失败。高并发的依赖失败时如果没有隔离措施，当前应用服务就有被拖垮的风险。

例如:一个依赖30个SOA服务的系统,每个服务99.99%可用。
99.99%的30次方 ≈ 99.7%
0.3% 意味着一亿次请求 会有 3,000,00次失败
换算成时间大约每月有2个小时服务不稳定.
随着服务依赖数量的变多，服务不稳定的概率会成指数性提高.
```

<br />

#### 设计理念

<img width="30%" src="/note/_v_images/java/框架/SpringCloud/Hystrix-designIdea-1.png">

```text
Command是在Receiver和Invoker之间添加的中间层，Command实现了对Receiver的封装。
那么Hystrix的应用场景如何与上图对应呢？

API既可以是Invoker又可以是reciever，通过继承Hystrix核心类HystrixCommand来封装这些API
（例如，远程接口调用，数据库查询之类可能会产生延时的操作）。就可以为API提供弹性保护了。
```

<br />

#### Hystrix流程结构解析

<img width="30%" src="/note/_v_images/java/框架/SpringCloud/Hystrix-process-1.png">

```text
流程说明:

1:每次调用创建一个新的HystrixCommand,把依赖调用封装在run()方法中.
2:执行execute()/queue做同步或异步调用.
3:判断熔断器(circuit-breaker)是否打开,如果打开跳到步骤8,进行降级策略,如果关闭进入步骤.
4:判断线程池/队列/信号量是否跑满，如果跑满进入降级步骤8,否则继续后续步骤.
5:调用HystrixCommand的run方法.运行依赖逻辑
5a:依赖逻辑调用超时,进入步骤8.
6:判断逻辑是否调用成功
6a:返回成功调用结果
6b:调用出错，进入步骤8.
7:计算熔断器状态,所有的运行状态(成功, 失败, 拒绝, 超时)上报给熔断器，用于统计从而判断熔断器状态.
8:getFallback()降级逻辑.
  以下四种情况将触发getFallback调用：
  (1):run()方法抛出非HystrixBadRequestException异常。
  (2):run()方法调用超时
  (3):熔断器开启拦截调用
  (4):线程池/队列/信号量是否跑满
8a:没有实现getFallback的Command将直接抛出异常
8b:fallback降级逻辑调用成功直接返回
8c:降级逻辑调用失败抛出异常
9:返回执行成功结果
```

<br />

#### CircuitBreaker流程结构解析

```text
每个熔断器默认维护10个bucket,每秒一个bucket,每个bucket记录成功,失败,超时,拒绝的状态，
默认错误超过50%且10秒内超过20个请求进行中断拦截.
```

<img width="30%" src="/note/_v_images/java/框架/SpringCloud/Hystrix-circuitBreaker-process-1.png">

<br />

#### HystrixCommand注解

```text
在@HystrixCommand注解中添加Command配置,
查看官网 https://github.com/Netflix/Hystrix/wiki/Configuration
```

#### Netflix Hystrix使用方式

```text
  1. 注解方式
  1.1 在启动类上添加@EnableHystrix
  1.2 在需要熔断机制的服务上添加@HystrixCommand(), 在里面填写具体的command配置
      (例如源码中设置了超时时间和回滚方法, 回滚方法的入参和出参必须和服务方法一致)

  2. 配置类
  2.1 在启动类上添加@EnabelCircuitBreaker(EnableHystrix是EnableCircuitBreaker的封装注释, 
      作用一样, 但是源码如果不加上这个注释也是能够进行超时熔断, 具体原因不知)
  2.2 新增继承HystrixCommand<?>的Command类(源码Command类是RibbonClientHystrixCommand)
  2.2.1 HystrixCommand<?>的泛型是表示Servlet返回类型
  2.2.2 继承HystrixCommand必须需要重写run方法, HystrixCommand#run是虚拟方法
  2.2.3 command配置可以在HystrixCommand(源码是RibbonClientHystrixCommand)的构造器中写入
  2.2.4 可以重写HystrixCommand的其他方法, 比如getFallback方法(回滚方法)
  2.2.5 servlet使用HystrixCommand类(源码是RibbonClientHystrixCommand)访问服务端, 每次访问
      必须创建一个新的HystrixCommand, 否则IllegalStateException, 提示
      “This instance can only be executed once. Please instantiate a new instance.” 错误,
      所以源码中把HystrixCommand类作为servlet的变量属性的做法是不可取的。

Netflix Hystrix 两个使用方式区别：
  注解方式明显比配置类的使用方便, 只需要在servlet的Mapping上添加HystrixCommand配置即可, 而且回滚方法
  能够获取服务的入参, 但是配置类的方式则不行。
```

<br />

### ReadMeFirst

```text
源码是spring boot多个功能的集合, 本节主要介绍服务熔断器(Netfilx Hystrix)

Module:
    user-service-provider(服务提供端), user-service-client(客户端),
    hystrix-dashboard(Hystrix监控).

user-service-provider(服务提供端):
    1. application.properties关闭Eureka
    2. 启动类激活@EnableHystrix

    使用Hystrix的方式是在服务添加HystrixCommand, 增加超时熔断, 自定义回滚方法

user-service-client(客户端):
    1. application.properties启用Ribbon LoadBalancerClient 提供服务列表并关闭Eureka
    2. 启动类激活@EnabelCircuitBreaker(由于使用HystrixCommand派生类, 不激活也能使用)
    3. bootstrap配置不要调用(改名即可)
    4. 使用 Service-provider 的相关内容

    使用Hystrix的方式是利用RibbonClientHystrixCommand(HystrixCommand的派生类)#run

hystrix-dashboard(Hystrix监控):
    启动类激活@EnableHystrixDashboard
```

<br />

### 搭建 Eureka Server

<br />

#### Maven依赖

```xml
<!-- Eureka Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
```

<br />

#### application.properties

```properties
# 应用名称
spring.application.name = eureka-server-application

# 应用服务
server.port = 3701

## Eureka
### 取消向注册中心注册
eureka.client.register-with-eureka = false
### 取消向注册中心获取注册信息(服务, 实例信息)
eureka.client.fetch-registry = false
## 将应用注册到 Eureka 服务器, 解决 peer 问题
### 当*.hostname 和 *.serviceUrl.defaultZone的 protocol一致时, Eureka服务端将不会再去寻找其他Eureka服务端
eureka.instance.hostname = 127.0.0.1
eureka.client.serviceUrl.defaultZone = http://${eureka.instance.hostname}:${server.port}/eureka
```

<br />

#### 启动类注册为 Eureka Server

```java
package com.sestar.eurekaserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer // 注册 Eureka Server
public class EurekaServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaServerApplication.class, args);
	}

}
```

<br />

### 搭建服务提供端

<br />

#### Maven依赖

```xml
<!-- Netflix Hystrix -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-hystrix</artifactId>
</dependency>

<!-- Actuator 插件 -->
<!-- 访问 /hystrix.stream 需要Actuator插件 -->
<!-- 源码中Actuator是由父模块继承, 所以没有添加 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

<br />

#### 启动类注册熔断器

```Java
package com.sestar.cloudserverprovider;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.hystrix.EnableHystrix;

@SpringBootApplication
@EnableDiscoveryClient // 使用Eureka服务发现方式发现服务提供端
@EnableHystrix // Netflix Hystrix 熔断器
public class CloudServerProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(CloudServerProviderApplication.class, args);
    }

}
```

<br />

#### 服务添加Hystrix Command配置

```Java
package com.sestar.cloudserverprovider.web.controller;

import com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand;
import com.netflix.hystrix.contrib.javanica.annotation.HystrixProperty;
import com.sestar.cloudserverprovider.domain.User;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

import java.util.Random;

/**
 * @description Server Provider Controller
 * @author zhangxinxin
 * @date 2019/1/10 14:30
 **/
@RestController
public class ServerProviderController {

    @Value("${server.port}")
    private String port;

    /**
     * 随机数工具类
     **/
    private static final Random random = new Random();

    /**
     * @description 测试Hystrix的超时
     * @author zhangxinxin
     * @date 2019/2/12 17:11
     * @return java.lang.String
     **/
    @HystrixCommand(
        commandProperties = {   // command 配置
            @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "100") // 设置最大执行时间
        },
        fallbackMethod = "cuttingOutFallBack" // 设置熔断后执行回滚方法
    )
    @PostMapping("/operaHystrix")
    public String operaHystrix() throws InterruptedException {
        int randomNbr = random.nextInt(200);
        System.out.println("Execution Time: " + randomNbr);
        Thread.sleep(randomNbr);
        return "randomNbr:" + randomNbr + ", port:" + port;
    }

    /**
     * @description operaHystrix熔断后执行回滚方法
     * @author zhangxinxin
     * @date 2019/2/12 14:57
     * @return java.lang.String
     **/
    private String cuttingOutFallBack() {
        return "operaHystrix Method Execution is cutting out!";
    }

}
```

<br />

### 搭建客户端

<br />

#### Maven依赖

```xml
<!-- Netflix Hystrix -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-hystrix</artifactId>
</dependency>

<!-- Actuator 插件 -->
<!-- 访问 /hystrix.stream 需要Actuator插件 -->
<!-- 源码中Actuator是由父模块继承, 所以没有添加 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

<br />

#### 启动类注册熔断器

```Java
package com.sestar.springcloudribbonclient;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.cloud.netflix.ribbon.RibbonClient;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

@SpringBootApplication
@RibbonClient(name = "user-server-provider-application")
@EnableDiscoveryClient // 使用Eureka服务发现方式发现服务提供端
@EnableCircuitBreaker  // Netflix Hystrix 熔断器
public class SpringCloudRibbonClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudRibbonClientApplication.class, args);
    }

    /**
     * @description 获取用以负载均衡的RestTemplate的Bean对象,
     *          注释@LoadBalanced将会把RestTemplate也注册为 LoadBalancerClient,
     *          所以如果自动注入LoadBalancerClient对象也是本方法返回对象
     *          而且使用该RestTemplate.execute调用的也是LoadBalancerClient#execute
     * @author zhangxinxin
     * @date 2019/1/10 14:43
     * @return org.springframework.web.client.RestTemplate
     **/
    @Bean(name = "loadBalanceRestTemplate")
    @LoadBalanced
    public RestTemplate getLoadBalanceRestTemplate() {
        return new RestTemplate();
    }

    // ... 省略其他与本节知识点无关内容

}
```

<br />

#### 自定义HystrixCommand派生类

```Java
package com.sestar.springcloudribbonclient.hystrix;

import com.netflix.hystrix.HystrixCommand;
import com.netflix.hystrix.HystrixCommandGroupKey;
import org.springframework.web.client.RestTemplate;

/**
 * @description Ribbon Client 的熔断器
 * @author zhangxinxin
 * @date 2019/2/12 22:07
 **/
public class RibbonClientHystrixCommand extends HystrixCommand<Object> {

    /**
     * 服务提供方的服务名称
     **/
    private String applicationName;

    /**
     * 具有负载均衡的RestTemplate
     **/
    private RestTemplate restTemplate;

    public RibbonClientHystrixCommand(String applicationName, RestTemplate restTemplate) {
        // 设置超时时间, name可以随便定义
        super(HystrixCommandGroupKey.Factory.asKey("Ribbon-Client-Hystrix"),
                100);
        this.restTemplate = restTemplate;
        this.applicationName = applicationName;
    }

    @Override
    protected Object run() {
        String url = "http://" + applicationName + "/operaHystrix";
        return restTemplate.postForObject(url, null, String.class);
    }

    @Override
    protected Object getFallback() {
        return "RibbonClientHystrixCommand run Method Execution has be cutted out!";
    }
}
```

<br />

#### HystrixCommand派生类访问服务

```text
从下面源码可以看出, 如果使用HystrixCommand方式, 每个HystrixCommand只能访问一次, 所以servlet每次
访问必须新建一个HystrixCommand实例。
```

```Java
package com.sestar.springcloudribbonclient.web.controller;

import com.sestar.springcloudribbonclient.domain.User;
import com.sestar.springcloudribbonclient.hystrix.RibbonClientHystrixCommand;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

import javax.annotation.PostConstruct;

/**
 * @author zhangxinxin
 * @description 访问服务提供端Controller的服务
 * @date 2019/1/10 14:39
 **/
@RestController
public class RibbonClientController {

    /**
     * 具有负载均衡的RestTemplate
     **/
    private RestTemplate restTemplate;

    /**
     * 服务提供端的服务名称
     **/
    private String applicationName;

    /**
     * HystrixCommand只能执行一次, 所以servlet必须每次执行前先 new HystrixCommand对象, 否则IllegalStateException
     * 所以HystrixCommand不适用于作为属性变量
     **/
    @SuppressWarnings("all")
    private RibbonClientHystrixCommand hystrixCommand;

    /**
     * PostConstruct注释使得该方法在初始化之前执行, 配合上面hystrixCommand变量属性使用, 当使用hystrixCommand执行两次之后会爆
     * “This instance can only be executed once. Please instantiate a new instance.” 错误
     * 所以HystrixCommand不适用于作为属性变量
     **/
    @PostConstruct
    public void init() {
        this.hystrixCommand = new RibbonClientHystrixCommand(applicationName, restTemplate);
    }

    public RibbonClientController(@Autowired @Qualifier("loadBalanceRestTemplate") RestTemplate restTemplate,
                                  @Value("${service-provider.application.name}") String applicationName) {
        this.restTemplate = restTemplate;
        this.applicationName = applicationName;
    }

    /**
     * @description 测试Hystrix熔断器(RibbonClient和ServerProvider都有Hystrix熔断器)
     * @author zhangxinxin
     * @date 2019/2/12 18:01
     * @return java.lang.String
     **/
    @GetMapping("/timeoutHystrixOfProvider")
    public String timeoutHystrixOfProvider() {
        /*
         * 该方法执行两次之后会爆
         *“This instance can only be executed once. Please instantiate a new instance.” 错误
         * 是由于HystrixCommand一个实例只能执行一次, 所以每次执行需要一个新的HystrixCommand
         **/
//        return hystrixCommand.execute().toString();
        return new RibbonClientHystrixCommand(applicationName, restTemplate).execute().toString();
    }

}
```

<br />

### 搭建Netflix Hystrix监控模块

<br />

#### Maven依赖

```xml
<!-- Hystrix Dashboard -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-hystrix-dashboard</artifactId>
</dependency>

<!-- Actuator 插件 -->
<!-- 访问 /hystrix.stream 需要Actuator插件 -->
<!-- 源码中Actuator是由父模块继承, 所以没有添加, 监控模块必须得有Actuator -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

<br />

#### 启动类注册为Netflix Hystrix监控平台

```Java
package com.sestar.hystrixdashboard;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.hystrix.dashboard.EnableHystrixDashboard;

@SpringBootApplication
@EnableHystrixDashboard // 注册Netflix Hystrix监控
public class HystrixDashboardApplication {

    public static void main(String[] args) {
        SpringApplication.run(HystrixDashboardApplication.class, args);
    }

}
```

<br />

### 测试熔断部署

<br />

#### 测试服务提供端的熔断部署

```text
访问 http://localhost:3721/timeoutHystrixOfProvider
查看服务提供端的Console, 如果输出的Execution Time大于100, 则应该会被服务提供端的熔断器熔断,
由于服务提供端源码(..cloudserverprovider.web.controller.ServerProviderController#operaHystrix)
有设置回滚方法, 如果断点在回滚方法有执行, 这就代表服务提供端的熔断部署成功。

休眠时间在100毫秒附近, 有可能发生错误:
  第一次执行会有类加载问题, 加载也需要时间, 即使休眠90毫秒,如果加载类时间过长也会让执行时间超过100毫秒。
  超过100毫秒几毫秒, 几毫秒也可能程序反应不来, 还没进行熔断就通过了(我猜的..反正几毫秒的风险还是可以承担的)
  所以多测试几次, 看执行时间不在100毫秒附近
```

<br />

#### 测试客户端的熔断部署

```text
访问 http://localhost:3721/timeoutHystrixOfProvider
查看服务提供端的Console, 如果输出的Execution Time大于100, 会被服务提供端熔断, 也会被客户端熔断
由于客户端源码(..springcloudribbonclient.hystrix.RibbonClientHystrixCommand)有设置回滚方法,
如果断点在回滚方法有执行, 这就代表客户端的熔断部署成功。
```

<br />

### 查看Netflix Hystrix监控数据

<br />

#### Hystrix Stream数据

```text
Netflix Hystrix获取监控数据有两种方式:
    1. 熔断器自带获取监控数据 /hystrix.stream (必须有actuator插件)
       访问 localhost:3741/hystrix.stream 获取json数据(下方展示),
       json数组不够直观

    2. 利用Netflix Hystrix Dashboard监控模块
       访问 http://localhost:3751/hystrix,
       在Hystrix DashBoard下方url填写localhost:3771/hystrix.stream
       监控模块将会把json转化为图形数据
```

<br />

#### /hystrix.stream json数据样例

```json
data: {
	"type": "HystrixCommand",
	"name": "RibbonClientHystrixCommand",  // 自定义的HystrixCommand派生类
	"group": "Ribbon-Client-Hystrix",      // 自定义的HystrixCommand的GroupKey
	"currentTime": 1550025781920,
	"isCircuitBreakerOpen": false,
	"errorPercentage": 0,
	"errorCount": 0,
	"requestCount": 0,
	"rollingCountBadRequests": 0,
	"rollingCountCollapsedRequests": 0,
	"rollingCountEmit": 0,
	"rollingCountExceptionsThrown": 0,
	"rollingCountFailure": 0,
	"rollingCountFallbackEmit": 0,
	"rollingCountFallbackFailure": 0,
	"rollingCountFallbackMissing": 0,
	"rollingCountFallbackRejection": 0,
	"rollingCountFallbackSuccess": 0,
	"rollingCountResponsesFromCache": 0,
	"rollingCountSemaphoreRejected": 0,
	"rollingCountShortCircuited": 0,
	"rollingCountSuccess": 0,
	"rollingCountThreadPoolRejected": 0,
	"rollingCountTimeout": 0,
	"currentConcurrentExecutionCount": 0,
	"rollingMaxConcurrentExecutionCount": 0,
	"latencyExecute_mean": 0,
	"latencyExecute": {
		"0": 0,
		"25": 0,
		"50": 0,
		"75": 0,
		"90": 0,
		"95": 0,
		"99": 0,
		"99.5": 0,
		"100": 0
	},
	"latencyTotal_mean": 0,
	"latencyTotal": {
		"0": 0,
		"25": 0,
		"50": 0,
		"75": 0,
		"90": 0,
		"95": 0,
		"99": 0,
		"99.5": 0,
		"100": 0
	},
	"propertyValue_circuitBreakerRequestVolumeThreshold": 20,
	"propertyValue_circuitBreakerSleepWindowInMilliseconds": 5000,
	"propertyValue_circuitBreakerErrorThresholdPercentage": 50,
	"propertyValue_circuitBreakerForceOpen": false,
	"propertyValue_circuitBreakerForceClosed": false,
	"propertyValue_circuitBreakerEnabled": true,
	"propertyValue_executionIsolationStrategy": "THREAD",
	"propertyValue_executionIsolationThreadTimeoutInMilliseconds": 100,
	"propertyValue_executionTimeoutInMilliseconds": 100,
	"propertyValue_executionIsolationThreadInterruptOnTimeout": true,
	"propertyValue_executionIsolationThreadPoolKeyOverride": null,
	"propertyValue_executionIsolationSemaphoreMaxConcurrentRequests": 10,
	"propertyValue_fallbackIsolationSemaphoreMaxConcurrentRequests": 10,
	"propertyValue_metricsRollingStatisticalWindowInMilliseconds": 10000,
	"propertyValue_requestCacheEnabled": true,
	"propertyValue_requestLogEnabled": true,
	"reportingHosts": 1,
	"threadPool": "Ribbon-Client-Hystrix"
}

data: {
	"type": "HystrixThreadPool",
	"name": "Ribbon-Client-Hystrix",
	"currentTime": 1550025781920,
	"currentActiveCount": 0,
	"currentCompletedTaskCount": 7,
	"currentCorePoolSize": 10,
	"currentLargestPoolSize": 7,
	"currentMaximumPoolSize": 10,
	"currentPoolSize": 7,
	"currentQueueSize": 0,
	"currentTaskCount": 7,
	"rollingCountThreadsExecuted": 0,
	"rollingMaxActiveThreads": 0,
	"rollingCountCommandRejections": 0,
	"propertyValue_queueSizeRejectionThreshold": 5,
	"propertyValue_metricsRollingStatisticalWindowInMilliseconds": 10000,
	"reportingHosts": 1
}
```

<br />

#### Hystrix DashBoard图例化

<img width="100%" src="/note/_v_images/java/框架/SpringCloud/HystrixDoard-1.png">

<img width="100%" src="/note/_v_images/java/框架/SpringCloud/HystrixDoard-2.png">

<br />

## Feign(熔断器 + 负载均衡)

[参考源码](https://github.com/Sestar/firstLearn/tree/master/spring-cloud/cloud-multiple)

<br />

### Feign 简介

```text
Feign是 Netflix 和 Ribbon 的整合。也就是实现了熔断和负载均衡的功能。
是一个申明式的Web Service, 可以让 Web Service调用更加简单。源码中就是让User Client调用
User api中Feign申明的接口, 在user provider中实现服务, 让服务实现不是直接暴露在User Client
```

<br />

### ReadMeFirst

```text
源码是spring boot多个功能的集合, 本节主要介绍Feign

Module:
    eureka-server(Eureka服务注册端), config-server(服务配置端),
    user-api(用户服务API), user-service-provider(服务提供端),
    user-service-client(客户端).

user-service-provider(服务提供端):
    application.properties开启Eureka
    启动类激活@EnableHystrix, @EnableDiscoveryClient
    使用Hystrix的方式是在服务添加HystrixCommand, 增加超时熔断, 自定义回滚方法

user-service-client(客户端):
    application.properties启用Ribbon LoadBalancerClient 提供服务列表并关闭Eureka
    启动类激活@RibbonClient, @EnableDiscoveryClient, @EnabelFeignClients
    关闭ribbon.listOfServers, Feign和Service-provider(config-server的profile中获取)
```

<br />

### 结构图

<br />

<img width="30%" src="/note/_v_images/java/框架/SpringCloud/Feign-1.png">

<br />

### 搭建 Eureka Server

<br />

#### Maven依赖

```xml
<!-- Eureka Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
```

<br />

#### application.properties

```properties
# 应用名称
spring.application.name = eureka-server-application

# 应用服务
server.port = 3701

## Eureka
### 取消向注册中心注册
eureka.client.register-with-eureka = false
### 取消向注册中心获取注册信息(服务, 实例信息)
eureka.client.fetch-registry = false
## 将应用注册到 Eureka 服务器, 解决 peer 问题
### 当*.hostname 和 *.serviceUrl.defaultZone的 protocol一致时, Eureka服务端将不会再去寻找其他Eureka服务端
eureka.instance.hostname = 127.0.0.1
eureka.client.serviceUrl.defaultZone = http://${eureka.instance.hostname}:${server.port}/eureka
```

<br />

#### 启动类注册为 Eureka Server

```java
package com.sestar.eurekaserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer // 注册 Eureka Server
public class EurekaServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaServerApplication.class, args);
	}

}
```

<br />

### 搭建服务配置端

<br />

#### Maven依赖

```xml
<!-- Spring Cloud Config Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>

<!-- Eureka Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```

<br />

#### application.properties

```properties
## Eureka
### Eureka 注册
eureka.client.serviceUrl.defaultZone = http://127.0.0.1:3701/eureka
### Eureka 使用 IP 注册, 不使用IP注册user-service-client无法找到config-server, 具体原因不知道
eureka.instance.prefer-ip-address = true

## 配置服务器文件系统git仓库, 配置文件git仓库最好是git单独的repository
spring.cloud.config.server.git.uri = https://github.com/Sestar/cloud-config.git
## 强制拉取git内容, 如果本地副本是脏数据, 将强制从远程仓库拉取配置
spring.cloud.config.server.git.force-pull = true
```

<br />

#### 启动类注册Eureka Client, Config Server

```java
package com.sestar.configserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.config.server.EnableConfigServer;

/**
 * @description Config Server
 * @author zhangxinxin
 * @date 2019/2/18 14:15
 **/
@SpringBootApplication
@EnableDiscoveryClient  // 能够Eureka发现
@EnableConfigServer     // Cloud Server
public class ConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }

}
```

<br />

### 搭建用户服务API模块

<br />

#### Maven依赖

```xml
<!-- 添加 Spring Cloud Feign 依赖 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-feign</artifactId>
</dependency>
```

<br />

#### 编写一个服务API 实现 Feign

<br />

##### Feign 服务接口

```Java
package com.sestar.userapi.api;

import com.sestar.userapi.domain.User;
import com.sestar.userapi.hystrix.UserServerFallBack;
import org.springframework.cloud.netflix.feign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;

import java.util.List;

/**
 * @description 用户服务API
 * @author zhangxinxin
 * @date 2019/2/15 18:00
 */
@FeignClient(name = "${user.server.name}", fallback = UserServerFallBack.class)
// user.server.name通过config-server获取配置文件中得到
public interface IUserService {

    /**
     * @description 保存用户
     * @author zhangxinxin
     * @date 2019/2/15 17:59
     * @param user 用户信息
     * @return boolean
     */
    @PostMapping("/user/save")
    boolean saveUser(User user);

    /**
     * @description 查询所有的用户列表
     * @author zhangxinxin
     * @date 2019/2/15 17:59
     * @return java.util.List<com.sestar.userapi.domain.User>
     */
    @GetMapping("/user/find/all")
    List<User> findAll();

}
```

<br />

##### Feign 回滚(暂未实现)

```Java
package com.sestar.userapi.hystrix;

import com.sestar.userapi.api.IUserService;
import com.sestar.userapi.domain.User;

import java.util.Collections;
import java.util.List;

/**
 * @description IUserService FallBack
 * @author zhangxinxin
 * @date 2019/2/18 9:55
 **/
public class UserServerFallBack implements IUserService {

    @Override
    public boolean saveUser(User user) {
        return false;
    }

    @Override
    public List<User> findAll() {
        return Collections.emptyList();
    }
}
```

<br />

### 搭建服务提供端

<br />

#### Maven依赖

```xml
<!-- 依赖 用户 API -->
<dependency>
    <groupId>com.sestar</groupId>
    <artifactId>user-api</artifactId>
    <version>${project.version}</version>
</dependency>

<!-- Eureka Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>

<!-- Netflix Hystrix -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-hystrix</artifactId>
</dependency>
```

<br />

#### application.properties

```properties
## Eureka
### Eureka注册
eureka.client.serviceUrl.defaultZone = http://127.0.0.1:3701/eureka
### 上传注册信息时间间隔
eureka.client.instance-info-replication-interval-seconds = 5
### 获取注册信息时间间隔
eureka.client.registry-fetch-interval-seconds = 5
### Eureka注册名使用IP
eureka.instance.prefer-ip-address = true
```

<br />

#### 启动类注册Eureka Client, 实现Hystrix

```Java
package com.sestar.userserverprovider;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.hystrix.EnableHystrix;

@SpringBootApplication
@EnableDiscoveryClient // 使用Eureka服务发现方式发现服务提供端
@EnableHystrix // Netflix Hystrix 熔断器
public class UserServerProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserServerProviderApplication.class, args);
    }

}
```

<br />

#### Controller实现用户服务API的接口

```Java
package com.sestar.userserverprovider.web.controller;

import com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand;
import com.netflix.hystrix.contrib.javanica.annotation.HystrixProperty;
import com.sestar.userapi.api.IUserService;
import com.sestar.userapi.domain.User;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

import java.util.Collections;
import java.util.List;
import java.util.Random;

/**
 * @description Server Provider Controller
 * @author zhangxinxin
 * @date 2019/1/10 14:30
 **/
@RestController
public class UserServiceProviderController implements IUserService {

    /**
     * 用户服务中心
     **/
    private final IUserService userServer;

    /**
     * 随机数工具类
     **/
    private static final Random random = new Random();

    @Value("${server.port}")
    private String port;

    @Autowired
    public UserServiceProviderController(@Qualifier("inMemoryUserService") IUserService userServer) {
        this.userServer = userServer;
    }

    // 通过方法继承，URL 映射 ："/user/save"
    @Override
    public boolean saveUser(@RequestBody User user) {
        return userServer.saveUser(user);
    }

    // 通过方法继承，URL 映射 ："/user/find/all"
    @HystrixCommand(
            commandProperties = {   // command 配置
                    @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "100") // 设置最大执行时间
            },
            fallbackMethod = "fallbackForGetUsers" // 设置熔断后执行回滚方法
    )
    @Override
    public List<User> findAll() {
        return userServer.findAll();
    }

    // ...省略其他不相关内容
}
```

<br />

#### ServiceImpl 实现用户服务API接口

```Java
package com.sestar.userserverprovider.web.service;

import com.sestar.userapi.api.IUserService;
import com.sestar.userapi.domain.User;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * @description 用户服务
 * @author zhangxinxin
 * @date 2019/2/15 20:42
 **/
@Service("inMemoryUserService")
public class InMemoryUserService implements IUserService {

    private Map<Long, User> repository = new ConcurrentHashMap<>();

    @Override
    public boolean saveUser(User user) {
        return repository.put(user.getId(), user) == null;
    }

    @Override
    public List<User> findAll() {
        return new ArrayList(repository.values());
    }

}
```

<br />

### 搭建客户端

<br />

#### Maven依赖

```xml
<!-- 依赖 用户 API -->
<dependency>
    <groupId>com.sestar</groupId>
    <artifactId>user-api</artifactId>
    <version>${project.version}</version>
</dependency>

<!-- Eureka Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>

<!-- Ribbon -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-ribbon</artifactId>
</dependency>

<!-- Netflix Hystrix -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-hystrix</artifactId>
</dependency>

<!-- Config Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-client</artifactId>
</dependency>
```

<br />

#### application.properties

```properties
## Ribbon
### 配置名称可以看 org.springframework.cloud.netflix.ribbon.PropertiesFactory
user-service-provider-application.ribbon.NFLoadBalancerRuleClassName = com.sestar.userserviceclient.loadbalanced.MyRule
user-service-provider-application.ribbon.NFLoadBalancerPingClassName = com.sestar.userserviceclient.loadbalanced.MyPing
```

<br />

#### bootstrap.properties

```properties
## 应用名称
spring.application.name = user-service-client-application

##  Eureka
### spring-eureka-server-application注册到 Eureka 服务器中
### ${}都为服务器的配置内容 http://服务器地址:${server.port}/${server.context-path}/eureka
### 如果 bootstrap 有配置 spring.cloud.config.discovery.service-id, 则需要把该配置移到 bootstrap 中
### 因为*.discovery.service-id属性是查找 Eureka 服务配置端, 而查找范围就是eureka.client.service-url.defaultZone
### 需要最先加载 Eureka 注册信息, eureka.client.service-url.defaultZone才能确定范围,
### 而 *.discovery.service-id属性配置在 bootstrap 中, 比 application 配置先加载,
### 所以当有配置Eureka服务配置端, 则需要把 eureka.client.service-url.defaultZone必须配置在 bootstrap 中
eureka.client.serviceUrl.defaultZone = http://127.0.0.1:3701/eureka

## 配置客户端应用关联的应用
## spring.cloud.config.name 是可选的
## 如果没有配置，采用 ${spring.application.name}
spring.cloud.config.name = user-server
## 关联profile
spring.cloud.config.profile = default
## 关联label
spring.cloud.config.label = master
## 激活 Config Server 服务发现和 spring.cloud.config.discovery.serviceId 捆绑配置
spring.cloud.config.discovery.enabled = true
## Config Server 服务器应用名称
spring.cloud.config.discovery.serviceId = config-server-application
```

<br />

#### 启动类注册RibbonClient, Eureka Client; 开启Hystrix, FeignClient

```Java
package com.sestar.userserviceclient;

import com.sestar.userapi.api.IUserService;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.cloud.netflix.feign.EnableFeignClients;
import org.springframework.cloud.netflix.ribbon.RibbonClient;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

@SpringBootApplication
@RibbonClient(name = "user-service-provider-application")
@EnableDiscoveryClient // 使用Eureka服务发现方式发现服务提供端
@EnableCircuitBreaker  // Netflix Hystrix 熔断器
@EnableFeignClients(clients = IUserService.class) // 申明 UserService 接口作为 Feign Client 调用
public class UserServiceClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserServiceClientApplication.class, args);
    }

    /**
     * @description 获取用以负载均衡的RestTemplate的Bean对象,
     *          注释@LoadBalanced将会把RestTemplate也注册为 LoadBalancerClient,
     *          所以如果自动注入LoadBalancerClient对象也是本方法返回对象
     *          而且使用该RestTemplate.execute调用的也是LoadBalancerClient#execute
     * @author zhangxinxin
     * @date 2019/1/10 14:43
     * @return org.springframework.web.client.RestTemplate
     **/
    @Bean(name = "loadBalanceRestTemplate")
    @LoadBalanced
    public RestTemplate getLoadBalanceRestTemplate() {
        return new RestTemplate();
    }

    // ...省略其他不相关内容
}
```

<br />

#### Controller调用用户服务API的接口

```Java
package com.sestar.userserviceclient.web.controller;

import com.sestar.userapi.api.IUserService;
import com.sestar.userapi.domain.User;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

/**
 * @description Server Client Controller
 * @author zhangxinxin
 * @date 2019/2/15 21:30
 **/
@RestController
public class UserServiceClientController {

    /**
     * 用户服务中心
     **/
    @SuppressWarnings("all")
    @Autowired
    private IUserService userServer;

    @PostMapping("/user/save")
    public boolean saveUser(@RequestBody User user) {
        return userServer.saveUser(user);
    }

    @GetMapping("/user/find/all")
    public List<User> findAll() {
        return userServer.findAll();
    }

}
```

### 测试Feign熔断和负载

<br />

#### 熔断(暂时只有Hystrix, Feign.fallback暂未实现)

```text
访问 http://localhost:3721/timeoutHystrix
如果user-service-provider-application的console输出Execution Time超过100, Hystrix将会进行超时熔断, 
页面输出 “operaHystrix Method Execution is cutting out!”
```

<br />

#### 负载均衡

```text
1. user-service-provider-application启动多个端口: 3741, 3751, 3761

2. 用postman POST访问 http://localhost:3761/user/save, Body传User的json: {"id": 1, "name": "sestar"}
对三个不同端口存储不同User对象;

3. GET 访问 http://localhost:3721/user/find/all, 每次返回User对象不一样(端口不同, 在默认负载均衡IRule下)
```

<br />

## 路由(Zuul)

[参考源码](https://github.com/Sestar/firstLearn/tree/master/spring-cloud/cloud-multiple)

<br />

### 简介

```text
激活Zuul有两种方式:
    1. @EnableZuulServer
       导入ZuulServerMarkerConfiguration, 随后生成一个ZuulServerMarkerConfiguration.Marker的Bean对象,
       作为ZuulServerAutoConfiguration的装配前置条件

       Web请求时会交给 DispatcherServlet 处理, 再获取 DispatcherServlet.handlerMappings 匹配映射地址进行访问
       handlerMapperings 是 ApplicationContext 匹配 HandlerMapper.class(DispatcherServlet#initHandlerMappings),
       在此之前其实ZuulServerAutoConfiguration#zuulHandlerMapping 已经生成 ZuulHandlerMapping(基类是HandlerMapper)
       ZuulController 将请求委派给 ZuulServlet, 所以所有的 ZuulFilter 实例都会被执行

       处理顺序：DispatcherServlet -> ZuulHandlerMapping -> ZuulController -> ZuulServlet -> ZuulFilter

    2. @EnableZuulProxy
       导入ZuulProxyMarkerConfiguration, 随后生成一个ZuulProxyMarkerConfiguration.Marker的Bean对象,
       作为ZuulProxyAutoConfiguration的装配前置条件

       处理顺序：DispatcherServlet -> ZuulHandlerMapping -> ZuulController -> ZuulServlet -> RibbonRoutingFilter

两种激活方式的区别其实就是 ZuulProxyAutoConfiguration 和 ZuulServerAutoConfiguration 的区别:
    ZuulServerAutoConfiguration 最后是 ZuulFilter 过滤处理
    ZuulProxyAutoConfiguration 最后是 RibbonRoutingFilter(基类ZuulFilter) 过滤处理
    ZuulProxyAutoConfiguration 指定了 ZuulFilter 的类型: RibbonRoutingFilter
```

<br />

### ReadMeFirst

```text
源码是spring boot多个功能的集合, 本节主要介绍Zuul

Module:
    eureka-server(Eureka服务注册端), config-server(服务配置端),
    user-api(用户服务API), user-service-provider(服务提供端),
    user-service-client(客户端), zuul-proxy(路由管理端).

user-service-provider(服务提供端):
    application.properties开启Eureka
    启动类激活@EnableHystrix, @EnableDiscoveryClient
    使用Hystrix的方式是在服务添加HystrixCommand, 增加超时熔断, 自定义回滚方法

user-service-client(客户端):
    application.properties开启Eureka
    启动类激活@RibbonClient, @EnableDiscoveryClient, @EnabelFeignClients
    关闭ribbon.listOfServers, Feign和Service-provider(config-server的profile中获取)

zuul-proxy(路由管理端):
    application.properties开启Eureka
    启动类激活@EnableZuulProxy
    关闭ribbon.listOfServers, zuul.routes(config-server的profile中获取)
```

<br />

### 结构图

<img width="30%" src="/note/_v_images/java/框架/SpringCloud/Zuul-1.png">

<br />

### 搭建 Eureka Server

<br />

#### Maven依赖

```xml
<!-- Eureka Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
```

<br />

#### application.properties

```properties
# 应用名称
spring.application.name = eureka-server-application

# 应用服务
server.port = 3701

## Eureka
### 取消向注册中心注册
eureka.client.register-with-eureka = false
### 取消向注册中心获取注册信息(服务, 实例信息)
eureka.client.fetch-registry = false
## 将应用注册到 Eureka 服务器, 解决 peer 问题
### 当*.hostname 和 *.serviceUrl.defaultZone的 protocol一致时, Eureka服务端将不会再去寻找其他Eureka服务端
eureka.instance.hostname = 127.0.0.1
eureka.client.serviceUrl.defaultZone = http://${eureka.instance.hostname}:${server.port}/eureka
```

<br />

#### 启动类注册为 Eureka Server

```java
package com.sestar.eurekaserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer // 注册 Eureka Server
public class EurekaServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaServerApplication.class, args);
	}

}
```

<br />

### 搭建服务配置端

<br />

#### Maven依赖

```xml
<!-- Spring Cloud Config Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>

<!-- Eureka Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```

<br />

#### application.properties

```properties
## Eureka
### Eureka 注册
eureka.client.serviceUrl.defaultZone = http://127.0.0.1:3701/eureka
### Eureka 使用 IP 注册, 不使用IP注册user-service-client无法找到config-server, 具体原因不知道
eureka.instance.prefer-ip-address = true

## 配置服务器文件系统git仓库, 配置文件git仓库最好是git单独的repository
spring.cloud.config.server.git.uri = https://github.com/Sestar/cloud-config.git
## 强制拉取git内容, 如果本地副本是脏数据, 将强制从远程仓库拉取配置
spring.cloud.config.server.git.force-pull = true
```

<br />

#### 启动类注册Eureka Client, Config Server

```java
package com.sestar.configserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.config.server.EnableConfigServer;

/**
 * @description Config Server
 * @author zhangxinxin
 * @date 2019/2/18 14:15
 **/
@SpringBootApplication
@EnableDiscoveryClient  // 能够Eureka发现
@EnableConfigServer     // Cloud Server
public class ConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }

}
```

<br />

### 搭建用户服务API模块

<br />

#### Maven依赖

```xml
<!-- 添加 Spring Cloud Feign 依赖 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-feign</artifactId>
</dependency>
```

<br />

#### 服务接口(Feign功能可以忽略不看)

```Java
package com.sestar.userapi.api;

import com.sestar.userapi.domain.User;
import com.sestar.userapi.hystrix.UserServerFallBack;
import org.springframework.cloud.netflix.feign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;

import java.util.List;

/**
 * @description 用户服务API
 * @author zhangxinxin
 * @date 2019/2/15 18:00
 */
@FeignClient(name = "${user.server.name}", fallback = UserServerFallBack.class)
// user.server.name通过config-server获取配置文件中得到
public interface IUserService {

    /**
     * @description 保存用户
     * @author zhangxinxin
     * @date 2019/2/15 17:59
     * @param user 用户信息
     * @return boolean
     */
    @PostMapping("/user/save")
    boolean saveUser(User user);

    /**
     * @description 查询所有的用户列表
     * @author zhangxinxin
     * @date 2019/2/15 17:59
     * @return java.util.List<com.sestar.userapi.domain.User>
     */
    @GetMapping("/user/find/all")
    List<User> findAll();

}
```

<br />

### 搭建服务提供端

<br />

#### Maven依赖

```xml
<!-- 依赖 用户 API -->
<dependency>
    <groupId>com.sestar</groupId>
    <artifactId>user-api</artifactId>
    <version>${project.version}</version>
</dependency>

<!-- Eureka Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>

<!-- Netflix Hystrix -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-hystrix</artifactId>
</dependency>
```

<br />

#### application.properties

```properties
## Eureka
### Eureka注册
eureka.client.serviceUrl.defaultZone = http://127.0.0.1:3701/eureka
### 上传注册信息时间间隔
eureka.client.instance-info-replication-interval-seconds = 5
### 获取注册信息时间间隔
eureka.client.registry-fetch-interval-seconds = 5
### Eureka注册名使用IP
eureka.instance.prefer-ip-address = true
```

<br />

#### 启动类注册Eureka Client, 实现Hystrix

```Java
package com.sestar.userserverprovider;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.hystrix.EnableHystrix;

@SpringBootApplication
@EnableDiscoveryClient // 使用Eureka服务发现方式发现服务提供端
@EnableHystrix // Netflix Hystrix 熔断器
public class UserServerProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserServerProviderApplication.class, args);
    }

}
```

<br />

#### Controller实现用户服务API的接口

```Java
package com.sestar.userserverprovider.web.controller;

import com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand;
import com.netflix.hystrix.contrib.javanica.annotation.HystrixProperty;
import com.sestar.userapi.api.IUserService;
import com.sestar.userapi.domain.User;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

import java.util.Collections;
import java.util.List;
import java.util.Random;

/**
 * @description Server Provider Controller
 * @author zhangxinxin
 * @date 2019/1/10 14:30
 **/
@RestController
public class UserServiceProviderController implements IUserService {

    /**
     * 用户服务中心
     **/
    private final IUserService userServer;

    /**
     * 随机数工具类
     **/
    private static final Random random = new Random();

    @Value("${server.port}")
    private String port;

    @Autowired
    public UserServiceProviderController(@Qualifier("inMemoryUserService") IUserService userServer) {
        this.userServer = userServer;
    }

    // 通过方法继承，URL 映射 ："/user/save"
    @Override
    public boolean saveUser(@RequestBody User user) {
        return userServer.saveUser(user);
    }

    // 通过方法继承，URL 映射 ："/user/find/all"
    @HystrixCommand(
            commandProperties = {   // command 配置
                    @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "100") // 设置最大执行时间
            },
            fallbackMethod = "fallbackForGetUsers" // 设置熔断后执行回滚方法
    )
    @Override
    public List<User> findAll() {
        return userServer.findAll();
    }

    // ...省略其他不相关内容
}
```

<br />

#### ServiceImpl 实现用户服务API接口

```Java
package com.sestar.userserverprovider.web.service;

import com.sestar.userapi.api.IUserService;
import com.sestar.userapi.domain.User;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * @description 用户服务
 * @author zhangxinxin
 * @date 2019/2/15 20:42
 **/
@Service("inMemoryUserService")
public class InMemoryUserService implements IUserService {

    private Map<Long, User> repository = new ConcurrentHashMap<>();

    @Override
    public boolean saveUser(User user) {
        return repository.put(user.getId(), user) == null;
    }

    @Override
    public List<User> findAll() {
        return new ArrayList(repository.values());
    }

}
```

<br />

### 搭建客户端

<br />

#### Maven依赖

```xml
<!-- 依赖 用户 API -->
<dependency>
    <groupId>com.sestar</groupId>
    <artifactId>user-api</artifactId>
    <version>${project.version}</version>
</dependency>

<!-- Eureka Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>

<!-- Ribbon -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-ribbon</artifactId>
</dependency>

<!-- Netflix Hystrix -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-hystrix</artifactId>
</dependency>

<!-- Config Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-client</artifactId>
</dependency>
```

<br />

#### application.properties

```properties
## Ribbon
### 配置名称可以看 org.springframework.cloud.netflix.ribbon.PropertiesFactory
user-service-provider-application.ribbon.NFLoadBalancerRuleClassName = com.sestar.userserviceclient.loadbalanced.MyRule
user-service-provider-application.ribbon.NFLoadBalancerPingClassName = com.sestar.userserviceclient.loadbalanced.MyPing
```

<br />

#### bootstrap.properties

```properties
## 应用名称
spring.application.name = user-service-client-application

##  Eureka
### spring-eureka-server-application注册到 Eureka 服务器中
### ${}都为服务器的配置内容 http://服务器地址:${server.port}/${server.context-path}/eureka
### 如果 bootstrap 有配置 spring.cloud.config.discovery.service-id, 则需要把该配置移到 bootstrap 中
### 因为*.discovery.service-id属性是查找 Eureka 服务配置端, 而查找范围就是eureka.client.service-url.defaultZone
### 需要最先加载 Eureka 注册信息, eureka.client.service-url.defaultZone才能确定范围,
### 而 *.discovery.service-id属性配置在 bootstrap 中, 比 application 配置先加载,
### 所以当有配置Eureka服务配置端, 则需要把 eureka.client.service-url.defaultZone必须配置在 bootstrap 中
eureka.client.serviceUrl.defaultZone = http://127.0.0.1:3701/eureka

## 配置客户端应用关联的应用
## spring.cloud.config.name 是可选的
## 如果没有配置，采用 ${spring.application.name}
spring.cloud.config.name = user-server
## 关联profile
spring.cloud.config.profile = default
## 关联label
spring.cloud.config.label = master
## 激活 Config Server 服务发现和 spring.cloud.config.discovery.serviceId 捆绑配置
spring.cloud.config.discovery.enabled = true
## Config Server 服务器应用名称
spring.cloud.config.discovery.serviceId = config-server-application
```

<br />

#### 启动类注册RibbonClient, Eureka Client; 开启Hystrix, FeignClient

```Java
package com.sestar.userserviceclient;

import com.sestar.userapi.api.IUserService;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.cloud.netflix.feign.EnableFeignClients;
import org.springframework.cloud.netflix.ribbon.RibbonClient;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

@SpringBootApplication
@RibbonClient(name = "user-service-provider-application")
@EnableDiscoveryClient // 使用Eureka服务发现方式发现服务提供端
@EnableCircuitBreaker  // Netflix Hystrix 熔断器
@EnableFeignClients(clients = IUserService.class) // 申明 UserService 接口作为 Feign Client 调用
public class UserServiceClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserServiceClientApplication.class, args);
    }

    /**
     * @description 获取用以负载均衡的RestTemplate的Bean对象,
     *          注释@LoadBalanced将会把RestTemplate也注册为 LoadBalancerClient,
     *          所以如果自动注入LoadBalancerClient对象也是本方法返回对象
     *          而且使用该RestTemplate.execute调用的也是LoadBalancerClient#execute
     * @author zhangxinxin
     * @date 2019/1/10 14:43
     * @return org.springframework.web.client.RestTemplate
     **/
    @Bean(name = "loadBalanceRestTemplate")
    @LoadBalanced
    public RestTemplate getLoadBalanceRestTemplate() {
        return new RestTemplate();
    }

    // ...省略其他不相关内容
}
```

<br />

#### Controller调用用户服务API的接口

```Java
package com.sestar.userserviceclient.web.controller;

import com.sestar.userapi.api.IUserService;
import com.sestar.userapi.domain.User;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

/**
 * @description Server Client Controller
 * @author zhangxinxin
 * @date 2019/2/15 21:30
 **/
@RestController
public class UserServiceClientController {

    /**
     * 用户服务中心
     **/
    @SuppressWarnings("all")
    @Autowired
    private IUserService userServer;

    @PostMapping("/user/save")
    public boolean saveUser(@RequestBody User user) {
        return userServer.saveUser(user);
    }

    @GetMapping("/user/find/all")
    public List<User> findAll() {
        return userServer.findAll();
    }

}
```

<br />

### 搭建路由管理端

<br />

#### Maven依赖

```xml
<!-- Spring Cloud Netflix Zuul -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zuul</artifactId>
</dependency>

<!-- Eureka Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>

<!-- Config Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-client</artifactId>
</dependency>
```

<br />

#### application.properties

```properties
## 应用服务
server.port = 3731

## Actuator
### 主要是 zuul.RouteEndPoint(/routes) 安全权限打开
management.security.enabled = false
```

<br />

#### bootstrap.properties

```properties
## 应用名称
spring.application.name = zuul-proxy-application

##  Eureka
### spring-eureka-server-application注册到 Eureka 服务器中
### ${}都为服务器的配置内容 http://服务器地址:${server.port}/${server.context-path}/eureka
### 如果 bootstrap 有配置 spring.cloud.config.discovery.service-id, 则需要把该配置移到 bootstrap 中
### 因为*.discovery.service-id属性是查找 Eureka 服务配置端, 而查找范围就是eureka.client.service-url.defaultZone
### 需要最先加载 Eureka 注册信息, eureka.client.service-url.defaultZone才能确定范围,
### 而 *.discovery.service-id属性配置在 bootstrap 中, 比 application 配置先加载,
### 所以当有配置Eureka服务配置端, 则需要把 eureka.client.service-url.defaultZone必须配置在 bootstrap 中
eureka.client.serviceUrl.defaultZone = http://127.0.0.1:3701/eureka

## 配置客户端应用关联的应用
## spring.cloud.config.name 是可选的
## 如果没有配置，采用 ${spring.application.name}
spring.cloud.config.name = zuul-config
## 关联profile
spring.cloud.config.profile = default
## 关联label
spring.cloud.config.label = master
## 激活 Config Server 服务发现和 spring.cloud.config.discovery.serviceId 捆绑配置
spring.cloud.config.discovery.enabled = true
## Config Server 服务器应用名称
spring.cloud.config.discovery.serviceId = config-server-application
```

<br />

#### 启动类注册Zuul路由代理

```java
package com.sestar.zuulproxy;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.zuul.EnableZuulProxy;

/**
 * @description Zuul 路由代理启动类
 * @author zhangxinxin
 * @date 2019/2/27 9:36
 **/
@EnableZuulProxy    // 注册Zuul路由代理
@SpringBootApplication
public class ZuulProxyApplication {

    public static void main(String[] args) {
        SpringApplication.run(ZuulProxyApplication.class, args);
    }

}
```

<br />

### 测试路由代理

```text
项目启动顺序:
    1. eureka-server (用以Eureka服务发现, 必定第一个启动)
    2. config-server (给 user-service-client, zuul-proxy 获取profile文件, 肯定需要比user-service-client,
                      zuul-proxy项目先启动)
    3. user-service-provider (eureka项目端, zuul-proxy项目也会将该项目注册路由, 所以肯定要比zuul-proxy先启动)
    4. user-service-client (eureka项目端, zuul-proxy项目也会将该项目注册路由, 所以肯定要比zuul-proxy先启动)
    5. zuul-proxy (需要把所有的eureka项目端都注册路由, 所以需要最后一个启动)

开始测试路由代理:
    1. 访问 http://127.0.0.1:3731/routes
    2. 查看路由代理Map是否完整
        {
            /user-service-client/**: "user-service-client-application", # 静态路由, config-server的profile中获取
            /user-service-provider/**: "user-service-provider-application", # 静态路由, config-server的profile中获取
            /user-service-provider-application/**: "user-service-provider-application", # 动态路由, spring.application.name直接作为server-id
            /config-server-application/**: "config-server-application", # 动态路由, spring.application.name直接作为server-id
            /user-service-client-application/**: "user-service-client-application" # 动态路由, spring.application.name直接作为server-id
        }
    3. 访问静态路由: http://127.0.0.1:3731/user-service-provider/user/find/all, 访问成功代表静态路由注册成功
    4. 访问静态路由: http://127.0.0.1:3731/user-service-client-application/user/find/all, 访问成功代表动态路由注册成功
```

<br />

* 附： zuul-proxy模块加载config-server(服务配置端)的配置内容如下:
```text
## Zuul Proxy 配置信息
### 指定user-service-provider路由(zuul.routes后面需要eureka服务发现的应用名称)
zuul.routes.user-service-provider-application = /user-service-provider/**
### 指定user-service-client路由
zuul.routes.user-service-client-application = /user-service-client/**
```

<br />

## 消息驱动(Kafka, RabbitMQ, ActiveMQ)

[参考源码](https://github.com/Sestar/firstLearn/tree/master/spring-cloud/cloud-multiple)

<br />

### 结构图

<img width="30%" src="/note/_v_images/java/框架/SpringCloud/MQ-1.png">

<br />

### Kafka

<br />

#### ReadMeFirst

```text
源码是spring boot多个功能的集合, 本节主要介绍Kafka

Module:
    eureka-server(Eureka服务注册端), config-server(服务配置端),
    user-api(用户服务API), user-service-provider(服务提供端),
    user-service-client(客户端)

    user-service-client(客户端,充当生产者角色)发送消息, user-service-provider(服务提供端,充当消费者角色)订阅消息

    1. user-service-client 只需要 spring-kafka 依赖
    2. user-service-client 需要 application.properties 配置 Kafka-Producer(Kafka服务端地址,发送消息对象的序列化)
    4. user-service-provider 需要 spring-cloud-stream-binder-kafka 依赖
    5. user-service-provider 需要 application.properties 配置 kafka-service服务地址 和 消息消费管道的bindings
    6. 测试项目前需要先启动 Zookeeper 和 Kafka-Server 服务
```

<br />

#### Eureka Server

<br />

##### Maven依赖

```xml
<!-- Eureka Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
```

<br />

##### application.properties

```properties
# 应用名称
spring.application.name = eureka-server-application

# 应用服务
server.port = 3701

## Eureka
### 取消向注册中心注册
eureka.client.register-with-eureka = false
### 取消向注册中心获取注册信息(服务, 实例信息)
eureka.client.fetch-registry = false
## 将应用注册到 Eureka 服务器, 解决 peer 问题
### 当*.hostname 和 *.serviceUrl.defaultZone的 protocol一致时, Eureka服务端将不会再去寻找其他Eureka服务端
eureka.instance.hostname = 127.0.0.1
eureka.client.serviceUrl.defaultZone = http://${eureka.instance.hostname}:${server.port}/eureka
```

<br />

##### 启动类注册为 Eureka Server

```java
package com.sestar.eurekaserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer // 注册 Eureka Server
public class EurekaServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaServerApplication.class, args);
	}

}
```

##### 测试Eureka Server搭建成功

> 访问 http://127.0.0.1:3701/spring-eureka-server 能进入Eureka页面, 即Eureka Server搭建成功

<br />

#### Config Server

<br />

##### Maven依赖

```xml
<!-- Spring Cloud Config Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>

<!-- Eureka Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```

<br />

##### application.properties

```properties
## Eureka
### Eureka 注册
eureka.client.serviceUrl.defaultZone = http://127.0.0.1:3701/eureka
### Eureka 使用 IP 注册, 不使用IP注册user-service-client无法找到config-server, 具体原因不知道
eureka.instance.prefer-ip-address = true

## 配置服务器文件系统git仓库, 配置文件git仓库最好是git单独的repository
spring.cloud.config.server.git.uri = https://github.com/Sestar/cloud-config.git
## 强制拉取git内容, 如果本地副本是脏数据, 将强制从远程仓库拉取配置
spring.cloud.config.server.git.force-pull = true
```

<br />

##### 启动类注册Eureka Client, Config Server

```java
package com.sestar.configserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.config.server.EnableConfigServer;

/**
 * @description Config Server
 * @author zhangxinxin
 * @date 2019/2/18 14:15
 **/
@SpringBootApplication
@EnableDiscoveryClient  // 注册为 Eureka-Client
@EnableConfigServer     // 注册为 Cloud Server
public class ConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }

}
```

<br />

##### 测试 Config-Server 搭建成功

```text
访问url -> http://localhost:3711/zuul-config/default/master 成功返回就说明 Config-Server 搭建成功
```

<br />

#### User-API

<br />

##### Maven依赖

```xml
<!-- Lombok -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
</dependency>
```

<br />

##### Domain(消息的封装实体类)

```Java
package com.sestar.userapi.domain;

import lombok.Data;
import lombok.ToString;

import java.io.Serializable;

/**
 * @description 实体类
 * @author zhangxinxin
 * @date 2019/1/10 14:19
 **/
@Data
@ToString
public class User implements Serializable {

    private static final long serialVersionUID = -4961274264021393330L;

    /**
     * 账户姓名
     */
    private String name;

    /**
     * 账户标识
     */
    private Long id;

    /**
     * 账户简介, 该字段无需序列化
     */
    private transient String desc;

}
```

```text
Domain实体类实现 java.io.Serializable 序列化接口, 是为了消息的消费者也是订阅者(user-service-provider)解析消息时, 能够转化为字节流
```

<br />

#### User-Service-Client

<br />

##### Maven 依赖

```xml
<!-- 用户 API -->
<dependency>
    <groupId>com.sestar</groupId>
    <artifactId>user-api</artifactId>
    <version>${project.version}</version>
</dependency>

<!-- Eureka Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>

<!-- Config Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-client</artifactId>
</dependency>

<!-- Kafka -->
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

<br />

##### 启动类注册 Eureka-Client

```java
package com.sestar.userserviceclient;

import com.sestar.userapi.api.IUserService;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

@SpringBootApplication
@EnableDiscoveryClient // 注册 Eureka-Client
public class UserServiceClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserServiceClientApplication.class, args);
    }

}
```

<br />

##### 生产者发送消息序列化配置类

```java
package com.sestar.userserviceclient.serializer;

import org.apache.kafka.common.serialization.Serializer;

import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.ObjectOutputStream;
import java.util.Map;

/**
 * @description 生产者发送消息序列化(Producer)
 * @author zhangxinxin
 * @date 2019/3/11 20:46
 **/
public class ObjectSerializer implements Serializer {

    @Override
    public void configure(Map configs, boolean isKey) {

    }

    @Override
    public byte[] serialize(String topic, Object data) {
        try {
            System.out.println(this.getClass().getName() + "#serialize -> 输出生产者发送的内容：\r\n" + data);
            ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
            ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);
            objectOutputStream.writeObject(data);

            return byteArrayOutputStream.toByteArray();
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public void close() {

    }
}
```

<br />

##### application.properites

```properties
## 应用名称
spring.application.name = user-service-client-application

## 应用服务
server.port = 3721

## Actuator
management.security.enabled = false

## Kafka-Producer
### Kafka服务端口
spring.kafka.bootstrap-servers = 127.0.0.1:9092
### Serializer
spring.kafka.producer.value-serializer = com.sestar.userserviceclient.serializer.ObjectSerializer
```

##### bootstrap.properties

```properties
## 应用名称
spring.application.name = user-service-client-application

##  Eureka
### spring-eureka-server-application注册到 Eureka 服务器中
### ${}都为服务器的配置内容 http://服务器地址:${server.port}/${server.context-path}/eureka
### 如果 bootstrap 有配置 spring.cloud.config.discovery.service-id, 则需要把该配置移到 bootstrap 中
### 因为*.discovery.service-id属性是查找 Eureka 服务配置端, 而查找范围就是eureka.client.service-url.defaultZone
### 需要最先加载 Eureka 注册信息, eureka.client.service-url.defaultZone才能确定范围,
### 而 *.discovery.service-id属性配置在 bootstrap 中, 比 application 配置先加载,
### 所以当有配置Eureka服务配置端, 则需要把 eureka.client.service-url.defaultZone必须配置在 bootstrap 中
eureka.client.serviceUrl.defaultZone = http://127.0.0.1:3701/eureka

## 配置客户端应用关联的应用
## spring.cloud.config.name 是可选的
## 如果没有配置，采用 ${spring.application.name}
spring.cloud.config.name = user-server
## 关联profile
spring.cloud.config.profile = default
## 关联label
spring.cloud.config.label = master
## 激活 Config Server 服务发现和 spring.cloud.config.discovery.serviceId 捆绑配置
spring.cloud.config.discovery.enabled = true
## Config Server 服务器应用名称
spring.cloud.config.discovery.serviceId = config-server-application
```

<br />

##### 消息生产接口

```java
package com.sestar.userserviceclient.web.controller;

import com.sestar.userapi.domain.User;
import com.sestar.userserviceclient.stream.ProducerChannel;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.support.SendResult;
import org.springframework.util.concurrent.ListenableFuture;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

/**
 * @description Server Client Controller
 * @author zhangxinxin
 * @date 2019/2/15 21:30
 **/
@RestController
@EnableBinding({ProducerChannel.class}) // 绑定用户信息接口
public class UserServiceClientController {

    /**
     * Kafka Template
     */
    @SuppressWarnings("all")
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;

    @Autowired
    public UserServiceClientController(KafkaTemplate kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    /**
     * 测试 KafkaTemplate 发送消息
     */
    @PostMapping("/mq/send/user")
    public boolean sendUserByMQ(@RequestBody User user) {
        ListenableFuture<SendResult<String, Object>> future = kafkaTemplate.send("replicated-topic", user);
        return future.isDone();
    }

}
```

<br />

#### User-Service-Provider

<br />

##### Maven依赖

```xml
<!-- 用户 API -->
<dependency>
    <groupId>com.sestar</groupId>
    <artifactId>user-api</artifactId>
    <version>${project.version}</version>
</dependency>

<!-- Eureka Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>

<!-- Kafka Binder -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-binder-kafka</artifactId>
</dependency>
```

<br />

##### 创建订阅管道

```java
package com.sestar.userserverprovider.stream;

import com.sestar.userapi.domain.User;
import org.springframework.cloud.stream.annotation.Input;
import org.springframework.messaging.SubscribableChannel;

/**
 * @description {@link User} 接收消息 stream 接口定义
 * @author zhangxinxin
 * @date 2019/3/12 16:07
 **/
public interface ConsumerChannel {

    String INPUT = "consumer-channel-1";

    // 管道
    @Input(INPUT)
    SubscribableChannel input();

}
```

<br />

##### 启动类注册 Eureka-Client，订阅管道绑定

```java
package com.sestar.userserverprovider;

import com.sestar.userserverprovider.stream.ConsumerChannel;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.hystrix.EnableHystrix;
import org.springframework.cloud.stream.annotation.EnableBinding;

@SpringBootApplication
@EnableDiscoveryClient // 注册为 Eureka-Client
@EnableBinding(ConsumerChannel.class) // 订阅管道绑定
public class UserServerProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserServerProviderApplication.class, args);
    }

}
```

<br />

##### application.properties

```properties
### 通用配置
## 应用名称
spring.application.name = user-service-provider-application

## 应用服务
server.port = 3741

## Eureka
### Eureka开关
#eureka.client.enabled = false
### Eureka注册
eureka.client.serviceUrl.defaultZone = http://127.0.0.1:3701/eureka
### 上传注册信息时间间隔
eureka.client.instance-info-replication-interval-seconds = 5
### 获取注册信息时间间隔
eureka.client.registry-fetch-interval-seconds = 5
### Eureka注册名使用IP
eureka.instance.prefer-ip-address = true

## Spring Cloud Binding Stream
### MQ管道, accept-channel-1:管道名称, replicated-topic:topic名称
spring.cloud.stream.bindings.consumer-channel-1.destination = replicated-topic

## Kafka-Consumer
### Kafka服务端口
spring.kafka.bootstrap-servers = 127.0.0.1:9092
```

<br />

##### 消息订阅服务

```java
package com.sestar.userserverprovider.web.service;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.sestar.userapi.api.IUserService;
import com.sestar.userapi.domain.User;
import com.sestar.userserverprovider.stream.ConsumerChannel;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.messaging.SubscribableChannel;
import org.springframework.stereotype.Service;

import javax.annotation.PostConstruct;
import java.io.ByteArrayInputStream;
import java.io.ObjectInputStream;

import static com.sestar.userserverprovider.stream.ConsumerChannel.INPUT;

/**
 * @description 用户消息订阅服务
 * @author zhangxinxin
 * @date 2019/3/12 16:34
 **/
@Service
public class UserMessageService {

    /**
     * 接收消息 stream 接口定义
     */
    private ConsumerChannel consumerChannel;

    /**
     * 用户服务接口
     */
    private IUserService userService;

    /**
     * Object -> String 工具类
     */
    private ObjectMapper objectMapper;

    /**
     * ConsumerChannel Bean会通过 @EnableBinding 生成，但是一直爆红，使用SuppressWarnings去除错误
     */
    @SuppressWarnings("all")
    @Autowired
    public UserMessageService(ConsumerChannel consumerChannel, @Qualifier("inMemoryUserService") IUserService userService,
                              ObjectMapper objectMapper) {
        this.consumerChannel = consumerChannel;
        this.userService = userService;
        this.objectMapper = objectMapper;
    }

    /**
     * @description @StreamListener 方式接收消息, 和@ServiceActivator, SubscribableChannel功能一样,
     *                  应用场景: Spring-Cloud
     * @author zhangxinxin
     * @date 2019/3/12 16:59
     */
    @StreamListener(INPUT)
    public void onMessage(String data) {
        System.out.println(this.getClass().getName() + "#onMessage -> Subscribe by @StreamListener");
        saveUser(data);
    }

    /**
     * @description @ServiceActivator 方式接收消息, 和@ServiceActivator, SubscribableChannel功能一样,
     *                  应用场景: integration
     * @author zhangxinxin
     * @date 2019/3/12 16:59
     */
    @ServiceActivator(inputChannel = INPUT)
    public void listen(String data) {
        System.out.println(this.getClass().getName() + "#listen -> Subscribe by @ServiceActivator");
        saveUser(data);
    }

    /**
     * @description SubscribableChannel 管道方式接收消息, 和@ServiceActivator, SubscribableChannel功能一样,
     *                  应用场景: messaging
     * @author zhangxinxin
     * @date 2019/3/12 16:59
     */
    @PostConstruct
    public void init() {
        // 获取管道
        SubscribableChannel channel = consumerChannel.input();
        channel.subscribe(message -> {
            System.out.println(this.getClass().getName() + "#init -> Subscribe by SubscribableChannel");
            String contentType = message.getHeaders().get("contentType", String.class);
            if ("text/plain".equals(contentType)) {
                saveUser((String) message.getPayload());
            } else {
                saveUser((byte[]) message.getPayload());
            }
        });
    }

    /**
     * @description 处理MQ字节流数据
     * @author zhangxinxin
     * @date 2019/3/12 16:57
     * @param data 管道字节消息
     */
    private void saveUser(byte[] data) {
        try {
            // 将字节流数据封装成对象
            ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(data);
            ObjectInputStream objectInputStream = new ObjectInputStream(byteArrayInputStream);
            // 反序列化 User 对象
            User user = (User) objectInputStream.readObject();
            System.out.println(this.getClass().getName() + "#saveUser -> 订阅消息内容如下: " + user);
            // 调用服务
            userService.saveUser(user);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    /**
     * @description 处理MQ字符串数据
     * @author zhangxinxin
     * @date 2019/3/12 16:57
     * @param data 管道字节消息
     */
    private void saveUser(String data) {
        try {
            User user = objectMapper.readValue(data, User.class);
            System.out.println(this.getClass().getName() + "#saveUser -> 订阅消息内容如下: " + user);
            userService.saveUser(user);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

}
```



#### 测试 Kafka 消息互通

```text

1. 启动 Zookeeper (E:\ProgramFiles\zookeeper\bin\zkServer.cmd)
2. 打开文件路径 Kafka-Server(E:\ProgramFiles\kafka\bin\windows)
   本路径下执行命令 $ zookeeper-server-start.bat ../../config/server.properties

Zookeeper配置文件(E:\ProgramFiles\zookeeper\conf\zoo.cfg)中, clientPort配置的端口 和
Kafka命令行执行的server.properties配置文件中的 zookeeper.connect 端口需要一直(默认是2181)

3. 启动 Maven 项目
  启动模块顺序：1. Eureka Server  2. Config-Server  3. User-Service-Client  4. User-Service-Provider (3和4俩模块顺序无所谓)

4. 使用postman POST访问 -> http:127.0.0.1:3721/mq/send/user -> 入参Body: {"id": 1, "name": "what", "desc": "hahah"}

5. user-service-client 的 Console 输出:

    'com.sestar.userserviceclient.serializer.ObjectSerializer#serialize -> 输出生产者发送的内容：
     User(name=what, id=1, desc=hahah)'

6. user-service-provider 的 Console 输出(出现StringIndexOutOfBoundException异常和正常功能不冲突, 但是不知道为什么会爆错):

    'com.sestar.userserverprovider.web.service.UserMessageService#onMessage -> Subscribe by @StreamListener
     com.sestar.userserverprovider.web.service.UserMessageService#listen -> Subscribe by @ServiceActivator
     com.sestar.userserverprovider.web.service.UserMessageService#init -> Subscribe by SubscribableChannel
     com.sestar.userserverprovider.web.service.UserMessageService#saveUser -> 订阅消息内容如下:
     User(name=what, id=1, desc=null)'
```

<br />

### RabbitMQ

<br />

#### ReadMeFirst

```text
源码是spring boot多个功能的集合, 本节主要介绍RabbitMQ

Module:
    eureka-server(Eureka服务注册端), config-server(服务配置端),
    user-api(用户服务API), user-service-provider(服务提供端),
    user-service-client(客户端)

    user-service-client(客户端,充当生产者角色)发送消息, user-service-provider(服务提供端,充当消费者角色)订阅消息

查看Rabbit Binder项目:
    1. user-service-client 需要 spring-cloud-stream-binder-rabbit 依赖
    2. user-service-client 需要 application.properties 配置消息生产管道的bindings(如果修改rabbit默认配置,则需要配置rabbit链接属性)
    3. user-service-client 激活消息生产管道(启动类加上注解 @EnableBinding(ProducerChannel.class))
    4. user-service-provider 需要 spring-cloud-stream-binder-rabbit 依赖
    5. user-service-provider 需要 application.properties 配置消息消费管道的bindings(如果修改rabbit默认配置,则需要配置rabbit链接属性)
    6. user-service-provider 激活消息消费管道(启动类加上注解 @EnableBinding(ConsumerChannel.class))
```

<br />

#### Eureka Server

<br />

##### Maven依赖

```xml
<!-- Eureka Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
```

<br />

##### application.properties

```properties
# 应用名称
spring.application.name = eureka-server-application

# 应用服务
server.port = 3701

## Eureka
### 取消向注册中心注册
eureka.client.register-with-eureka = false
### 取消向注册中心获取注册信息(服务, 实例信息)
eureka.client.fetch-registry = false
## 将应用注册到 Eureka 服务器, 解决 peer 问题
### 当*.hostname 和 *.serviceUrl.defaultZone的 protocol一致时, Eureka服务端将不会再去寻找其他Eureka服务端
eureka.instance.hostname = 127.0.0.1
eureka.client.serviceUrl.defaultZone = http://${eureka.instance.hostname}:${server.port}/eureka
```

<br />

##### 启动类注册为 Eureka Server

```java
package com.sestar.eurekaserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer // 注册 Eureka Server
public class EurekaServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaServerApplication.class, args);
	}

}
```

##### 测试Eureka Server搭建成功

> 访问 http://127.0.0.1:3701/spring-eureka-server 能进入Eureka页面, 即Eureka Server搭建成功

<br />

#### Config Server

<br />

##### Maven依赖

```xml
<!-- Spring Cloud Config Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>

<!-- Eureka Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```

<br />

##### application.properties

```properties
## Eureka
### Eureka 注册
eureka.client.serviceUrl.defaultZone = http://127.0.0.1:3701/eureka
### Eureka 使用 IP 注册, 不使用IP注册user-service-client无法找到config-server, 具体原因不知道
eureka.instance.prefer-ip-address = true

## 配置服务器文件系统git仓库, 配置文件git仓库最好是git单独的repository
spring.cloud.config.server.git.uri = https://github.com/Sestar/cloud-config.git
## 强制拉取git内容, 如果本地副本是脏数据, 将强制从远程仓库拉取配置
spring.cloud.config.server.git.force-pull = true
```

<br />

##### 启动类注册Eureka Client, Config Server

```java
package com.sestar.configserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.config.server.EnableConfigServer;

/**
 * @description Config Server
 * @author zhangxinxin
 * @date 2019/2/18 14:15
 **/
@SpringBootApplication
@EnableDiscoveryClient  // 注册为 Eureka-Client
@EnableConfigServer     // 注册为 Cloud Server
public class ConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }

}
```

<br />

##### 测试 Config-Server 搭建成功

```text
访问url -> http://localhost:3711/zuul-config/default/master 成功返回就说明 Config-Server 搭建成功
```

<br />

#### User-API

<br />

##### Maven依赖

```xml
<!-- Lombok -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
</dependency>
```

<br />

##### Domain(消息的封装实体类)

```Java
package com.sestar.userapi.domain;

import lombok.Data;
import lombok.ToString;

import java.io.Serializable;

/**
 * @description 实体类
 * @author zhangxinxin
 * @date 2019/1/10 14:19
 **/
@Data
@ToString
public class User implements Serializable {

    private static final long serialVersionUID = -4961274264021393330L;

    /**
     * 账户姓名
     */
    private String name;

    /**
     * 账户标识
     */
    private Long id;

    /**
     * 账户简介, 该字段无需序列化
     */
    private transient String desc;

}
```

```text
Domain实体类实现 java.io.Serializable 序列化接口, 是为了消息的消费者也是订阅者(user-service-provider)解析消息时, 能够转化为字节流
```

<br />

#### User-Service-Client

<br />

##### Maven 依赖

```xml
<!-- 用户 API -->
<dependency>
    <groupId>com.sestar</groupId>
    <artifactId>user-api</artifactId>
    <version>${project.version}</version>
</dependency>

<!-- Eureka Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>

<!-- Config Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-client</artifactId>
</dependency>

<!-- Spring Cloud Stream Binder Rabbit MQ -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-binder-rabbit</artifactId>
</dependency>
```

<br />

##### 创建消息生产管道

```java
package com.sestar.userserviceclient.stream;

import com.sestar.userapi.domain.User;
import org.springframework.cloud.stream.annotation.Output;
import org.springframework.messaging.MessageChannel;

/**
 * @description {@link User} 发送消息 stream 接口定义
 * @author zhangxinxin
 * @date 2019/3/13 16:19
 **/
public interface ProducerChannel {

    String OUTPUT = "producer-channel-1";

    @Output(OUTPUT)
    MessageChannel output();

}
```

<br />

##### 启动类注册 Eureka-Client

```java
package com.sestar.userserviceclient;

import com.sestar.userserviceclient.stream.ProducerChannel;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.stream.annotation.EnableBinding;

@SpringBootApplication
@EnableDiscoveryClient // 注册 Eureka-Client
@EnableBinding({ProducerChannel.class}) // 绑定消息生产者接口
public class UserServiceClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserServiceClientApplication.class, args);
    }

}
```

<br />

##### application.properites

```properties
## 应用名称
spring.application.name = user-service-client-application

## 应用服务
server.port = 3721

## Actuator
management.security.enabled = false


## Spring Cloud Binding Stream
### 解决多个 binder 问题
spring.cloud.stream.default-binder = rabbit
### 所有MQ通用管道bindings配置, consumer-channel-1:管道名称, replicated-topic:topic名称
spring.cloud.stream.bindings.producer-channel-1.destination = replicated-topic

## RabbitMQ
### 如果rabbit服务默认配置有修改, 则需要配置rabbitmq的连接属性
spring.rabbitmq.host = 127.0.0.1
spring.rabbitmq.port = 5672
spring.rabbitmq.username = guest
spring.rabbitmq.password = guest
```

##### bootstrap.properties

```properties
## 应用名称
spring.application.name = user-service-client-application

##  Eureka
### spring-eureka-server-application注册到 Eureka 服务器中
### ${}都为服务器的配置内容 http://服务器地址:${server.port}/${server.context-path}/eureka
### 如果 bootstrap 有配置 spring.cloud.config.discovery.service-id, 则需要把该配置移到 bootstrap 中
### 因为*.discovery.service-id属性是查找 Eureka 服务配置端, 而查找范围就是eureka.client.service-url.defaultZone
### 需要最先加载 Eureka 注册信息, eureka.client.service-url.defaultZone才能确定范围,
### 而 *.discovery.service-id属性配置在 bootstrap 中, 比 application 配置先加载,
### 所以当有配置Eureka服务配置端, 则需要把 eureka.client.service-url.defaultZone必须配置在 bootstrap 中
eureka.client.serviceUrl.defaultZone = http://127.0.0.1:3701/eureka

## 配置客户端应用关联的应用
## spring.cloud.config.name 是可选的
## 如果没有配置，采用 ${spring.application.name}
spring.cloud.config.name = user-server
## 关联profile
spring.cloud.config.profile = default
## 关联label
spring.cloud.config.label = master
## 激活 Config Server 服务发现和 spring.cloud.config.discovery.serviceId 捆绑配置
spring.cloud.config.discovery.enabled = true
## Config Server 服务器应用名称
spring.cloud.config.discovery.serviceId = config-server-application
```

<br />

##### 消息生产服务

```java
package com.sestar.userserviceclient.web.controller;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.sestar.userapi.domain.User;
import com.sestar.userserviceclient.stream.ProducerChannel;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.support.GenericMessage;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

/**
 * @description Server Client Controller
 * @author zhangxinxin
 * @date 2019/2/15 21:30
 **/
@RestController
@EnableBinding({ProducerChannel.class}) // 绑定用户信息接口
public class UserServiceClientController {

    /**
     * 发送消息 stream 接口定义
     */
    private ProducerChannel producerChannel;

    /**
     * Object -> String 工具类
     */
    private ObjectMapper objectMapper;

    /**
     * producerChannel Bean会通过 @EnableBinding 生成，但是一直爆红，使用SuppressWarnings去除错误
     */
    @SuppressWarnings("all")
    @Autowired
    public UserServiceClientController(ProducerChannel producerChannel, ObjectMapper objectMapper) {
        this.producerChannel = producerChannel;
        this.objectMapper = objectMapper;
    }

    /**
     * 测试 Rabbit Binder 发送消息
     */
    @PostMapping("/rabbit/send/user")
    public boolean sendUserByRibbon(@RequestBody User user) throws JsonProcessingException {
        System.out.println(this.getClass().getName() + "#sendUserByRibbon 获取入参: " + user);
        MessageChannel output1 = producerChannel.output();
        // User 序列化 JSON
        String payload1 = objectMapper.writeValueAsString(user);
        GenericMessage<String> msg1 = new GenericMessage<String>(payload1);
        // 发送消息
        return output1.send(msg1);
    }

}
```

<br />

#### User-Service-Provider

<br />

##### Maven依赖

```xml
<!-- 用户 API -->
<dependency>
    <groupId>com.sestar</groupId>
    <artifactId>user-api</artifactId>
    <version>${project.version}</version>
</dependency>

<!-- Eureka Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>

<!-- Spring Cloud Stream Binder Rabbit MQ -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-binder-rabbit</artifactId>
</dependency>
```

<br />

##### 创建消息订阅管道

```java
package com.sestar.userserverprovider.stream;

import com.sestar.userapi.domain.User;
import org.springframework.cloud.stream.annotation.Input;
import org.springframework.messaging.SubscribableChannel;

/**
 * @description {@link User} 接收消息 stream 接口定义
 * @author zhangxinxin
 * @date 2019/3/12 16:07
 **/
public interface ConsumerChannel {

    String INPUT = "consumer-channel-1";

    // 管道
    @Input(INPUT)
    SubscribableChannel input();

}
```

<br />

##### 启动类注册 Eureka-Client，订阅管道绑定

```java
package com.sestar.userserverprovider;

import com.sestar.userserverprovider.stream.ConsumerChannel;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.hystrix.EnableHystrix;
import org.springframework.cloud.stream.annotation.EnableBinding;

@SpringBootApplication
@EnableDiscoveryClient // 注册为 Eureka-Client
@EnableBinding(ConsumerChannel.class) // 订阅管道绑定
public class UserServerProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserServerProviderApplication.class, args);
    }

}
```

<br />

##### application.properties

```properties
### 通用配置
## 应用名称
spring.application.name = user-service-provider-application

## 应用服务
server.port = 3741

## Eureka
### Eureka开关
#eureka.client.enabled = false
### Eureka注册
eureka.client.serviceUrl.defaultZone = http://127.0.0.1:3701/eureka
### 上传注册信息时间间隔
eureka.client.instance-info-replication-interval-seconds = 5
### 获取注册信息时间间隔
eureka.client.registry-fetch-interval-seconds = 5
### Eureka注册名使用IP
eureka.instance.prefer-ip-address = true

## Spring Cloud Binding Stream
### 解决多个 binder 问题
spring.cloud.stream.default-binder = rabbit
### MQ管道, accept-channel-1:管道名称, replicated-topic:topic名称
spring.cloud.stream.bindings.consumer-channel-1.destination = replicated-topic

## RabbitMQ
### 如果rabbit服务默认配置有修改, 则需要配置rabbitmq的连接属性
spring.rabbitmq.host = 127.0.0.1
spring.rabbitmq.port = 5672
spring.rabbitmq.username = guest
spring.rabbitmq.password = guest
```

<br />

##### 消息订阅监听

```java
package com.sestar.userserverprovider.web.service;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.sestar.userapi.api.IUserService;
import com.sestar.userapi.domain.User;
import com.sestar.userserverprovider.stream.ConsumerChannel;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.messaging.SubscribableChannel;
import org.springframework.stereotype.Service;

import javax.annotation.PostConstruct;
import java.io.ByteArrayInputStream;
import java.io.ObjectInputStream;

import static com.sestar.userserverprovider.stream.ConsumerChannel.INPUT;

/**
 * @description 用户消息订阅服务
 * @author zhangxinxin
 * @date 2019/3/12 16:34
 **/
@Service
public class UserMessageService {

    /**
     * 接收消息 stream 接口定义
     */
    private ConsumerChannel consumerChannel;

    /**
     * 用户服务接口
     */
    private IUserService userService;

    /**
     * Object -> String 工具类
     */
    private ObjectMapper objectMapper;

    /**
     * ConsumerChannel Bean会通过 @EnableBinding 生成，但是一直爆红，使用SuppressWarnings去除错误
     */
    @SuppressWarnings("all")
    @Autowired
    public UserMessageService(ConsumerChannel consumerChannel, @Qualifier("inMemoryUserService") IUserService userService,
                              ObjectMapper objectMapper) {
        this.consumerChannel = consumerChannel;
        this.userService = userService;
        this.objectMapper = objectMapper;
    }

    /**
     * @description @StreamListener 方式接收消息, 和@ServiceActivator, SubscribableChannel功能一样,
     *                  应用场景: Spring-Cloud
     * @author zhangxinxin
     * @date 2019/3/12 16:59
     */
    @StreamListener(INPUT)
    public void onMessage(String data) {
        System.out.println(this.getClass().getName() + "#onMessage -> Subscribe by @StreamListener");
        saveUser(data);
    }

    /**
     * @description @ServiceActivator 方式接收消息, 和@ServiceActivator, SubscribableChannel功能一样,
     *                  应用场景: integration
     * @author zhangxinxin
     * @date 2019/3/12 16:59
     */
    @ServiceActivator(inputChannel = INPUT)
    public void listen(String data) {
        System.out.println(this.getClass().getName() + "#listen -> Subscribe by @ServiceActivator");
        saveUser(data);
    }

    /**
     * @description SubscribableChannel 管道方式接收消息, 和@ServiceActivator, SubscribableChannel功能一样,
     *                  应用场景: messaging
     * @author zhangxinxin
     * @date 2019/3/12 16:59
     */
    @PostConstruct
    public void init() {
        // 获取管道
        SubscribableChannel channel = consumerChannel.input();
        channel.subscribe(message -> {
            System.out.println(this.getClass().getName() + "#init -> Subscribe by SubscribableChannel");
            String contentType = message.getHeaders().get("contentType", String.class);
            if ("text/plain".equals(contentType)) {
                saveUser((String) message.getPayload());
            } else {
                saveUser((byte[]) message.getPayload());
            }
        });
    }

    /**
     * @description 处理MQ字节流数据
     * @author zhangxinxin
     * @date 2019/3/12 16:57
     * @param data 管道字节消息
     */
    private void saveUser(byte[] data) {
        try {
            // 将字节流数据封装成对象
            ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(data);
            ObjectInputStream objectInputStream = new ObjectInputStream(byteArrayInputStream);
            // 反序列化 User 对象
            User user = (User) objectInputStream.readObject();
            System.out.println(this.getClass().getName() + "#saveUser -> 订阅消息内容如下: " + user);
            // 调用服务
            userService.saveUser(user);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    /**
     * @description 处理MQ字符串数据
     * @author zhangxinxin
     * @date 2019/3/12 16:57
     * @param data 管道字节消息
     */
    private void saveUser(String data) {
        try {
            User user = objectMapper.readValue(data, User.class);
            System.out.println(this.getClass().getName() + "#saveUser -> 订阅消息内容如下: " + user);
            userService.saveUser(user);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

}
```



#### 测试 Rabbit 消息互通

```text

1. RabbitMQ 服务

2. 启动 Maven 项目
  启动模块顺序：1. Eureka Server  2. Config-Server  3. User-Service-Client  4. User-Service-Provider (3和4俩模块顺序无所谓)

4. 使用postman POST访问 -> http:127.0.0.1:3721/rabbit/send/user -> 入参Body: {"id": 1, "name": "what", "desc": "hahah"}

5. user-service-client 的 Console 输出:

    'com.sestar.userserviceclient.web.controller.UserServiceClientController$$EnhancerBySpringCGLIB$$ff3e087a#sendUserByRibbon 获取入参: User(name=what, id=1, desc=nanana)'

6. user-service-provider 的 Console 输出(出现StringIndexOutOfBoundException异常和正常功能不冲突, 但是不知道为什么会爆错):

    'com.sestar.userserverprovider.web.service.UserMessageService#init -> Subscribe by SubscribableChannel
     com.sestar.userserverprovider.web.service.UserMessageService#saveUser -> 订阅消息内容如下: User(name=what, id=1, desc=nanana)'
```

<br />

### ActiveMQ

<br />

#### ReadMeFirst

```text
源码是spring boot多个功能的集合, 本节主要介绍RabbitMQ

Module:
    eureka-server(Eureka服务注册端), config-server(服务配置端),
    user-api(用户服务API), user-service-provider(服务提供端),
    user-service-client(客户端), stream-binder-activemq(ActiveMQ端)

    user-service-client(客户端,充当生产者角色)发送消息, user-service-provider(服务提供端,充当消费者角色)订阅消息

查看Rabbit Binder项目:
    1. user-service-client 需要 stream-binder-activemq 依赖
    2. user-service-client 需要 application.properties 配置消息生产管道的destination,binder和 ActiveMQ 链接配置
    3. user-service-client 激活消息生产管道(启动类加上注解 @EnableBinding(ProducerChannel.class))
    4. user-service-provider 需要 stream-binder-activemq 依赖
    5. user-service-provider 需要 application.properties 配置消息订阅管道的destination,binder和 ActiveMQ 链接配置
    6. user-service-provider 激活消息消费管道(启动类加上注解 @EnableBinding(ConsumerChannel.class))
```

<br />

#### Eureka Server

<br />

##### Maven依赖

```xml
<!-- Eureka Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
```

<br />

##### application.properties

```properties
# 应用名称
spring.application.name = eureka-server-application

# 应用服务
server.port = 3701

## Eureka
### 取消向注册中心注册
eureka.client.register-with-eureka = false
### 取消向注册中心获取注册信息(服务, 实例信息)
eureka.client.fetch-registry = false
## 将应用注册到 Eureka 服务器, 解决 peer 问题
### 当*.hostname 和 *.serviceUrl.defaultZone的 protocol一致时, Eureka服务端将不会再去寻找其他Eureka服务端
eureka.instance.hostname = 127.0.0.1
eureka.client.serviceUrl.defaultZone = http://${eureka.instance.hostname}:${server.port}/eureka
```

<br />

##### 启动类注册为 Eureka Server

```java
package com.sestar.eurekaserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer // 注册 Eureka Server
public class EurekaServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaServerApplication.class, args);
	}

}
```

##### 测试Eureka Server搭建成功

> 访问 http://127.0.0.1:3701/spring-eureka-server 能进入Eureka页面, 即Eureka Server搭建成功

<br />

#### Config Server

<br />

##### Maven依赖

```xml
<!-- Spring Cloud Config Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>

<!-- Eureka Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```

<br />

##### application.properties

```properties
## Eureka
### Eureka 注册
eureka.client.serviceUrl.defaultZone = http://127.0.0.1:3701/eureka
### Eureka 使用 IP 注册, 不使用IP注册user-service-client无法找到config-server, 具体原因不知道
eureka.instance.prefer-ip-address = true

## 配置服务器文件系统git仓库, 配置文件git仓库最好是git单独的repository
spring.cloud.config.server.git.uri = https://github.com/Sestar/cloud-config.git
## 强制拉取git内容, 如果本地副本是脏数据, 将强制从远程仓库拉取配置
spring.cloud.config.server.git.force-pull = true
```

<br />

##### 启动类注册Eureka Client, Config Server

```java
package com.sestar.configserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.config.server.EnableConfigServer;

/**
 * @description Config Server
 * @author zhangxinxin
 * @date 2019/2/18 14:15
 **/
@SpringBootApplication
@EnableDiscoveryClient  // 注册为 Eureka-Client
@EnableConfigServer     // 注册为 Cloud Server
public class ConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }

}
```

<br />

##### 测试 Config-Server 搭建成功

```text
访问url -> http://localhost:3711/zuul-config/default/master 成功返回就说明 Config-Server 搭建成功
```

<br />

#### User-API

<br />

##### Maven依赖

```xml
<!-- Lombok -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
</dependency>
```

<br />

##### Domain(消息的封装实体类)

```Java
package com.sestar.userapi.domain;

import lombok.Data;
import lombok.ToString;

import java.io.Serializable;

/**
 * @description 实体类
 * @author zhangxinxin
 * @date 2019/1/10 14:19
 **/
@Data
@ToString
public class User implements Serializable {

    private static final long serialVersionUID = -4961274264021393330L;

    /**
     * 账户姓名
     */
    private String name;

    /**
     * 账户标识
     */
    private Long id;

    /**
     * 账户简介, 该字段无需序列化
     */
    private transient String desc;

}
```

```text
Domain实体类实现 java.io.Serializable 序列化接口, 是为了消息的消费者也是订阅者(user-service-provider)解析消息时, 能够转化为字节流
```

<br />

#### Stream-Binder-ActiveMQ

<br />

##### Maven 依赖

```xml
<!-- Spring Boot 自动装配依赖 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-autoconfigure</artifactId>
    <optional>true</optional>
</dependency>

<!-- Sprig Boot Starter ActiveMQ -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-activemq</artifactId>
</dependency>

<!-- Spring Cloud Stream 核心库 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream</artifactId>
</dependency>
```

<br />

##### Binder

```java
package com.sestar.streambinderactivemq;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.binder.Binder;
import org.springframework.cloud.stream.binder.Binding;
import org.springframework.cloud.stream.binder.ConsumerProperties;
import org.springframework.cloud.stream.binder.ProducerProperties;
import org.springframework.jms.core.JmsTemplate;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.SubscribableChannel;
import org.springframework.messaging.support.GenericMessage;
import org.springframework.util.Assert;

import javax.jms.Connection;
import javax.jms.ConnectionFactory;
import javax.jms.Destination;
import javax.jms.MessageConsumer;
import javax.jms.ObjectMessage;
import javax.jms.Session;

/**
 * @description Active MQ Message Channel Binder
 * @author zhangxinxin
 * @date 2019/3/29 14:52
 **/
public class ActiveMQMessageChannelBinder implements Binder<MessageChannel, ConsumerProperties, ProducerProperties> {

    @Autowired
    private JmsTemplate jmsTemplate;

    /**
     * @description 生产消息
     * @author zhangxinxin
     * @date 2019/3/29 14:54
     * @param name topic
     * @param outboundBindTarget 消息管道
     * @param producerProperties 生产者配置
     * @return org.springframework.cloud.stream.binder.Binding<org.springframework.messaging.MessageChannel>
     */
    @Override
    public Binding<MessageChannel> bindProducer(String name, MessageChannel outboundBindTarget, ProducerProperties producerProperties) {
        Assert.isInstanceOf(SubscribableChannel.class, outboundBindTarget,
                "Binding is supported only for SubscribableChannel instances");
        SubscribableChannel subscribableChannel = (SubscribableChannel) outboundBindTarget;

        subscribableChannel.subscribe(message -> {
            Object messageBody = message.getPayload();
            System.out.println(ActiveMQMessageChannelBinder.class.getName() + "#bindProducer 生产消息: " + messageBody);
            // 生产消息的目的为 name, 即 application.properties 中 default-destination
            jmsTemplate.convertAndSend(name, messageBody);
        });

        return () -> System.out.println(ActiveMQMessageChannelBinder.class.getName() + "#bindProducer Unbinding MessageChannel");
    }

    /**
     * @description 消费消息
     * @author zhangxinxin
     * @date 2019/3/29 14:54
     * @param name topic
     * @param group consumer-group
     * @param inboundBindTarget 消息管道
     * @param consumerProperties 消费者配置
     * @return org.springframework.cloud.stream.binder.Binding<org.springframework.messaging.MessageChannel>
     */
    @Override
    public Binding<MessageChannel> bindConsumer(String name, String group, MessageChannel inboundBindTarget, ConsumerProperties consumerProperties) {
        try {
            ConnectionFactory connectionFactory = jmsTemplate.getConnectionFactory();
            // 创建 jms 链接
            Connection connection = connectionFactory.createConnection();
            // 启动链接
            connection.start();
            // 创建会话
            Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
            // 创建消息目的
            Destination destination = session.createQueue(name);
            // 创建消息消费者
            MessageConsumer consumer = session.createConsumer(destination);
            // 消费消息
            consumer.setMessageListener(message -> {
                // message 来自于 ActiveMQ
                if (message instanceof ObjectMessage) {
                    ObjectMessage objectMessage = (ObjectMessage) message;
                    try {
                        Object object = objectMessage.getObject();
                        System.out.println(ActiveMQMessageChannelBinder.class.getName() + "#bindConsumer 消费消息: " + object);
                        // 订阅消息目的为 inboundBindTarget 绑定的topic, application.properties 中设置
                        inboundBindTarget.send(new GenericMessage<>(object), 100);
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            });

        } catch (Exception e) {
            e.printStackTrace();
        }

        return null;
    }

}
```

<br />

##### BinderAutoConfiguration

```java
package com.sestar.streambinderactivemq;

import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.cloud.stream.binder.Binder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * @description Active MQ Stream Binder 自动装配
 * @author zhangxinxin
 * @date 2019/3/29 15:11
 **/
@Configuration
@ConditionalOnMissingBean(Binder.class)
public class ActiveMQStreamBinderAutoConfiguration {

    @Bean
    public ActiveMQMessageChannelBinder getActiveMQMessageChannelBinder() {
        return new ActiveMQMessageChannelBinder();
    }

}
```

<br />

##### 配置加载MQ自定义Binder

```text
添加文件:
    - resources
    -- META-INF
    --- spring.binders

文件内容:
activemq-demo:\
com.sestar.streambinderactivemq.ActiveMQStreamBinderAutoConfiguration
```

#### User-Service-Client

<br />

##### Maven 依赖

```xml
<!-- 用户 API -->
<dependency>
    <groupId>com.sestar</groupId>
    <artifactId>user-api</artifactId>
    <version>${project.version}</version>
</dependency>

<!-- Eureka Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>

<!-- Config Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-client</artifactId>
</dependency>

<!-- 依赖 stream-binder-activemq -->
<dependency>
    <groupId>${project.groupId}</groupId>
    <artifactId>stream-binder-activemq</artifactId>
    <version>${project.version}</version>
</dependency>
```

<br />

##### 创建消息生产管道

```java
package com.sestar.userserviceclient.stream;

import com.sestar.userapi.domain.User;
import org.springframework.cloud.stream.annotation.Output;
import org.springframework.messaging.MessageChannel;

/**
 * @description {@link User} 发送消息 stream 接口定义
 * @author zhangxinxin
 * @date 2019/3/13 16:19
 **/
public interface ProducerChannel {

    String OUTPUT = "producer-channel-1";

    @Output(OUTPUT)
    MessageChannel output();

}
```

<br />

##### 启动类注册 Eureka-Client

```java
package com.sestar.userserviceclient;

import com.sestar.userserviceclient.stream.ProducerChannel;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.stream.annotation.EnableBinding;

@SpringBootApplication
@EnableDiscoveryClient // 注册 Eureka-Client
@EnableBinding({ProducerChannel.class}) // 绑定消息生产者接口
public class UserServiceClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserServiceClientApplication.class, args);
    }

}
```

<br />

##### application.properites

```properties
## 应用名称
spring.application.name = user-service-client-application

## 应用服务
server.port = 3721

## Actuator
management.security.enabled = false

## Spring Cloud Binding Stream
### 解决多个 binder 问题
spring.cloud.stream.default-binder = activemq-demo
### RabbitMQ, Kafka,ActiveMQ 管道bindings配置, consumer-channel-1:管道名称, replicated-topic:topic名称
spring.cloud.stream.bindings.producer-channel-1.destination = replicated-topic
### binder指定MQ的执行对象, activemq-demo 是 stream-binder-activemq 的 META-INF 配置的 binder 名称
spring.cloud.stream.bindings.producer-channel-1.binder = activemq-demo

## ActiveMQ
### 链接配置
### brokeUrl
spring.activemq.broker-url = tcp://localhost:61616
### JMS消息目的
spring.jms.template.default-destination = replicated-topic
```

##### bootstrap.properties

```properties
## 应用名称
spring.application.name = user-service-client-application

##  Eureka
### spring-eureka-server-application注册到 Eureka 服务器中
### ${}都为服务器的配置内容 http://服务器地址:${server.port}/${server.context-path}/eureka
### 如果 bootstrap 有配置 spring.cloud.config.discovery.service-id, 则需要把该配置移到 bootstrap 中
### 因为*.discovery.service-id属性是查找 Eureka 服务配置端, 而查找范围就是eureka.client.service-url.defaultZone
### 需要最先加载 Eureka 注册信息, eureka.client.service-url.defaultZone才能确定范围,
### 而 *.discovery.service-id属性配置在 bootstrap 中, 比 application 配置先加载,
### 所以当有配置Eureka服务配置端, 则需要把 eureka.client.service-url.defaultZone必须配置在 bootstrap 中
eureka.client.serviceUrl.defaultZone = http://127.0.0.1:3701/eureka

## 配置客户端应用关联的应用
## spring.cloud.config.name 是可选的
## 如果没有配置，采用 ${spring.application.name}
spring.cloud.config.name = user-server
## 关联profile
spring.cloud.config.profile = default
## 关联label
spring.cloud.config.label = master
## 激活 Config Server 服务发现和 spring.cloud.config.discovery.serviceId 捆绑配置
spring.cloud.config.discovery.enabled = true
## Config Server 服务器应用名称
spring.cloud.config.discovery.serviceId = config-server-application
```

<br />

##### 消息生产服务

```java
package com.sestar.userserviceclient.web.controller;

import com.sestar.userapi.domain.User;
import com.sestar.userserviceclient.stream.ProducerChannel;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jms.core.JmsTemplate;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.support.GenericMessage;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

/**
 * @description Server Client Controller
 * @author zhangxinxin
 * @date 2019/2/15 21:30
 **/
@RestController
public class UserServiceClientController {

    /**
     * 发送消息 stream 接口定义
     */
    private ProducerChannel producerChannel;

    /**
     * JMS Template
     */
    private JmsTemplate jmsTemplate;

    /**
     * producerChannel Bean会通过 @EnableBinding 生成，但是一直爆红，不影响功能
     */
    @Autowired
    public UserServiceClientController(ProducerChannel producerChannel, JmsTemplate jmsTemplate) {
        this.producerChannel = producerChannel;
        this.jmsTemplate = jmsTemplate;
    }

    /**
     * 测试 ActiveMQ 发送消息
     */
    @PostMapping("/activemq/send/user/myBinder")
    public boolean sendUserByMyBinder(@RequestBody User user) {
        System.out.println(this.getClass().getName() + "#sendUserByMyBinder 生产消息: " + user);
        MessageChannel messageChannel = producerChannel.output();
        GenericMessage<User> message = new GenericMessage<>(user);
        return messageChannel.send(message);
    }

}
```

<br />

#### User-Service-Provider

<br />

##### Maven依赖

```xml
<!-- 用户 API -->
<dependency>
    <groupId>com.sestar</groupId>
    <artifactId>user-api</artifactId>
    <version>${project.version}</version>
</dependency>

<!-- Eureka Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>

<!-- 依赖 stream-binder-activemq -->
<dependency>
    <groupId>${project.groupId}</groupId>
    <artifactId>stream-binder-activemq</artifactId>
    <version>${project.version}</version>
</dependency>
```

<br />

##### 创建消息订阅管道

```java
package com.sestar.userserverprovider.stream;

import com.sestar.userapi.domain.User;
import org.springframework.cloud.stream.annotation.Input;
import org.springframework.messaging.SubscribableChannel;

/**
 * @description {@link User} 接收消息 stream 接口定义
 * @author zhangxinxin
 * @date 2019/3/12 16:07
 **/
public interface ConsumerChannel {

    String INPUT = "consumer-channel-1";

    // 管道
    @Input(INPUT)
    SubscribableChannel input();

}
```

<br />

##### 启动类注册 Eureka-Client，订阅管道绑定

```java
package com.sestar.userserverprovider;

import com.sestar.userserverprovider.stream.ConsumerChannel;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.hystrix.EnableHystrix;
import org.springframework.cloud.stream.annotation.EnableBinding;

@SpringBootApplication
@EnableDiscoveryClient // 注册为 Eureka-Client
@EnableBinding(ConsumerChannel.class) // 订阅管道绑定
public class UserServerProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserServerProviderApplication.class, args);
    }

}
```

<br />

##### application.properties

```properties
### 通用配置
## 应用名称
spring.application.name = user-service-provider-application

## 应用服务
server.port = 3741

## Eureka
### Eureka开关
#eureka.client.enabled = false
### Eureka注册
eureka.client.serviceUrl.defaultZone = http://127.0.0.1:3701/eureka
### 上传注册信息时间间隔
eureka.client.instance-info-replication-interval-seconds = 5
### 获取注册信息时间间隔
eureka.client.registry-fetch-interval-seconds = 5
### Eureka注册名使用IP
eureka.instance.prefer-ip-address = true

## Spring Cloud Binding Stream
### 解决多个 binder 问题
spring.cloud.stream.default-binder = activemq-demo
### RabbitMQ, Kafka 管道bindings配置, consumer-channel-1:管道名称, replicated-topic:topic名称
spring.cloud.stream.bindings.consumer-channel-1.destination = replicated-topic
spring.cloud.stream.bindings.consumer-channel-1.binder = activemq-demo
## ActiveMQ
### brokeUrl
spring.activemq.broker-url = tcp://localhost:61616
### JMS消息目的
spring.jms.template.default-destination = replicated-topic
### 添加 User 实体类信任(若没有添加信任, 消息会消费, 但是不会被解析)
spring.activemq.packages.trusted = java.lang, com.sestar.userapi.domain
```

<br />

#### 测试 ActiveMQ 消息互通

```text

1. ActiveMQ 服务

2. 启动 Maven 项目
  启动模块顺序：1. Eureka Server  2. Config-Server  3. User-Service-Client  4. User-Service-Provider (3和4俩模块顺序无所谓)

4. 使用postman POST访问 -> http://localhost:3721/activemq/send/user/myBinder -> 入参Body: {"id": 1, "name": "what", "desc": "hahah"}

5. user-service-client 的 Console 输出:

    'com.sestar.userserviceclient.web.controller.UserServiceClientController#sendUserByMyBinder 生产消息: User(name=what, id=1, desc=hahah)
com.sestar.streambinderactivemq.ActiveMQMessageChannelBinder#bindProducer 生产消息: User(name=what, id=1, desc=hahah)'

6. user-service-provider 的 Console 输出(出现StringIndexOutOfBoundException异常和正常功能不冲突, 但是不知道为什么会爆错):

    'com.sestar.streambinderactivemq.ActiveMQMessageChannelBinder#bindConsumer 消费消息: User(name=what, id=1, desc=null)'
```

<br />

## 消息总线

[参考源码](https://github.com/Sestar/firstLearn/tree/master/spring-cloud/cloud-multiple)

<br />

### ReadMeFirst

```text
源码是spring boot多个功能的集合, 本节主要介绍消息总线(Spring Cloud Bus)

Module:
    eureka-server(Eureka服务注册端), config-server(服务配置端),
    user-api(用户服务API), user-service-client(客户端)

user-service-client 使用的消息驱动是 `Rabbit`
```

<br />

### Eureka Server

<br />

#### Maven依赖

```xml
<!-- Eureka Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
```

<br />

#### application.properties

```properties
# 应用名称
spring.application.name = eureka-server-application

# 应用服务
server.port = 3701

## Eureka
### 取消向注册中心注册
eureka.client.register-with-eureka = false
### 取消向注册中心获取注册信息(服务, 实例信息)
eureka.client.fetch-registry = false
## 将应用注册到 Eureka 服务器, 解决 peer 问题
### 当*.hostname 和 *.serviceUrl.defaultZone的 protocol一致时, Eureka服务端将不会再去寻找其他Eureka服务端
eureka.instance.hostname = 127.0.0.1
eureka.client.serviceUrl.defaultZone = http://${eureka.instance.hostname}:${server.port}/eureka
```

<br />

#### 启动类注册为 Eureka Server

```java
package com.sestar.eurekaserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer // 注册 Eureka Server
public class EurekaServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaServerApplication.class, args);
	}

}
```

#### 测试Eureka Server搭建成功

> 访问 http://127.0.0.1:3701/spring-eureka-server 能进入Eureka页面, 即Eureka Server搭建成功

<br />

### Config Server

<br />

#### Maven依赖

```xml
<!-- Spring Cloud Config Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>

<!-- Eureka Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```

<br />

#### application.properties

```properties
## Eureka
### Eureka 注册
eureka.client.serviceUrl.defaultZone = http://127.0.0.1:3701/eureka
### Eureka 使用 IP 注册, 不使用IP注册user-service-client无法找到config-server, 具体原因不知道
eureka.instance.prefer-ip-address = true

## 配置服务器文件系统git仓库, 配置文件git仓库最好是git单独的repository
spring.cloud.config.server.git.uri = https://github.com/Sestar/cloud-config.git
## 强制拉取git内容, 如果本地副本是脏数据, 将强制从远程仓库拉取配置
spring.cloud.config.server.git.force-pull = true
```

<br />

#### 启动类注册Eureka Client, Config Server

```java
package com.sestar.configserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.config.server.EnableConfigServer;

/**
 * @description Config Server
 * @author zhangxinxin
 * @date 2019/2/18 14:15
 **/
@SpringBootApplication
@EnableDiscoveryClient  // 注册为 Eureka-Client
@EnableConfigServer     // 注册为 Cloud Server
public class ConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }

}
```

<br />

#### 测试 Config-Server 搭建成功

```text
访问url -> http://localhost:3711/zuul-config/default/master 成功返回就说明 Config-Server 搭建成功
```

<br />

### User-API

<br />

#### Maven依赖

```xml
<!-- Lombok -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
</dependency>

<!-- Spring Cloud Bus -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-bus</artifactId>
    <optional>true</optional>
</dependency>
```

<br />

#### Domain(消息的封装实体类)

```Java
package com.sestar.userapi.domain;

import lombok.Data;
import lombok.ToString;

import java.io.Serializable;

/**
 * @description 实体类
 * @author zhangxinxin
 * @date 2019/1/10 14:19
 **/
@Data
@ToString
public class User implements Serializable {

    private static final long serialVersionUID = -4961274264021393330L;

    /**
     * 账户姓名
     */
    private String name;

    /**
     * 账户标识
     */
    private Long id;

    /**
     * 账户简介, 该字段无需序列化
     */
    private transient String desc;

}
```

```text
Domain实体类实现 java.io.Serializable 序列化接口, 是为了消息的消费者也是订阅者(user-service-provider)解析消息时, 能够转化为字节流
```

<br />

#### 自定义 RemoteApplicationEvent

```java
package com.sestar.userapi.bus.applicationEvent;

import com.sestar.userapi.domain.User;
import org.springframework.cloud.bus.event.RemoteApplicationEvent;

/**
 * @description 自定义 RemoteApplicationEvent, 实现监听 publishEvent 自定义事件
 * @author zhangxinxin
 * @date 2019/4/2 16:31
 **/
public class UserRemoteApplicationEvent extends RemoteApplicationEvent {

    /**
     * @description 必须要有一个无参数构造函数, 否则监听事件会报错
     * @author zhangxinxin
     * @date 2019/4/3 17:05
     */
    public UserRemoteApplicationEvent() {}

    public UserRemoteApplicationEvent(User user, String originService, String destinationService) {
        super(user, originService, destinationService);
    }

}
```

<br />

### User-Service-Client

<br />

#### Maven 依赖

```xml
<!-- 依赖 用户 API -->
<dependency>
    <groupId>com.sestar</groupId>
    <artifactId>user-api</artifactId>
    <version>${project.version}</version>
</dependency>

<!-- Eureka Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>

<!-- Config Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-client</artifactId>
</dependency>

<!-- Spring Cloud Bus : AMQP -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

<br />

#### 启动类注册 Eureka-Client

```java
package com.sestar.userserviceclient;

import com.sestar.userapi.api.IUserService;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

@SpringBootApplication
@EnableDiscoveryClient // 注册 Eureka-Client
public class UserServiceClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserServiceClientApplication.class, args);
    }

}
```

<br />

#### application.properites

```properties
## 应用名称
spring.application.name = user-service-client-application

## 应用服务
server.port = 3721

## Actuator
management.security.enabled = false

## RabbitMQ
### 如果rabbit服务默认配置有修改, 则需要配置rabbitmq的连接属性
spring.rabbitmq.host = 127.0.0.1
spring.rabbitmq.port = 5672
spring.rabbitmq.username = guest
spring.rabbitmq.password = guest

## Bus 配置
### 激活 Bus 跟踪
spring.cloud.bus.trace.enabled = true
```

#### bootstrap.properties

```properties
## 应用名称
spring.application.name = user-service-client-application

##  Eureka
### spring-eureka-server-application注册到 Eureka 服务器中
### ${}都为服务器的配置内容 http://服务器地址:${server.port}/${server.context-path}/eureka
### 如果 bootstrap 有配置 spring.cloud.config.discovery.service-id, 则需要把该配置移到 bootstrap 中
### 因为*.discovery.service-id属性是查找 Eureka 服务配置端, 而查找范围就是eureka.client.service-url.defaultZone
### 需要最先加载 Eureka 注册信息, eureka.client.service-url.defaultZone才能确定范围,
### 而 *.discovery.service-id属性配置在 bootstrap 中, 比 application 配置先加载,
### 所以当有配置Eureka服务配置端, 则需要把 eureka.client.service-url.defaultZone必须配置在 bootstrap 中
eureka.client.serviceUrl.defaultZone = http://127.0.0.1:3701/eureka

## 配置客户端应用关联的应用
## spring.cloud.config.name 是可选的
## 如果没有配置，采用 ${spring.application.name}
spring.cloud.config.name = user-server
## 关联profile
spring.cloud.config.profile = default
## 关联label
spring.cloud.config.label = master
## 激活 Config Server 服务发现和 spring.cloud.config.discovery.serviceId 捆绑配置
spring.cloud.config.discovery.enabled = true
## Config Server 服务器应用名称
spring.cloud.config.discovery.serviceId = config-server-application
```

<br />

#### publishEvent 自定义 RemoteApplicationEvent 

```java
package com.sestar.userserviceclient.web.controller;

import com.sestar.userapi.bus.applicationEvent.UserRemoteApplicationEvent;
import com.sestar.userapi.domain.User;
import org.springframework.beans.BeansException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.context.ApplicationEventPublisherAware;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

/**
 * @description 生成 {@link com.sestar.userapi.bus.applicationEvent.UserRemoteApplicationEvent} 事件,
 * 触发 {@link com.sestar.userserviceclient.bus.config.BusConfiguration}#onUserRemoteApplicationEvent
 * @author zhangxinxin
 * @date 2019/4/2 16:36
 **/
@RestController
public class BusEventController implements ApplicationContextAware, ApplicationEventPublisherAware {

    private ApplicationContext applicationContext;

    private ApplicationEventPublisher applicationEventPublisher;

    /**
     * @description applicationEventPublisher 发布自定义 RemoteApplicationEvent
     * @author zhangxinxin
     * @date 2019/4/8 17:32
     * @param user 消息内容
     * @param destination 消息目的
     */
    @PostMapping("/bus/event/publish/user")
    public void publishUserRemoteApplication(@RequestBody User user,
                                             @RequestParam(value = "destination", required = false) String destination) {
        String originService = applicationContext.getId();
        UserRemoteApplicationEvent userRemoteApplicationEvent = new UserRemoteApplicationEvent(user, originService, destination);
        applicationEventPublisher.publishEvent(userRemoteApplicationEvent);
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
        this.applicationEventPublisher = applicationEventPublisher;
    }
}
```

<br />

#### 监听事件配置类

```Java
package com.sestar.userserviceclient.bus.config;

import com.sestar.userapi.bus.applicationEvent.UserRemoteApplicationEvent;
import org.springframework.cloud.bus.SpringCloudBusClient;
import org.springframework.cloud.bus.event.AckRemoteApplicationEvent;
import org.springframework.cloud.bus.event.EnvironmentChangeRemoteApplicationEvent;
import org.springframework.cloud.bus.event.RefreshRemoteApplicationEvent;
import org.springframework.cloud.bus.jackson.RemoteApplicationEventScan;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.event.EventListener;

/**
 * @description 消息总线配置类
 * @author zhangxinxin
 * @date 2019/4/2 10:27
 **/
@Configuration
@RemoteApplicationEventScan(basePackageClasses = UserRemoteApplicationEvent.class) // 激活 UserRemoteApplicationEvent 监听
public class BusConfiguration {

    /**
     * @description 监听 RefreshRemoteApplicationEvent
     *    POST 访问 http://localhost:3721/bus/refresh?destination=user-service-client:**
     *    发现日志：o.s.cloud.bus.event.RefreshListener      : Received remote refresh request. Keys refreshed []
     *    可见, bus/refresh 的 EndPoint 是由 RefreshListener 监听, 查看 RefreshListener 源码,
     *    监听的是 RefreshRemoteApplicationEvent 事件
     * @author zhangxinxin
     * @date 2019/4/2 10:29
     */
    @EventListener
    public void onRefreshRemoteApplicationEvent(RefreshRemoteApplicationEvent event) {
        System.out.printf(BusConfiguration.class.getName() + "#onRefreshRemoteApplicationEvent\r\n" +
                "监听事件 ->  Source: %s, OriginService: %s, DestinationService: %s\r\n",
                event.getSource(), event.getOriginService(), event.getDestinationService());
    }

    /**
     * @description 监听 EnvironmentChangeRemoteApplicationEvent
     *    POST 访问 http://localhost:3721/bus/env
     *    发现日志：o.s.c.b.event.EnvironmentChangeListener  : Received remote environment change request. Keys/values to update {}
     *    可见, bus/env 的 EndPoint是由 EnvironmentChangeListener 监听, 查看 EnvironmentChangeListener 源码,
     *    监听的是 EnvironmentChangeRemoteApplicationEvent 事件
     * @author zhangxinxin
     * @date 2019/4/2 10:29
     */
    @EventListener
    public void onEnvironmentChangeRemoteApplicationEvent(EnvironmentChangeRemoteApplicationEvent event) {
        System.out.printf(BusConfiguration.class.getName() + "#onEnvironmentChangeRemoteApplicationEvent\r\n" +
                        "监听事件 ->  Source: %s, OriginService: %s, DestinationService: %s\r\n",
                event.getSource(), event.getOriginService(), event.getDestinationService());
    }

    /**
     * @description 监听 SpringCloudBusClient.OUTPUT 管道的 AckRemoteApplicationEvent
     * BusAutoConfiguration#acceptRemote 源码中 this.applicationEventPublisher.publishEvent(ack); 触发
     * 会和 #onEnvironmentChangeRemoteApplicationEvent 事件一起触发
     * 最后 ack(即AckRemoteApplicationEvent) 会生产消息到 cloudBusOutboundChannel 中,
     * 等 BusAutoConfiguration 生产消息后再查看其中事件发生源和目的源生
     * @author zhangxinxin
     * @date 2019/4/2 16:23
     */
    @StreamListener(SpringCloudBusClient.OUTPUT)
    public void onAckRemoteApplicationEvent(AckRemoteApplicationEvent event) {
        System.out.printf(BusConfiguration.class.getName() + "#onAckRemoteApplicationEvent\r\n" +
                        "监听事件 ->  Source: %s, OriginService: %s, DestinationService: %s\r\n",
                event.getSource(), event.getOriginService(), event.getDestinationService());
    }

    /**
     * @description 监听 {@link UserRemoteApplicationEvent}
     * POST 访问自定义接口 BusEventController#publishUserRemoteApplication
     * POST 访问 http://localhost:8080/bus/event/publish/user?destination=user-service-client-application:8080
     * json 入参 {"id": 1, "name": "sestar", "desc": "hahaha"}
     * @author zhangxinxin
     * @date 2019/4/2 10:29
     */
    @EventListener
    public void onUserRemoteApplicationEvent(UserRemoteApplicationEvent event) {
        System.out.printf(BusConfiguration.class.getName() + "#onUserRemoteApplicationEvent\r\n" +
                        "监听事件 ->  Source: %s, OriginService: %s, DestinationService: %s\r\n",
                event.getSource(), event.getOriginService(), event.getDestinationService());
    }

}
```

<br />

#### 测试监听事件

<br />

> 监听 RefreshRemoteApplicationEvent

```text
POST 访问 http://localhost:3721/bus/refresh?destination=user-service-client:**

程序将会重新获取配置内容, 输出相关内容, 但是只有一个监听器日志
o.s.cloud.bus.event.RefreshListener      : Received remote refresh request. Keys refreshed []
可见, bus/refresh 的 EndPoint 是由 RefreshListener 监听, 查看 RefreshListener 源码, 监听的是 RefreshRemoteApplicationEvent

并且日志中有输出 BusConfiguration 抓取内容
com.sestar.userserviceclient.bus.config.BusConfiguration#onRefreshRemoteApplicationEvent
监听事件 ->  Source: org.springframework.cloud.bus.endpoint.RefreshBusEndpoint@5f14590c,
OriginService: user-service-client-application:3721, DestinationService: user-service-client:**
```

> 监听 EnvironmentChangeRemoteApplicationEvent

```text
POST 访问 http://localhost:3721/bus/env

日志内容:
o.s.c.b.event.EnvironmentChangeListener  : Received remote environment change request. Keys/values to update {}
可见, bus/env 的 EndPoint是由 EnvironmentChangeListener 监听, 查看 EnvironmentChangeListener 源码,
监听的是EnvironmentChangeRemoteApplicationEvent

并且日志有输出 BusConfiguration 抓取内容
com.sestar.userserviceclient.bus.config.BusConfiguration#onEnvironmentChangeRemoteApplicationEvent
监听事件 ->  Source: org.springframework.cloud.bus.endpoint.EnvironmentBusEndpoint@78f35e39,
OriginService: user-service-client-application:3721, DestinationService: **
也会输出 AckRemoteApplicationEvent 监听内容, 具体原因将在下面介绍
```

> 监听 AckRemoteApplicationEvent

```text
源码: BusAutoConfiguration#acceptLocal 有监听 RemoteApplicationEvent, 如果该事件是本身发布,
将会把该事件生产 MQ 到 SpringCloudBusClient.INPUT 管道, 并且将会被 BusAutoConfiguration#acceptRemote 监听到。
this.cloudBusOutboundChannel.send(MessageBuilder.withPayload(ack).build()); this.applicationEventPublisher.publishEvent(ack);
这两个代码将会发布 AckRemoteApplicationEvent, 并将 AckRemoteApplicationEvent 封装成 MQ 给 SpringCloudBusClient.INPUT 管道

所以监听 EnvironmentChangeRemoteApplicationEvent 时, BusAutoConfiguration#acceptRemote 将会发布一个 AckRemoteApplicationEvent
两个 Event 将会同时触发监听器, 而自定义 Bus 配置类中 BusConfiguration#onAckRemoteApplicationEvent 将会监听 SpringCloudBusClient.INPUT 管道
中 AckRemoteApplicationEvent, 所以能够抓取 RemoteApplicationEvent 监听后又发布的 AckRemoteApplicationEvent 的监听内容
日志有输出 BusConfiguration 抓取内容
com.sestar.userserviceclient.bus.config.BusConfiguration#onAckRemoteApplicationEvent
监听事件 ->  Source: java.lang.Object@28c13d49, OriginService: user-service-client-application:3721, DestinationService: **
```

> 监听自定义 UserRemoteApplicationEvent

```text
自定义 RemoteApplicationEvent 是为了更好得理解 BusAutoConfiguration#acceptRemote 的思路
启动三个 UserServiceClientApplication, 端口分别是: 8080, 8081, 8082

调用 BusEventController#publishUserRemoteApplication 的 Mapper 地址
POST 访问 http://localhost:8080/bus/event/publish/user?destination=user-service-client-application:8082
json 入参: {"id": 1,	"name": "sestar",	"desc": "asdfsadf"}
即将会实现 8080 端会发送一个 UserRemoteApplicationEvent 给 8082 端

8080 端日志:
com.sestar.userserviceclient.bus.config.BusConfiguration#onUserRemoteApplicationEvent
监听事件 ->  Source: User(name=sestar, id=1, desc=asdfsadf), OriginService: user-service-client-application:8080, DestinationService: user-service-client-application:8082:**

8081 端日志: 无

8082 端日志:
com.sestar.userserviceclient.bus.config.BusConfiguration#onUserRemoteApplicationEvent
监听事件 ->  Source: java.lang.Object@31d1ec17, OriginService: user-service-client-application:8080, DestinationService: user-service-client-application:8082:**
com.sestar.userserviceclient.bus.config.BusConfiguration#onAckRemoteApplicationEvent
监听事件 ->  Source: java.lang.Object@31d1ec17, OriginService: user-service-client-application:8082, DestinationService: **

BusAutoConfiguration#acceptRemote 俩判断:
1. RemoteApplicationEvent 的源服务(OriginService)是本身, 则发布 RemoteApplicationEvent -> 8080 端将会发布UserRemoteApplicationEvent
2. RemoteApplicationEvent 的目的服务(DestinationService)是本身
    2.1 RemoteApplicationEvent 的源服务(OriginService)不是本身, 则发布 RemoteApplicationEvent -> 8082 端将会发布UserRemoteApplicationEvent
    2.2 spring.cloud.bus.trace.enabled(默认打开)如果打开, 则新建 AckRemoteApplicationEvent 发送给 SpringCloudBusClient.INPUT 管道且发布该事件
```

<br />

## 分布式应用跟踪

[参考源码](https://github.com/Sestar/firstLearn/tree/master/spring-cloud/cloud-multiple)

<br />

### ReadMeFirst

```text
源码是spring boot多个功能的集合, 本节主要介绍分布式应用跟踪(Spring Cloud Sleuth, Zipkin)

Module:
    eureka-server(Eureka服务注册端), config-server(服务配置端),
    user-api(用户服务API), user-service-client(客户端), user-service-provider(服务提供端)
    zipkin-server(微服务链路跟踪模块)

user-service-client 和 user-service-provider使用的 MQ 是 RabbitMQ, zipkin-server(使用Stream时使用的是Rabbit)

Spring Cloud Sleuth 有两种方式:
    1. Http 方式
        1. 不需要 user-api 模块
        2. zipkin-server 的 Maven 依赖有 zipkin-server 和 zipkin-autoconfigure-ui, 并且启动类使用 @EnableZipkinServer 注解激活 Zipkin
        3. user-service-client 的 Maven 依赖有 spring-cloud-starter-eureka, spring-cloud-config-client, spring-cloud-starter-zipkin
        4. user-service-provider 的 Maven 依赖有 spring-cloud-starter-eurek, spring-cloud-starter-zipkin
    2. Stream 方式
        1. 需要 user-api 模块
        2. zipkin-server 的 Maven 依赖有  zipkin-server 和 zipkin-autoconfigure-ui, spring-cloud-sleuth-zipkin-stream,
            spring-cloud-stream-binder-rabbit, 并且启动类使用@EnableZipkinStreamServer 注解激活 Zipkin Stream
        3. user-service-client 的 Maven 依赖有 user-api, spring-cloud-starter-eureka, spring-cloud-config-client,
            spring-cloud-stream-binder-rabbit, spring-cloud-starter-sleuth, spring-cloud-sleuth-stream
        4. user-service-provider 的 Maven 依赖有 user-api, spring-cloud-starter-eureka, spring-cloud-stream-binder-rabbit
            spring-cloud-starter-sleuth, spring-cloud-sleuth-stream

    两种方式的区别:
        1. Stream 方式和 Http 方式激活 Zipkin Server 的注解不一样, Stream 用的是 @EnableZipkinStreamServer, 而 Http 是 @EnabelZipkinServer
        2. Stream 引入的 Zipkin 依赖比 Http 多了 spring-cloud-sleuth-zipkin-stream 依赖
        3. Stream 方式相对于 Http 方式需要 MQ, 所以多了 rabbit 的依赖
```

<br />

### Eureka Server

<br />

#### Maven依赖

```xml
<!-- Eureka Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
```

<br />

#### application.properties

```properties
# 应用名称
spring.application.name = eureka-server-application

# 应用服务
server.port = 3701

## Eureka
### 取消向注册中心注册
eureka.client.register-with-eureka = false
### 取消向注册中心获取注册信息(服务, 实例信息)
eureka.client.fetch-registry = false
## 将应用注册到 Eureka 服务器, 解决 peer 问题
### 当*.hostname 和 *.serviceUrl.defaultZone的 protocol一致时, Eureka服务端将不会再去寻找其他Eureka服务端
eureka.instance.hostname = 127.0.0.1
eureka.client.serviceUrl.defaultZone = http://${eureka.instance.hostname}:${server.port}/eureka
```

<br />

#### 启动类注册为 Eureka Server

```java
package com.sestar.eurekaserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer // 注册 Eureka Server
public class EurekaServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaServerApplication.class, args);
	}

}
```

#### 测试Eureka Server搭建成功

> 访问 http://127.0.0.1:3701/spring-eureka-server 能进入Eureka页面, 即Eureka Server搭建成功

<br />

### Config Server

<br />

#### Maven依赖

```xml
<!-- Spring Cloud Config Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>

<!-- Eureka Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```

<br />

#### application.properties

```properties
## Eureka
### Eureka 注册
eureka.client.serviceUrl.defaultZone = http://127.0.0.1:3701/eureka
### Eureka 使用 IP 注册, 不使用IP注册user-service-client无法找到config-server, 具体原因不知道
eureka.instance.prefer-ip-address = true

## 配置服务器文件系统git仓库, 配置文件git仓库最好是git单独的repository
spring.cloud.config.server.git.uri = https://github.com/Sestar/cloud-config.git
## 强制拉取git内容, 如果本地副本是脏数据, 将强制从远程仓库拉取配置
spring.cloud.config.server.git.force-pull = true
```

<br />

#### 启动类注册Eureka Client, Config Server

```java
package com.sestar.configserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.config.server.EnableConfigServer;

/**
 * @description Config Server
 * @author zhangxinxin
 * @date 2019/2/18 14:15
 **/
@SpringBootApplication
@EnableDiscoveryClient  // 注册为 Eureka-Client
@EnableConfigServer     // 注册为 Cloud Server
public class ConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }

}
```

<br />

#### 测试 Config-Server 搭建成功

```text
访问url -> http://localhost:3711/zuul-config/default/master 成功返回就说明 Config-Server 搭建成功
```

<br />

### Zipkin

<br />

#### Maven 依赖

```xml
<!-- Zipkin Server -->
<dependency>
    <groupId>io.zipkin.java</groupId>
    <artifactId>zipkin-server</artifactId>
</dependency>

<!-- Zipkin Server UI -->
<dependency>
    <groupId>io.zipkin.java</groupId>
    <artifactId>zipkin-autoconfigure-ui</artifactId>
</dependency>

<!-- 下面两个是 Zipkin Stream 方式需要增加的两个依赖 -->
<!-- Spring Cloud Sleuth Stream  -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-sleuth-zipkin-stream</artifactId>
</dependency>

<!-- Spring Cloud Stream Binder Rabbit MQ -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-binder-rabbit</artifactId>
</dependency>
```

<br />

#### application.properties

```properties
## 应用元信息
## Zipkin 服务器应用名称
spring.application.name = zipkin-server-application

## Zipkin 服务器服务端口
server.port = 3751

## 管理端口安全失效
management.security.enabled = false
```

<br />

#### 启动类激活 Zipkin

```java
package com.sestar.zipkinserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import zipkin.server.EnableZipkinServer;

/**
 * @description Zipkin 服务器应用
 * @author zhangxinxin
 * @date 2019/4/8 20:52
 **/
@SpringBootApplication
//@EnableZipkinServer // 激活 Zipkin 服务
@EnableZipkinStreamServer // 激活 Zipkin Stream 服务
public class ZipkinServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ZipkinServerApplication.class, args);
    }

}
```

<br />

#### 测试 Zipkin 模块搭建成功

> 访问 url: http://localhost:3751 或者 http://localhost:3751/zipkin/ 能够进入 zipkin 的查看界面, 说明搭建 zipkin 应用跟踪模块成功

<br />

### User-API (Stream 方式需要)

<br />

#### Maven依赖

```xml
<!-- Lombok -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
</dependency>
```

<br />

#### Domain(消息的封装实体类)

```Java
package com.sestar.userapi.domain;

import lombok.Data;
import lombok.ToString;

import java.io.Serializable;

/**
 * @description 实体类
 * @author zhangxinxin
 * @date 2019/1/10 14:19
 **/
@Data
@ToString
public class User implements Serializable {

    private static final long serialVersionUID = -4961274264021393330L;

    /**
     * 账户姓名
     */
    private String name;

    /**
     * 账户标识
     */
    private Long id;

    /**
     * 账户简介, 该字段无需序列化
     */
    private transient String desc;

}
```

```text
Domain实体类实现 java.io.Serializable 序列化接口, 是为了消息的消费者也是订阅者(user-service-provider)解析消息时, 能够转化为字节流
```

<br />

#### 创建 Domain 的服务接口

```java
package com.sestar.userapi.api;

import com.sestar.userapi.domain.User;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;

import java.util.List;

/**
 * @description 用户服务API
 * @author zhangxinxin
 * @date 2019/2/15 18:00
 */
public interface IUserService {

    /**
     * @description 保存用户
     * @author zhangxinxin
     * @date 2019/2/15 17:59
     * @param user 用户信息
     * @return boolean
     */
    @PostMapping("/user/save")
    boolean saveUser(User user);

    /**
     * @description 查询所有的用户列表
     * @author zhangxinxin
     * @date 2019/2/15 17:59
     * @return java.util.List<com.sestar.userapi.domain.User>
     */
    @GetMapping("/user/find/all")
    List<User> findAll();

}
```

<br />

### User-Service-Client

<br />

#### Maven 依赖

<br />

##### Http 方式

```xml
<!-- Eureka Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>

<!-- Config Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-client</artifactId>
</dependency>

<!-- 下面一个依赖是 Zipkin Http 方式 -->
<!-- Spring Cloud Zipkin -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>
```

<br />

##### Stream 方式

```xml
<!-- 依赖 用户 API -->
<dependency>
    <groupId>com.sestar</groupId>
    <artifactId>user-api</artifactId>
    <version>${project.version}</version>
</dependency>

<!-- Eureka Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>

<!-- Config Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-client</artifactId>
</dependency>

<!-- Spring Cloud Stream Binder Rabbit MQ -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-binder-rabbit</artifactId>
</dependency>

<!-- 下面两个依赖是 Zipkin Stream 方式 并且还要加上一个 MQ -->
<!-- Spring Cloud Sleuth -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>

<!-- Spring Cloud Sleuth Stream -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-sleuth-stream</artifactId>
</dependency>
```

<br />

#### 创建 MQ 生产管道

```java
package com.sestar.userserviceclient.stream;

import com.sestar.userapi.domain.User;
import org.springframework.cloud.stream.annotation.Output;
import org.springframework.messaging.MessageChannel;

/**
 * @description {@link User} 发送消息 stream 接口定义
 * @author zhangxinxin
 * @date 2019/3/13 16:19
 **/
public interface ProducerChannel {

    String OUTPUT = "producer-channel-1";

    @Output(OUTPUT)
    MessageChannel output();

}
```

<br />

#### 启动类注册 Eureka-Client， Binder MQ 管道

```java
package com.sestar.userserviceclient;

import com.sestar.userserviceclient.stream.ProducerChannel;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.stream.annotation.EnableBinding;

@SpringBootApplication
@EnableDiscoveryClient // 使用Eureka服务发现方式发现服务提供端
@EnableBinding({ProducerChannel.class}) // 绑定消息生产者接口, Zipkin Stream 需要, Zipkin Http 方式跟踪不需要
public class UserServiceClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserServiceClientApplication.class, args);
    }

}
```

<br />

#### application.properites

```properties
## 应用名称
spring.application.name = user-service-client-application

## 应用服务
server.port = 3721

## Actuator
management.security.enabled = false

## MQ 配置只有 Zipkin Stream 方式需要
## RabbitMQ
### 如果rabbit服务默认配置有修改, 则需要配置rabbitmq的连接属性
spring.rabbitmq.host = 127.0.0.1
spring.rabbitmq.port = 5672
spring.rabbitmq.username = guest
spring.rabbitmq.password = guest

### RabbitMQ, Kafka,ActiveMQ 管道bindings配置, consumer-channel-1:管道名称, replicated-topic:topic名称
spring.cloud.stream.bindings.producer-channel-1.destination = replicated-topic
```

<br />

#### bootstrap.properties

```properties
## 应用名称
spring.application.name = user-service-client-application

##  Eureka
### spring-eureka-server-application注册到 Eureka 服务器中
### ${}都为服务器的配置内容 http://服务器地址:${server.port}/${server.context-path}/eureka
### 如果 bootstrap 有配置 spring.cloud.config.discovery.service-id, 则需要把该配置移到 bootstrap 中
### 因为*.discovery.service-id属性是查找 Eureka 服务配置端, 而查找范围就是eureka.client.service-url.defaultZone
### 需要最先加载 Eureka 注册信息, eureka.client.service-url.defaultZone才能确定范围,
### 而 *.discovery.service-id属性配置在 bootstrap 中, 比 application 配置先加载,
### 所以当有配置Eureka服务配置端, 则需要把 eureka.client.service-url.defaultZone必须配置在 bootstrap 中
eureka.client.serviceUrl.defaultZone = http://127.0.0.1:3701/eureka

## 配置客户端应用关联的应用
## spring.cloud.config.name 是可选的
## 如果没有配置，采用 ${spring.application.name}
spring.cloud.config.name = user-server
## 关联profile
spring.cloud.config.profile = default
## 关联label
spring.cloud.config.label = master
## 激活 Config Server 服务发现和 spring.cloud.config.discovery.serviceId 捆绑配置
spring.cloud.config.discovery.enabled = true
## Config Server 服务器应用名称
spring.cloud.config.discovery.serviceId = config-server-application
```

<br />

#### 创建用以测试 Zipkin Stream 的服务

```java
package com.sestar.userserviceclient.web.controller;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.sestar.userapi.domain.User;
import com.sestar.userserviceclient.stream.ProducerChannel;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.support.GenericMessage;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

/**
 * @description Server Client Controller
 * @author zhangxinxin
 * @date 2019/2/15 21:30
 **/
@RestController
public class UserServiceClientController {

    /**
     * 发送消息 stream 接口定义
     */
    private ProducerChannel producerChannel;

    /**
     * Object -> String 工具类
     */
    private ObjectMapper objectMapper;

    /**
     * producerChannel Bean会通过 @EnableBinding 生成，但是一直爆红，不影响功能
     */
    @SuppressWarnings("all")
    @Autowired
    public UserServiceClientController(ProducerChannel producerChannel, ObjectMapper objectMapper) {
        this.producerChannel = producerChannel;
        this.objectMapper = objectMapper;
    }

    /**
     * 测试 Rabbit Binder 发送消息
     */
    @PostMapping("/rabbit/send/user")
    public boolean sendUserByRibbon(@RequestBody User user) throws JsonProcessingException {
        System.out.println(this.getClass().getName() + "#sendUserByRibbon 生产消息: " + user);
        MessageChannel output1 = producerChannel.output();
        // User 序列化 JSON
        String payload1 = objectMapper.writeValueAsString(user);
        GenericMessage<String> msg = new GenericMessage<>(payload1);
        // 发送消息
        return output1.send(msg);
    }

}
```

<br />

### User-Service-Provider

<br />

#### Maven 依赖

<br />

##### Http 方式

```xml
<!-- Eureka Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>

<!-- Config Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-client</artifactId>
</dependency>

<!-- 下面一个依赖是 Zipkin Http 方式 -->
<!-- Spring Cloud Zipkin -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>
```

<br />

##### Stream 方式

```xml
<!-- 依赖 用户 API -->
<dependency>
    <groupId>com.sestar</groupId>
    <artifactId>user-api</artifactId>
    <version>${project.version}</version>
</dependency>

<!-- Eureka Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>

<!-- Config Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-client</artifactId>
</dependency>

<!-- Spring Cloud Stream Binder Rabbit MQ -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-binder-rabbit</artifactId>
</dependency>

<!-- 下面两个依赖是 Zipkin Stream 方式 并且还要加上一个 MQ -->
<!-- Spring Cloud Sleuth -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>

<!-- Spring Cloud Sleuth Stream -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-sleuth-stream</artifactId>
</dependency>
```

<br />

#### 创建 MQ 订阅管道

```java
package com.sestar.userserverprovider.stream;

import com.sestar.userapi.domain.User;
import org.springframework.cloud.stream.annotation.Input;
import org.springframework.messaging.SubscribableChannel;

/**
 * @description {@link User} 接收消息 stream 接口定义
 * @author zhangxinxin
 * @date 2019/3/12 16:07
 **/
public interface ConsumerChannel {

    String INPUT = "consumer-channel-1";

    // 管道
    @Input(INPUT)
    SubscribableChannel input();

}
```

<br />

#### 启动类注册 Eureka-Client, Binder MQ 管道

```java
package com.sestar.userserverprovider;

import com.sestar.userserverprovider.stream.ConsumerChannel;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.stream.annotation.EnableBinding;

@SpringBootApplication
@EnableDiscoveryClient // 注册为 Eureka-Client
@EnableBinding({ConsumerChannel.class}) // 订阅管道绑定, Zipkin Stream 需要, Zipkin Http 方式跟踪不需要
public class UserServerProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserServerProviderApplication.class, args);
    }

}
```

<br />

#### application.properites

```properties
### 通用配置
## 应用名称
spring.application.name = user-service-provider-application

## 应用服务
server.port = 3741

## Actuator
management.security.enabled = false

## Eureka
### Eureka开关
#eureka.client.enabled = false
### Eureka注册
eureka.client.serviceUrl.defaultZone = http://127.0.0.1:3701/eureka
### 上传注册信息时间间隔
eureka.client.instance-info-replication-interval-seconds = 5
### 获取注册信息时间间隔
eureka.client.registry-fetch-interval-seconds = 5
### Eureka注册名使用IP
eureka.instance.prefer-ip-address = true

## Spring Cloud Binding Stream
### RabbitMQ, Kafka 管道bindings配置, consumer-channel-1:管道名称, replicated-topic:topic名称
#spring.cloud.stream.bindings.consumer-channel-1.destination = replicated-topic
#spring.cloud.stream.bindings.consumer-channel-1.binder = activemq-demo

## RabbitMQ
### 如果rabbit服务默认配置有修改, 则需要配置rabbitmq的连接属性
spring.rabbitmq.host = 127.0.0.1
spring.rabbitmq.port = 5672
spring.rabbitmq.username = guest
spring.rabbitmq.password = guest
```

<br />

#### bootstrap.properties

```properties
## 应用名称
spring.application.name = user-service-provider-application

##  Eureka
### spring-eureka-server-application注册到 Eureka 服务器中
### ${}都为服务器的配置内容 http://服务器地址:${server.port}/${server.context-path}/eureka
### 如果 bootstrap 有配置 spring.cloud.config.discovery.service-id, 则需要把该配置移到 bootstrap 中
### 因为*.discovery.service-id属性是查找 Eureka 服务配置端, 而查找范围就是eureka.client.service-url.defaultZone
### 需要最先加载 Eureka 注册信息, eureka.client.service-url.defaultZone才能确定范围,
### 而 *.discovery.service-id属性配置在 bootstrap 中, 比 application 配置先加载,
### 所以当有配置Eureka服务配置端, 则需要把 eureka.client.service-url.defaultZone必须配置在 bootstrap 中
eureka.client.serviceUrl.defaultZone = http://127.0.0.1:3701/eureka

## 配置客户端应用关联的应用
## spring.cloud.config.name 是可选的
## 如果没有配置，采用 ${spring.application.name}
spring.cloud.config.name = user-server
## 关联profile
spring.cloud.config.profile = default
## 关联label
spring.cloud.config.label = master
## 激活 Config Server 服务发现和 spring.cloud.config.discovery.serviceId 捆绑配置
spring.cloud.config.discovery.enabled = true
## Config Server 服务器应用名称
spring.cloud.config.discovery.serviceId = config-server-application
```

<br />

#### 实现 User-Api 的 Domain 的服务接口 (Zipkin Stream 方式)

```java
package com.sestar.userserverprovider.web.service;

import com.sestar.userapi.api.IUserService;
import com.sestar.userapi.domain.User;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Random;
import java.util.concurrent.ConcurrentHashMap;

/**
 * @description 用户服务
 * @author zhangxinxin
 * @date 2019/2/15 20:42
 **/
@Service("inMemoryUserService")
public class InMemoryUserService implements IUserService {

    /**
     * User缓存
     */
    private Map<Long, User> repository = new ConcurrentHashMap<>();

    /**
     * 随机数工具类
     **/
    private static final Random random = new Random();

    @Override
    public boolean saveUser(User user) {
        return repository.put(user.getId(), user) == null;
    }

    @Override
    public List<User> findAll() {
        return new ArrayList<>(repository.values());
    }

}
```

<br />

#### 用以测试 Zipkin Stream 的消息监听

```java
package com.sestar.userserverprovider.web.service;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.sestar.userapi.api.IUserService;
import com.sestar.userapi.domain.User;
import com.sestar.userserverprovider.stream.ConsumerChannel;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.messaging.SubscribableChannel;
import org.springframework.messaging.support.GenericMessage;
import org.springframework.stereotype.Service;

import javax.annotation.PostConstruct;
import java.io.ByteArrayInputStream;
import java.io.ObjectInputStream;

/**
 * @description 用户消息订阅服务
 * @author zhangxinxin
 * @date 2019/3/12 16:34
 **/
@Service
public class UserMessageService {

    /**
     * 接收消息 stream 接口定义
     */
    private ConsumerChannel consumerChannel;

    /**
     * 用户服务接口
     */
    private IUserService userService;

    /**
     * Object -> String 工具类
     */
    private ObjectMapper objectMapper;

    /**
     * ConsumerChannel Bean会通过 @EnableBinding 生成，但是一直爆红，使用SuppressWarnings去除错误
     */
    @SuppressWarnings("all")
    @Autowired
    public UserMessageService(ConsumerChannel consumerChannel, @Qualifier("inMemoryUserService") IUserService userService,
                              ObjectMapper objectMapper) {
        this.consumerChannel = consumerChannel;
        this.userService = userService;
        this.objectMapper = objectMapper;
    }

    /**
     * @description @StreamListener 方式接收消息, 和@ServiceActivator, SubscribableChannel功能一样,
     *                  应用场景: Spring-Cloud
     * @author zhangxinxin
     * @date 2019/3/12 16:59
     */
    @StreamListener(ConsumerChannel.INPUT)
    public void onMessage(String data) {
        System.out.println(this.getClass().getName() + "#onMessage -> Subscribe by @StreamListener");
        saveUser(data);
    }

    /**
     * @description @ServiceActivator 方式接收消息, 和@StreamListener, SubscribableChannel功能一样,
     *                  应用场景: integration
     * @author zhangxinxin
     * @date 2019/3/12 16:59
     */
    @ServiceActivator(inputChannel = ConsumerChannel.INPUT)
    public void listen(String data) {
        System.out.println(this.getClass().getName() + "#listen -> Subscribe by @ServiceActivator");
        saveUser(data);
    }

    /**
     * @description SubscribableChannel 管道方式接收消息, 和@ServiceActivator, SubscribableChannel功能一样,
     *                  应用场景: messaging
     * @author zhangxinxin
     * @date 2019/3/12 16:59
     */
    @PostConstruct
    public void init() {
        // 获取管道
        SubscribableChannel channel = consumerChannel.input();
        channel.subscribe(message -> {
            System.out.println(this.getClass().getName() + "#init -> Subscribe by SubscribableChannel");
            String contentType = message.getHeaders().get("contentType", String.class);
            if ("text/plain".equals(contentType)) {
                saveUser((String) message.getPayload());
            } else {
                saveUser((byte[]) message.getPayload());
            }
        });
    }

    /**
     * @description 处理MQ字节流数据
     * @author zhangxinxin
     * @date 2019/3/12 16:57
     * @param data 管道字节消息
     */
    private void saveUser(byte[] data) {
        try {
            // 将字节流数据封装成对象
            ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(data);
            ObjectInputStream objectInputStream = new ObjectInputStream(byteArrayInputStream);
            // 反序列化 User 对象
            User user = (User) objectInputStream.readObject();
            System.out.println(this.getClass().getName() + "#saveUser -> 订阅消息内容如下: " + user);
            // 调用服务
            userService.saveUser(user);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    /**
     * @description 处理MQ字符串数据
     * @author zhangxinxin
     * @date 2019/3/12 16:57
     * @param data 管道字节消息
     */
    private void saveUser(String data) {
        try {
            User user = objectMapper.readValue(data, User.class);
            System.out.println(this.getClass().getName() + "#saveUser -> 订阅消息内容如下: " + user);
            userService.saveUser(user);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

}
```

<br />

### 测试 Zipkin 应用跟踪

> Http 方式

```text
访问 url: http://localhost:3721/env 或者 user-service-client-application 的其他mapper
访问 url: http://localhost:3721/env 或者 user-service-provider-application 的其他mapper

访问 url: http://localhost:3751 或者 http://localhost:3751/zipkin/ 查看应用跟踪内容
```

<img width="30%" src="/note/_v_images/java/框架/SpringCloud/Zipkin-Http-1.png">

<br />

> Stream 方式

```text
POST 访问 url: http://localhost:3721/rabbit/send/user  入参: {"id": 1,	"name": "sestar",	"desc": "haha"}

访问 url: http://localhost:3751 或者 http://localhost:3751/zipkin/ 查看应用跟踪内容
```

<img width="30%" src="/note/_v_images/java/框架/SpringCloud/Zipkin-Http-1.png">




