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

ConfigFileApplicationListener.java负责读取配置文件以及profile，该类有内置Loader私有类, 这个私有类负责读取文件内容。  
Loader私有类属性有PropertiesLoader实例对象, PropertyLoader有loaders数组(PropertyLoader接口实现对象),   
&emsp;PropertyLoader接口有两个实现类:  
&emsp;&emsp;1. PropertiesPropertySourcesLoader(负责读取properties和xml文件),  
&emsp;&emsp;2. YamlPropertySourcesLoader(负责读取yaml和yml文件)

</br>

## 随笔

* Application启动类的args是SpringBoot的配置内容(IDEA的Edit Configuration...), 比如: vm options, program arguments
</br>