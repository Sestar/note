# Maven
</br>

## 修改Maven项目的项目名

### 文字描述
1. 修改pom.xml
```xml
<groupId>com.sestar</groupId>
<!-- 修改为项目名 -->
<artifactId>spring-cloud-config-server</artifactId>
<version>0.0.1-SNAPSHOT</version>
<packaging>jar</packaging>

<!-- 修改为项目名 -->
<name>spring-cloud-config-server</name>
<description>Demo project for Spring Boot And Spring Cloud</description>
```

2. 修改包名

```text
将项目文件改成想要的项目名, 引导类和Source Folders下包名也要修改项目名
```

3. 删除 `.idea` 文件夹，重新打开项目即可

### 图例

<img src="/note/_v_images/code_sofa/maven/修改Maven项目名.png" width="100%"/>
</br>

## 配置Maven内存

```text
maven设置内存配置:
    默认情况下, maven内存和jvm一样, 大项目情况下maven需要内存一般很大, 可以自己配置:
    环境变量添加MAVEN_OPTS: -Xms128m -Xmx512m
```
</br>

## Maven版本

pom.xml文件的modelVersion在使用maven2，maven3都为4

pom.xml配置Java支持1.5版本以上
```xml
  <project>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>1.5</source>
                    <target>1.5</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
  </project>
```
</br>

## main方法打包

默认打包的jar是不能直接运行的, 即带有main方法的类信息不会添加到mainifest中(打开jar文件中的META-INF/MAINIFESE.MF文件, 将无法看到Main-Class一行),
为了生成可执行的jar文件, 需要借助maven-shade-plugin(\<project>\<build>\<plugins>), 可以打包两个jar, 一个正常, 一个前缀origin(可执行的jar包)

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <!-- 版本可视使用情况更改 -->
    <version>1.2.1</version>
    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>shade</goal>
            </goals>
            <configuration>
                <transformers>
                    <transeformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                        <mainClass>自己个的main方法的包名+类名</mainClass>
                    </transeformer>
                </transformers>
            </configuration>
        </execution>
    </executions>
</plugin>
```
</br>

## 修改setting.xml配置

> <localRepository>:本地仓库位置，默认为 ~/.m2/repository(~为电脑当前用户文档)
> <interactiveMode>:交互模式是否打开, 交互模式会让用户填写版本信息(手动创建Maven项目mvn archetype:generate)，非交互模式会使用默认值(交互模式默认打开)
> <offline>:表示在Maven进行项目编译和部署等操作时是否允许Maven进行联网来下载所需要的信息
> <pluginGroup>:


</br>

## 手动创建Maven项目

> <span style="color:orange">1. mvn compile:</span><span style="color: #ABB2BF">编译项目, 生成target文件, 包含classes文件夹</span>(所有class文件和resource文件)<span style="color: #ABB2BF">, maven-status</span>(编译项目的内容列表, 比如class列表)  
> <span style="color:orange">2. mvn clean:</span><span style="color: #ABB2BF">清除项目,  删除target文件</span>  
> <span style="color:orange">3. mvn clean test:</span><span style="color: #ABB2BF">先执行compile, 编译测试内容, 生成target文件中的test-classes文件</span>(所有测试类class文件)  
> <span style="color:orange">4. mvn cobertura:cobertura:</span><span style="color: #ABB2BF">对测试内容进行覆盖统计, 生成target文件中的cobertura, generated-classes</span>(cobertura的配置, 项目classes文件, resource文件)<span style="color: #ABB2BF">, site</span>(项目的所有类覆盖报告)   
> <span style="color:orange">5. mvn clean package:</span><span style="color: #ABB2BF">先执行test, 再打包项目到target中</span>(多了一个文件夹maven-archiver(pom文件配置))  
> <span style="color:orange">6. mvn clean install:</span><span style="color: #ABB2BF">先执行package, 就是将项目提交到maven的repository</span>

</br>

## pom配置

<span style="color:orange">mvn compile提示编码错误 </span>  
    Using platform encoding (GBK actually) to copy filtered resources, i.e. build is platform dependent!
编码错误, 需要项目使用utf8编码格式来编译, 在pom添加

```xml
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
</properties>
```

&nbsp;&nbsp;  
<span style="color:orange">单元测试覆盖率</span>

```xml
<!-- 单元测试工具, cobertura使用下方有具体内容 -->
<reporting>
    <plugins>
        <plugin>
            <groupId>org.codehaus.mojo</groupId>
            <artifactId>cobertura-maven-plugin</artifactId>
        </plugin>
    </plugins>
</reporting>
```
</br>

## cobertura(Maven-test插件)

instructions | describe
---|---
mvn cobertura:help  |   查看cobertura插件的帮助
mvn cobertura:clean    |     清空cobertura插件运行结果
mvn cobertura:check    |     运行cobertura的检查任务
mvn cobertura:cobertura  |   运行cobertura的检查任务并生成报表，报表生成在target/site/cobertura目录下
cobertura:dump-datafile   |  Cobertura Datafile Dump Mojo
mvn cobertura:instrument  |  Instrument the compiled classes

在target文件夹下出现了一个site目录，下面是一个静态站点，里面就是单元测试的覆盖率报告。  
具体学习可以参考：https://www.cnblogs.com/qyf404/archive/2015/12/12/5040593.html
</br>

## 随笔

maven升级会重置文件, 比如conf/setting.xml