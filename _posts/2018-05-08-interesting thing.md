---
    layout: post
　　 title: something interesting
---

### 1. fastJson 序列化问题

- 问题描述及原因：

  ```java
  public class Result<T> implements Serializable {
      public boolean isSuccess() {
      return String.valueOf(ResultCode.SUCCESS.getCode()).equals(code);
   }
  }

  Result<Object> result = Result.success(new Object());
      System.out.println(JSON.toJSONString(result));

  ```

- 这个 result 的结果：

  ```json
  { "code": "0", "data": {}, "message": "成功", "success": true }
  ```

  这边会多一个 success key，但是没有这个成员变量的，debug fastJson 源码,发现序列化解析的时候会根据 isXxx 这种方法形式 

  ```java
  @link com.alibaba.fastjson.util.TypeUtils;

   if (methodName.startsWith("is")) {
      Field field = ParserConfig.getField(clazz, propertyName);
   }
  ```

- 延伸：看 json 序列化，看到了 asm 的应用

        [liangfei 的一篇动态代理方案的性能对比](http://javatar.iteye.com/blog/814426)

### 2. DataGrip 配置文件问题 

- 之前鼓捣过 datagrip 的主题，安装后发现太丑了，就删除了，然后悲剧发现窗口边框黑了，看起来超变扭。
- - 配置文件的目录:/Users/xuyongjian/Library/Preferences/DataGrip2018.1

  - 之前  删除过一次 colors/，这个只是主题颜色的配置，删除完 /options 后重启就恢复了，

### 3. 新开的项目编译不通过

- 问题描述：编译报这个鬼错，初步定位是 lombok 跟 jdk 版本不兼容的问题

  ```java
  atal error compiling: java.lang.ExceptionInInitializerError: com.sun.tools.javac.code.TypeTags -> [Help 1]
  ```

  ![lombok异常信息](/images/lombok_e.jpg)

* 问题解决：当我在 iterm 中  查看 jdk 版本是 jdk1.8，然后又设置  成 jdk1.7， 发现还是 compile 失败，崩溃了。。。纠结了好几个小时，然后我无意中在 idea 自带的 terminal 中 java -version 才发现居然是 jdk10.这时候才发现是环境变量未加载的原因。重新在 idea 终端中执行`source ~/.bash_profile` 即可。

* 事后： 当一次 jdk 更新时，发现 oracle 的更新程序会  自动修改 JAVA_HOME 这个环境变量为最新的版本，需重新 source 用户的配置文件

### 4. 抽象工厂模式 和 工厂方法 区别

- 抽象工厂的主体是工厂，需定义一个 factory 接口，这个接口有抽象的 product 方法，不同的工厂为有相同的产品线，但具体产品是带牌子的
- 工厂方法的主体是 product 方法，不同的工厂类实现不同的生产方法
- 进一步的理解，抽象工厂是对工厂方法 再加了一层封装，工厂方法一个产品的不同形态，抽象工厂就是不同产品的不同形态

### 5. 内部类 json 序列化失败问题

```java
JSON.parseArray(gradeConfigJson,CustomzerGradeConfig.class);
```

当 CustomzerGradeConfig 是普通内部类时，会抛出异常信息

```java
Caused by: java.lang.IllegalArgumentException: argument type mismatch
	at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
	at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
	at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
	at com.alibaba.fastjson.parser.deserializer.JavaBeanDeserializer.createInstance(JavaBeanDeserializer.java:113)
	... 7 more
```

- 原因及解决
  - 普通内部类默认  持有外部类的一个引用，但在外部类的 static 方法中进行 json 序列化为内部类， 内部类  实  例化时  获取不到外部类引用，实例化失败，就 gg 了
  - 内部类设置为静态内部类，static

### 6.java lambda

- 可变长参数为 5，第 6 个 lambda invokeDynamic 推断不出类型
- vavr Api.Match.of

    ```java
     @SafeVarargs
            public final <R> R of(Case<? extends T, ? extends R>... cases) {
                Objects.requireNonNull(cases, "cases is null");
                for (Case<? extends T, ? extends R> _case : cases) {
                    final Case<T, R> __case = (Case<T, R>) _case;
                    if (__case.isDefinedAt(value)) {
                        return __case.apply(value);
                    }
                }
                throw new MatchError(value);
            }
    ```

    ```java
    R r= Match(T).of(
                    Case($(T1), R1),
                    Case($(T1), R1),
                    Case($(T1), R1),
                    Case($(T1), R1),
                    Case($(T1), R1),
                    Case($(), () -> {
                        throw new RuntimeException("推断类型不符合下限");
                    })
            );
    ```

- 问题猜测：可变长参数是java语法糖，会将参数包装成参数数组，初始化的数组大小为5或者6，瞎猜

- 解决方法：对不确定的参数类型进行强转
  ```java
  Case($(), (Supplier<R>)() -> {
                  throw new RuntimeException("推断类型不符合下限");
              })
  ```

### 7.tomcat7与一些jar不兼容问题
- 问题描述：升级了jackson（2.10.2） and lombok（1.16.22）的依赖，tomcat启动报错
```java
SEVERE: Unable to process Jar entry [module-info.class] from Jar [jar:file:/home/ewallet/risk-feature/risk-feature-
devjd/default/webroot/WEB-INF/lib/lombok-1.16.22.jar!/] for annotations
org.apache.tomcat.util.bcel.classfile.ClassFormatException: Invalid byte tag in constant pool: 19
        at org.apache.tomcat.util.bcel.classfile.Constant.readConstant(Constant.java:136)
```

- 原因：tomcat7会用 bcel这个字节码包去scan 加载的jars，然后有的jar中会有 module-info.class（java9模块化使用），readConstant不兼容
  * 使用JDK9创建模块打包后有一个module-info.class的类其中常量结构可能有19 - CONSTANT_Module and  20 - CONSTANT_Package两种
  
  * [tomcat bug链接](https://bz.apache.org/bugzilla/show_bug.cgi?id=60688)，tomcat8.5以后解决了

  * tomcat scan这些jars的作用，可以看 [how to](https://cwiki.apache.org/confluence/display/TOMCAT/HowTo+FasterStartUp),除了tomcat自身jar，主要是为了扫第三方包中tomcat相关Annotations

- 解决方法：
  * 升级tomcat到8.5及以后

  * 在 *tomcat/conf/catalina.properties* 的配置中，追加tomcat.util.scan.DefaultJarScanner.jarsToSkip的不扫描jar


### 8. java8 Collectors.toMap会npe
- [so连接](https://stackoverflow.com/questions/24630963/java-8-nullpointerexception-in-collectors-tomap)




### 时区的时间解析
1. joda的DateTimeFormat.forPattern 时区只支持Z格式；+0800 +08:00的字符串输入都可以
2. java本身 SimpleDateFormat/DateTimeFormatter 时区Z和XXX都支持；但字符串输入 Z:+0800, XXX:+08:00  有对应的格式要求
3. 具体Z和XXX的区别，见SimpleDateFormat jdk的doc注释