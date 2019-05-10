# SpringBoot
</br>

## SpringBoot 加载顺序

### springboot监听器发生顺序(SpringFactoriesLoader.java -> spring.factories)

```
    PropertySource Loaders(数据源文件) -> SpringApplicationRunListener(项目App运行) ->
    ApplicationContextInitializer(项目上下文初始化) -> ApplicationListener(包含项目加载数据源文件并配置) ->
    ...
```

### springboot事件发生顺序(SpringApplication#run)

```
    ApplicationStartingEnvent(项目开始事件) -> ApplicationEnvironmentPreparedEnvent(项目环境准备阶段) ->
    ApplicationPreparedEvent(项目准备阶段)  -> ApplicationReadyEnvent/ApplicationFailedEvnet(项目启动成功或者失败事件)
```
</br>


## Banner

Banner就是SpringBoot启动时, 控制台出现的`Spring`图案, 详情查看SpringApplicationBannerPrinter.java

</br>

### 配置Banner文件路径

* 如果没有配置相关Banner, 并且没有resources/banner.*文件，则默认通过SpringBootBanner.java生成`'Spring'`图案。  
</br>

* 如果想修改Banner, 查看SpringApplicationBannerPrinter.java。  

    * 修改Banner文件路径

```yml
banner:
  location: config/banner.txt  # banner文件定位
```

*  * Banner处理类

```text
下面是SpringApplicationBannerPrinter的常量，可以看出Banner的文件形式有两种。  
  一种是文本形式，默认路径是resources/banner.txt，使用定位banner.location, 重新定位banner文件
  一种是图片形式,  默认路径是resources/banner.*, 使用banner.image.location(图片必须是gif,jpg,png)重新定位banner图片
  如果有多个Banner文件, 则每个Banner都会打印(Image文件先加载, Text文件后加载)
```

```Java
    static final String BANNER_LOCATION_PROPERTY = "banner.location";

    static final String BANNER_IMAGE_LOCATION_PROPERTY = "banner.image.location";

    static final String DEFAULT_BANNER_LOCATION = "banner.txt";

    static final String[] IMAGE_EXTENSION = { "gif", "jpg", "png" };
```
</br>

### 关闭Banner

在Application的main方法中添加 application.setBannerMode(Banner.Mode.OFF)

```Java
import org.springframework.boot.Banner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SpringCloudFirstLearnApplication {

	public static void main(String[] args) {
		SpringApplication application = new SpringApplication(SpringCloudFirstLearnApplication.class);
		application.setBannerMode(Banner.Mode.OFF);
		application.run(args);
	}
}
```
</br>

### Text形式Banner图案案例

```text
 ${AnsiColor.BRIGHT_RED}_ooOoo_${AnsiColor.BRIGHT_YELLOW}
                  ${AnsiColor.BRIGHT_RED}o8888888o${AnsiColor.BRIGHT_YELLOW}
                  ${AnsiColor.BRIGHT_RED}88${AnsiColor.BRIGHT_YELLOW}" . "${AnsiColor.BRIGHT_RED}88${AnsiColor.BRIGHT_YELLOW}
                  (| -_- |)
                  O\  =  /O
               ____/`---'\____
             .'  \\|     |//  `.
            /  \\|||  :  |||//  \
           /  _||||| -:- |||||-  \
           |   | \\\  -  /// |   |
           | \_|  ''\---/''  |   |
           \  .-\__  `-`  ___/-. /
         ___`. .'  /--.--\  `. . __
      ."" '<  `.___\_<|>_/___.'  >'"".
     | | :  `- \`.;`\ _ /`;.`/ - ` : | |
     \  \ `-.   \_ __\ /__ _/   .-` /  /
======`-.____`-.___\_____/___.-`____.-'======
                   `=---='
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
         佛祖保佑       永无BUG
```
</br>

## SpringApplication#run
</br>

### Spring监听事件步骤
* 需要上下文对象ApplicatonContext
* 需要特定事件监听器对象
* 上下文增加特定事件(则上下文会自动监听事件)
</br>

### 源码

```Java
public ConfigurableApplicationContext run(String... args) {
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    ConfigurableApplicationContext context = null;
    FailureAnalyzers analyzers = null;
    configureHeadlessProperty();
    // 获取CommandLineArgs, 即Program args
    SpringApplicationRunListeners listeners = getRunListeners(args);
    // 监听器开始监听事件
    listeners.starting();
    try {
        // 将program args作为环境配置注入环境中
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        // 环境准备
        ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
        // 打印Banner
        Banner printedBanner = printBanner(environment);
        // 初始化ApplicationContext
        context = createApplicationContext();
        analyzers = new FailureAnalyzers(context);
        // ApplicationContext增加监听器
        prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        // 启动ApplicationContext
        refreshContext(context);
        afterRefresh(context, applicationArguments);
        listeners.finished(context, null);
        stopWatch.stop();
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
        }
    	return context;
    }
    catch (Throwable ex) {
        handleRunFailure(context, listeners, analyzers, ex);
        throw new IllegalStateException(ex);
    }
}
```
</br>

## Spring配置文件读取

<img width="100%" src="/note/_v_images/java/框架/SpringBoot/PropertySourceLoader.png" />

```text
ConfigFileApplicationListener.java负责读取配置文件以及profile，该类有内置Loader私有类, 这个私有类负责读取文件内容。
Loader私有类属性有PropertiesLoader实例对象, PropertyLoader有loaders数组(PropertyLoader接口实现对象),
PropertyLoader接口有两个实现类:
    1. PropertiesPropertySourcesLoader(负责读取properties和xml文件),
    2. YamlPropertySourcesLoader(负责读取yaml和yml文件)
```

</br>

## Spring Web MediaType

<br />

### HttpMessageConverter

```text
HttpMessageConverter: 处理Request和Response的数据格式

启动 Spring Boot, 加载 HttpMessageConverter 将会加载 HttpMessageConvertersAutoConfiguration 配置类
HttpMessageConvertersAutoConfiguration > JacksonHttpMessageConvertersConfiguration > JacksonHttpMessageConvertersConfiguration

一般 Spring Boot 项目只有三种 HttpMessageConverter 配置类:
1. HttpMessageConvertersAutoConfiguration
2. HypermediaHttpMessageConverterConfiguration -> application/octet-stream
3. StringHttpMessageConverterConfiguration -> text/plain

如果有依赖 jackson-databind(spring-boot-starter-web 都会引用), 将会增加一个 HttpMessageConverter 配置类:
MappingJackson2HttpMessageConverterConfiguration -> application/json

如果不仅有 jackson-databing, 还有 jackson-dataformat-xml, 还会再增加一个 HttpMessageConverter 配置类:
MappingJackson2XmlHttpMessageConverterConfiguration -> application/xml
```

<br />

### ResponseBody

```text
@ResponseBody 注释将会解析返回内容

Spring Boot Starter Web 支持的是将返回内容封装成 ModelAndView:

源码定位: AnnotationMethodHandlerAdapter$ServletHandlerMethodInvoker#getModelAndView
将会判断 返回值是否是 HttpEntity 数据, 这种数据对象一般是使用 Template 代理获取服务数据, 和本节内容无关不考虑
然后会判断请求的接口方法 handlerMethod 是否有 @ResponseBody 注释。
有 @ResponseBody 注释则会根据 returnValue 的类型确定 HttpMessageConverter：
    String -> StringHttpMessageConverter
    Object -> org.springframework.boot.autoconfigure.web.JacksonHttpMessageConvertersConfiguration(配置)
        MappingJackson2HttpMessageConverter(Object -> json)
        MappingJackson2XmlHttpMessageConverterConfiguration(Object -> json)

Object 转换对象类型判断:

    1. 仅依赖 jackson-databind, jackson-databind-2.8.10.jar!\META-INF\services\com.fasterxml.jackson.core.ObjectCodec
    默认会加载 ObjectMapper. 通过 JacksonHttpMessageConvertersConfiguration 配置类可以看出会自动加载
    MappingJackson2HttpMessageConverterConfiguration 配置, 即项目会增加 MappingJackson2HttpMessageConverter 这个 HttpMessageConverter.
    则 @ResponseBody 默认返回的是 MappingJackson2HttpMessageConverter 的转化内容, 而这个转化是将 Object -> json

    2. 但是还有依赖 jackson-dataformat-xml, jackson-dataformat-xml-2.8.10.jar!\META-INF\services\com.fasterxml.jackson.core.ObjectCodec
    也会默认加载 XmlMapper. 通过 JacksonHttpMessageConvertersConfiguration 配置类可以看出会自动加载
    MappingJackson2XmlHttpMessageConverterConfiguration 配置,即项目会增加 MappingJackson2XmlHttpMessageConverter 这个 HttpMessageConverter
    则此时既有 MappingJackson2HttpMessageConverter, 也有 MappingJackson2XmlHttpMessageConverter.
    那么又怎么确定返回什么类型的数据呢？是 Json 还是 Xml 呢?

    2.1 AnnotationMethodHandlerAdapter$ServletHandlerMethodInvoker#getModelAndView 处理 @ResponseBody 时会先处理 Accept(返回数据格式),
      AnnotationMethodHandlerAdapter$ServletHandlerMethodInvoker#writeWithMessageConverters 会对接收格式进行排序(
      MimeType$SpecificityComparator), 对所有的HttpMessageConverters 按顺序判断是否符合要求. 而 xml 格式的优先级比 json 高, 所以如果这两种都
      支持的情况下, 会优先转化为 application/xml
    2.2 @RequestMapping 指定返回的 MediaType(数据格式): (@RequestMapping(value = "/*", produces = MediaType.APPLICATION_XML_VALUE))
```

<br />

## 随笔

* Application启动类的args是SpringBoot的配置内容(IDEA的Edit Configuration...), 比如: vm options, program arguments
