# Unknow

## ThreadLocal和Synchronized区别

## java的原子性，现阶段java的最小单位

## ~~枚举可以防止反序列化，不会破坏单例模式~~，反序列化实现流程, readObject(), readResolve(), writeObject  
  -- 序列化
      没有serialVersionUID, jvm会根据类属性自动生成UID, 这种情况下, 如果该类属性修改, 反序列化将会失败InvalidClassException(查询不到对应的class);
      但是自设UID, 不会出现InvaildClassException(UID唯一, 必须显示申明: private Long serialVersionUID = 1L;)
      对于static属性, transient修饰的属性将不会进入序列化

  -- 序列化文件(具体: [序列化文件说明](https://www.cnblogs.com/senlinyang/p/8204752.html))  
      1.序列化文件头: ACED(序列化协议头), 00005(版本), 73(表明序列化为对象)  
      2.序列化类描述: 72(新的class), 0018(类名长度为24), **(24个字节, 对应class名称), * *(8个字节UID)  
      3.属性描述: private(), Long(), attrName()  
      4.父类描述: 和2相同  
      5.具体值  

  -- 反序列化(具体过程没有详细描述)
      实例对象 -> readObject -> readReslove -> validateObject -> garbageFinalize

* ~~<span style="color:orange">访问权限派生类比基类大~~  
  -- 调用父类对象时(其实是子类上转), 如果子类的方法的权限比父类小, 会发生访问错误异常

## Proxy.newInstance实现  [参考](https://www.cnblogs.com/kundeg/p/7942030.html)

## cglib实现, Enhancer.create(class, proxyDemoInst)

## 原子对象

* ~~<span style="color:orange">SQL的 IN 和 EXIST 效率比较~~  
 SELECT * FROM A WHERE id IN (SELECT * FROM B WHERE A.id = B.id);  
 SELECT * FROM A WHERE EXISTS (SELECT * FROM B WHERE A.id = B.id);  
 SELECT * FROM A WHERE id NOT IN (SELECT * FROM B);  
 SELECT * FROM A WHERE NOT EXISTS (SELECT * FROM B);  

 -- EXISTS 过程: A的数据遍历, 如果B表查询到就加入结果集, 否则舍弃(NOT EXISTS相反, 但是索引都是B表)  
 -- IN 过程: B的数据遍历在A表查询(索引是A表)  
 -- NOT IN 过程: A遍历数据在B表中继续遍历, 无法使用索引  

总结: 表数据大的使用索引会大大减少查询时间，即当A表数据量大, 使用IN, 反之使用EXISTS, 当然两个表数据量一样时, 两个方式没有差别;
    但是NOT EXISTS和NOT IN不一样, NOT EXISTS还是使用B表索引增强效率, NOT IN无法使用索引，NOT EXISTS总是比NOT IN效率高。

## 检索Map: ConcurrentHashMap结构

## com.github.ltsopensource.json.JSONObject.parseObject()

## Aho-Corasick算法(敏感词算法)

## WeakCache

## RestTemplate(ResponseExtractor<String> responseExtractor = response -> { 是什么鬼，getPaasRestTemplate为什么需要LinkMap)

```java
public static String readOfByte(String fileId) {
    String url = BYTE_URL + "{1}";
    // 设头
    RequestCallback requestCallback = request -> request.getHeaders()
            .setAccept(Arrays.asList(MediaType.APPLICATION_OCTET_STREAM, MediaType.ALL));
    // 流式处理文件
    ResponseExtractor<String> responseExtractor = response -> {
        InputStream InputStream = response.getBody();
        String line = null;
        StringBuilder sb = new StringBuilder();
        BufferedReader reader = new BufferedReader(new InputStreamReader(InputStream));
        while ((line = reader.readLine()) != null) {
            sb.append(line);
        }
        return sb.toString();
    };
    return getPaasRestTemplate().execute(url, HttpMethod.GET, requestCallback, responseExtractor, fileId);
}
```
```java
public static String uploadOfFile(File file, String center, String fileType) {
    if (file.length() <= 0) {
        throw ErrorCodeHelper.PARAM_ERROR_CODE(FILE_IS_EMPTY.getCode(), FILE_IS_EMPTY.getMsg()).toBaseException();
    }
    String url = FILE_URL + center + "/" + fileType;
    // 封装file, ContentType
    FileSystemResource resource = new FileSystemResource(file);
    MultiValueMap<String, Object> param = new LinkedMultiValueMap<>();
    param.add("file", resource);
    String res = "";
    HttpHeaders headers = new HttpHeaders();
    MediaType type = MediaType.parseMediaType("multipart/form-data; charset=UTF-8");
    headers.setContentType(type);
    HttpEntity<MultiValueMap<String, Object>> entity = new HttpEntity<>(param, headers);
    ResponseEntity<String> fileCall;
    // 增加resttemplate发生异常的处理
    try {
        fileCall = getPaasRestTemplate().exchange(url, HttpMethod.POST, entity, String.class);
    } catch (Exception e) {
        PaasLogger.error(e, "uploadOfFile 请求路径：{} 发生异常，异常信息：{}", url, e.getMessage());
        throw e;
    }
    res = fileCall.getBody();
    return res;
}
```
</br>

## @Bean 中 autowire属性作用
</br>

## BeanPostProcessor作用到底是什么?主要是AutowiredAnnotationBeanPostProcessor，Required..,Common..,Persistence..
(即使注册多个处理器,Spring仍然只会处理一次)
</br>

## <context:annotation-config />和<context:component-scan />区别
</br>

## bean实例化

```text
@Bean注解有两个,applicationContext.getBean("")获取其中一个bean,但是会执行另个bean的init方法.说明定义applicationContext时
 两个bean已经实例化了。到底用@Bean注解的bean什么时候实例化(详情：imooc/spring项目中test/bean/ImportResource/BeanImportResourceTest)
```
</br>

## @Scope(value="", proxyMode="") value和proxyMode每个取值区别
</br>

## execution
```text
execution(* bean.aop.advice.*.*(..))和execution(* bean.aop.advice...(..))要实现的功能一样,但是后面的表达式编译通不过
execution(* invoke(..))就不会执行before方法
execution(* path.class(..))匹配的是无参的方法,要匹配有参的改成execution(* path.class(参数类型)) and args(参数名称)这样做的原因,为什么还要有args()
(详情：imooc/spring项目中test/bean/aop/advice)
```
</br>


## ProxyFactoryBean生成对象,jdk动态代理,CGLIB代理工作原理
</br>


## before在around-start后面,after不是最后执行吗,为什么after-returning还在其后(详情：imooc/spring项目中test/bean/aop/aspectj)
</br>


## 事务的隔离级别,事务传播行为的具体原理
</br>


## ssh框架action由Spring管理为什么action的bean的scope是prototype
</br>


## @Cahce相关注解, cacheNames, key用处
</br>


## spel表达式, 动态赋值, 缓存的key用传递来的商家标识动态赋值
</br>


## 加密
```text
密码秘钥要分为加密和解密, 解密秘钥和加密秘钥有关联吗, 如果没有关联, 解密秘钥为什么能够对加密内容进行解密(
	对称密码: 即加密和解密密钥相同。但是也有非对称密码
)
```
</br>


## lambad stream 传参, 比如字符串必须是没有改动过的, final
</br>

* ~~<span style="color:orange">localhost和127的区别~~

```text
localhot是不经网卡传输！这点很重要，它不受网络防火墙和网卡相关的的限制。
访问localhost也不会解析成ip，不会占用网卡、网络资源。

而127.0.0.1是需要通过网卡传输，依赖网卡，并受到网络防火墙和网卡相关的限制。

这就是为什么有时候用localhost可以访问，但用127.0.0.1就不可以的情况。
```
</br>

## Gossip协议(https://www.cnblogs.com/williamjie/p/9369795.html)