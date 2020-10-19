# 关于使用byte-buddy和 javaagent 出现的NoClassDefFoundError，idea不报错，控制台和shell报错

大家好，我是烤鸭：

​		今天分享 -javaagent 遇到的 NoClassDefFoundError。推荐一篇比较好的排错文章。

​		https://www.jianshu.com/p/a36a35b66fab



## 报错日志

首先看一下报错信息：

```
[2020-08-28 15:16:19:741] [main] [WARN ]  [traceId:] [org.springframework.boot.SpringApplication:808] - Unable to close ApplicationContext
org.springframework.beans.factory.CannotLoadBeanClassException: Error loading class [com.xxx.holmes.agent.dependencies.com.alibaba.fastjson.support.spring.JSONPResponseBodyAdvice] for bean with name 'JSONPResponseBodyAdvice' defined in URL [jar:file:/D:/dev/workspace/holmes-sniffer/packages/holmes-agent.jar!/com/xxx/holmes/agent/dependencies/com/alibaba/fastjson/support/spring/JSONPResponseBodyAdvice.class]: problem with class file or dependent class; nested exception is java.lang.NoClassDefFoundError: org/springframework/web/servlet/mvc/method/annotation/ResponseBodyAdvice
        at org.springframework.beans.factory.support.AbstractBeanFactory.resolveBeanClass(AbstractBeanFactory.java:1480)
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.determineTargetType(AbstractAutowireCapableBeanFactory.java:682)
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.predictBeanType(AbstractAutowireCapableBeanFactory.java:649)
        at org.springframework.beans.factory.support.AbstractBeanFactory.isFactoryBean(AbstractBeanFactory.java:1605)
        at org.springframework.beans.factory.support.DefaultListableBeanFactory.doGetBeanNamesForType(DefaultListableBeanFactory.java:523)
        at org.springframework.beans.factory.support.DefaultListableBeanFactory.getBeanNamesForType(DefaultListableBeanFactory.java:494)
        at org.springframework.beans.factory.support.DefaultListableBeanFactory.getBeansOfType(DefaultListableBeanFactory.java:616)
        at org.springframework.beans.factory.support.DefaultListableBeanFactory.getBeansOfType(DefaultListableBeanFactory.java:608)
        at org.springframework.context.support.AbstractApplicationContext.getBeansOfType(AbstractApplicationContext.java:1242)
        at org.springframework.boot.SpringApplication.getExitCodeFromMappedException(SpringApplication.java:869)
        at org.springframework.boot.SpringApplication.getExitCodeFromException(SpringApplication.java:857)
        at org.springframework.boot.SpringApplication.handleExitCode(SpringApplication.java:844)
        at org.springframework.boot.SpringApplication.handleRunFailure(SpringApplication.java:795)
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:325)
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:1226)
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:1215)
        at com.xxx.ballast.TransferApplication.main(TransferApplication.java:17)
        at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.base/java.lang.reflect.Method.invoke(Method.java:566)
        at org.springframework.boot.loader.MainMethodRunner.run(MainMethodRunner.java:49)
        at org.springframework.boot.loader.Launcher.launch(Launcher.java:109)
        at org.springframework.boot.loader.Launcher.launch(Launcher.java:58)
        at org.springframework.boot.loader.JarLauncher.main(JarLauncher.java:88)
Caused by: java.lang.NoClassDefFoundError: org/springframework/web/servlet/mvc/method/annotation/ResponseBodyAdvice
        at java.base/java.lang.ClassLoader.defineClass1(Native Method)
        at java.base/java.lang.ClassLoader.defineClass(ClassLoader.java:1016)
        at java.base/java.security.SecureClassLoader.defineClass(SecureClassLoader.java:174)
        at java.base/jdk.internal.loader.BuiltinClassLoader.defineClass(BuiltinClassLoader.java:801)
        at java.base/jdk.internal.loader.BuiltinClassLoader.findClassOnClassPathOrNull(BuiltinClassLoader.java:699)
        at java.base/jdk.internal.loader.BuiltinClassLoader.loadClassOrNull(BuiltinClassLoader.java:622)
        at java.base/jdk.internal.loader.BuiltinClassLoader.loadClass(BuiltinClassLoader.java:580)
        at java.base/jdk.internal.loader.ClassLoaders$AppClassLoader.loadClass(ClassLoaders.java:178)
        at java.base/java.lang.ClassLoader.loadClass(ClassLoader.java:575)
        at org.springframework.boot.loader.LaunchedURLClassLoader.loadClass(LaunchedURLClassLoader.java:151)
        at java.base/java.lang.ClassLoader.loadClass(ClassLoader.java:521)
        at java.base/java.lang.Class.forName0(Native Method)
        at java.base/java.lang.Class.forName(Class.java:398)
        at org.springframework.util.ClassUtils.forName(ClassUtils.java:285)
        at org.springframework.beans.factory.support.AbstractBeanDefinition.resolveBeanClass(AbstractBeanDefinition.java:456)
        at org.springframework.beans.factory.support.AbstractBeanFactory.doResolveBeanClass(AbstractBeanFactory.java:1542)
        at org.springframework.beans.factory.support.AbstractBeanFactory.resolveBeanClass(AbstractBeanFactory.java:1469)
        ... 24 common frames omitted
Caused by: java.lang.ClassNotFoundException: org.springframework.web.servlet.mvc.method.annotation.ResponseBodyAdvice
        at java.base/jdk.internal.loader.BuiltinClassLoader.loadClass(BuiltinClassLoader.java:582)
        at java.base/jdk.internal.loader.ClassLoaders$AppClassLoader.loadClass(ClassLoaders.java:178)
        at java.base/java.lang.ClassLoader.loadClass(ClassLoader.java:521)
        ... 41 common frames omitted
```

## 报错分析

按照报错的路径找了一下，是有这个类的。

先说一下场景，javaagent 加载了agent这个包，下面是自定义的类加载器。

```
public class AgentClassLoader extends ClassLoader {

    private List<File> jarPathDir;
    private List<File> jarFiles;

    public AgentClassLoader(ClassLoader parent, String rootPath, String[] jarFolder) {
        super(parent);
        jarPathDir = new ArrayList<File>();
        for (int i = 0; i < jarFolder.length; i++) {
            jarPathDir.add(new File(rootPath + File.separator + jarFolder[i]));
        }
        jarFiles = getJarFiles();
    }
}
```

关于类加载器，《深入理解java 虚拟机》里 是这么说的：

**双亲委派模型的工作过程是：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传送到最顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需的类）时，子加载器才会尝试自己去完成加载。 **

很明显， ResponseBodyAdvice 这个类父类没办法加载，所以 agentClassLoader 加载了。

看一下报错的提示的地方，AbstractBeanFactory.resolveBeanClass。

抛出的是一个 error的子类LinkageError。

是 classloader的问题，springboot 的 AbstractBeanFactory 加载 bean的时候，用的 Class.forName 传的类加载器不是 agentClassLoader ，而是取的当前线程的类加载器  LaunchedURLClassLoader ，所以找不到类。

## idea 没报错，shell运行报错

为什么idea 中没有这个问题，我们看一下idea启动时的命令。

```
"C:\Program Files\Java\jdk1.8.0_45\bin\java.exe" -agentlib:jdwp=transport=dt_socket,address=127.0.0.1:57972,suspend=y,server=n -javaagent:D:\dev\workspace\holmes-sniffer\packages\holmes-agent.jar -XX:TieredStopAtLevel=1 -noverify -Dspring.output.ansi.enabled=always -Dcom.sun.management.jmxremote -Dspring.jmx.enabled=true -Dspring.liveBeansView.mbeanDomain -Dspring.application.admin.enabled=true -javaagent:C:\Users\wangguangmin\AppData\Local\JetBrains\IntelliJIdea2020.1\captureAgent\debugger-agent.jar -Dfile.encoding=UTF-8 -classpath "C:\Program Files\Java\jdk1.8.0_45\jre\lib\charsets.jar;C:\xxx" com.maggie.measure.lcoalcachedemo.LcoalCacheDemoApplication
```

可以看出 idea 是 使用 java -classpath 加载了所有jar(包括jdk下面的) 再指定mainclass 启动

这里说一下JVM 类加载的顺序和加载机制。

启动—拓展—应用类—自定义，双亲委托模型：

```
protected synchronized Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    // 首先，检查请求的类是否已经被加载过了 
    Class c = findLoadedClass(name);
    if (c == null) {
        try {
            if (parent != null) {
                c = parent.loadClass(name, false);
            } else {
                c = findBootstrapClassOrNull(name);
            }
        } catch (ClassNotFoundException e) {
            // 如果父类加载器抛出ClassNotFoundException 说明父类加载器无法完成加载请求 
        }
        if (c == null) {
            // 在父类加载器无法加载时 再调用本身的findClass方法来进行类加载 
            c = findClass(name);
        }
    }
    if (resolve) {
        resolveClass(c);
    }
    return c;
}
```

idea 的启动方式重新设置了 classpath ，这样我们jar包里的类会加载到 AppClassloader  类里，而 springboot 也有自己的类加载器，其实就是属于AppClassloader 的自定义类加载器 LaunchedURLClassLoader ，这样无论子类加载器需要加载什么类都可以在 父类AppClassloader 中找到。

再看下 java -jar 的方式为什么会有这个问题。

-jar 选项:如果通过java -jar 来运行一个可执行的jar包,这当前jar包会覆盖上面所有的值.换句话说,-jar 后面所跟的jar包的优先级别最高,如果指定了-jar选项,所有环境变量和命令行制定的搜索路径都将被忽略.JVM APPClassloader将只会以jar包为搜索范围。

也就是是APP classloader只加载了当前jar包内的类（自己编写的代码，以及关联的类）。而其他的第三方jar包，比如 springboot用到的，会由springboot的类加载器加载。而 java -agent 指定的jar 也是 基于AppClassloader  自定义类加载器。这样就会出现上边的问题。

### ## 查看springboot 类加载器 源码

pom.xml

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-loader</artifactId>
</dependency>
```

图：



- SpringBoot通过扩展JarFile、JarURLConnection及URLStreamHandler，实现了jar in jar中资源的加载
- SpringBoot通过扩展URLClassLoader--LauncherURLClassLoader，实现了jar in jar中class文件的加载
- JarLauncher通过加载BOOT-INF/classes目录及BOOT-INF/lib目录下jar文件，实现了fat jar的启动
- WarLauncher通过加载WEB-INF/classes目录及WEB-INF/lib和WEB-INF/lib-provided目录下的jar文件，实现了war文件的直接启动及web容器中的启动

## 怎么解决

1. agent包把相关的依赖移除

```
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>${com.alibaba.fastjson.version}</version>
    <exclusions>
        <exclusion>
            <groupId>org.springframework</groupId>
            <artifactId>spring-webmvc</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

2. 之前用的shade插件打包，在shade把相关类移除

```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <version>3.2.4</version>
    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>shade</goal>
            </goals>
            <configuration>
                <shadedArtifactAttached>false</shadedArtifactAttached>
                <createDependencyReducedPom>false</createDependencyReducedPom>
                <createSourcesJar>false</createSourcesJar>
                <shadeSourcesContent>false</shadeSourcesContent>
                <transformers>
                    <transformer
                            implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                        <manifestEntries>
                            <Premain-Class>${premain.class}</Premain-Class>
                            <Can-Retransform-Classes>true</Can-Retransform-Classes>
                        </manifestEntries>
                    </transformer>
                </transformers>
                <filters>
                    <filter>
                        <artifact>com.alibaba:fastjson</artifact>
                        <includes>
                            <include>com/alibaba/fastjson/**</include>
                        </includes>
                        <excludes>
                            <exclude>com/alibaba/fastjson/support/spring/**</exclude>
                            <exclude>com/yiche/holmes/agent/dependencies/com/alibaba/fastjson/support/spring/**</exclude>
                        </excludes>
                    </filter>
                </filters>
            </configuration>
        </execution>
    </executions>
</plugin>
```

3. 如果这个类确实需要，又想被 自定义类加载器加载，又想被springboot的LaunchedURLClassLoader 加载，怎么办。

   让更高级别的类加载器加载呗。当然目前这个场景可能不太现实。

​       比如 java的 Instrumentation 探针，可以直接用 boostrapclassloader 加载

```
public static void premain(String agentOps, Instrumentation inst) {
    String rootPath = SleuthUtils.getJarDirPath();
    inst.appendToBootstrapClassLoaderSearch(new JarFile(new File(rootPath + "/xxx.jar")));
}
```

4.  第三种方式太极端了，如果都是继承URLClassLoader，让父类的AppClassloader加载就好了。