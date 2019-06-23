---
title: Maven 打包趟坑与解法
date: 2018-09-14 11:19:06
tags: 
    - Java
    - Maven
---

### 基础
首先明确:

0. 当你使用 Maven 对项目打包时，你需要了解以下 3 个打包 plugin，它们分别是

| plugin | function |
| --- | --- |
| maven-jar-plugin | maven 默认打包插件，用来创建 project jar |
| maven-shade-plugin | 用来打可执行包，executable(fat) jar |
| maven-assembly-plugin | 支持定制化打包方式，例如 apache 项目的打包方式 |

不管你 dependences 里的 scope 设置为什么, mvn package 出来的 你 src 的 jar 包里, 只会有你的 class 文件, 不会有所依赖的 jar 包, 可以通过 maven assembly 插件来做这个事情. 但是如果打成 war 包, 是会包含 compile scope 的依赖的. 而 provided 是要容器提供, 比如说 Tomcat, 会到 Tomcat 的 `$liferay-tomcat-home\webapps\ROOT\WEB-INF\lib` 目录下找.

1. mvn install 出来的 jar 包只会包含自己的 src 的 classes. 即使你是 compile 的依赖, 也不会进去, 但是如果打成 war 包, 是会包含 compile scope 的依赖的. 而 provided 是要容器提供, 比如说 Tomcat, 会到 Tomcat 的 `$liferay-tomcat-home\webapps\ROOT\WEB-INF\lib` 目录下找. 而且 compile 的依赖是传递的, provided 的不传递.

2. 可以通过 assembly/shade 插件把依赖的 jar 包打到一个 assembly.jar 包中去. 和源码的 jar 包可以是独立的, 也可以打到一起. 如果你一个依赖(D1)有两个版本(在父/子pom 中都有定义, 但是版本不一样), 在打出的 jar 包里只会有一个版本, 因为路径里不带版本的. 所以会出现各种 NoSuchMethodError 等等问题, 因为编译的时候都是各自用的正确的 D1 编译的出的 class. 但是运行时用到的 D1 只会有一个版本, 会有不匹配.

所以, 不要在一个项目里, 不同 pom 里面尝试使用不同 version 的依赖. 来看个实例:

parquet-column 里会 shade 一个 fasttuil, 你 jar -tf parquet-column.jar 看他 里面会有这个 fastutil.

![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/18-11-8/50973815.jpg)

可以看到有两个 jar 包, 一个 origin 不带 shade 的 fastutil, 另外一个是带着的, 也是放到 maven 仓库的 jar 包.
![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/18-11-8/29736635.jpg)

jar -tf 确认
![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/18-11-8/57385130.jpg)


### 问题定位一般方法
0. 当你遇到 `java.lang.NoClassDefFoundError` 等错误的时候, 如果是在 IDEA 里运行的, 很有可能是是 provided 依赖. 具体可以先看 IDEA 中打出的 classpath 里有没有依赖的包,![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/5.png), 然后看看 iml 里的 scope ![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/11871540974627_.pic_hd.jpg). 解决方法是 把 iml 里的都改成 compile 或者在 IDEA 中勾选上这个:
![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/11861540974584_.pic.jpg)

1. 如果有遇到什么 NoSuchMethodError, ClassNotFoundException 等等的, 先看看打印出来的 classpath. IDEA 里可以直接看:
![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/18-11-8/76336665.jpg)

2. 然后可以 double shift, 搜下出问题的类, 一般会跳出来多个:
![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/18-11-8/65076854.jpg)

3. 然后再用 `mvn dependency:tree` 看下当前 model 用的哪个版本的依赖

然后就可以做相应的操作, 一般有以下几种:

    1. exclusive 相应依赖
    2. 写死用一个版本的
    3. 把 dependency 的依赖做 external 模块, 然后 shade + reloaction, 可以参见 spark 的 external model.

### 实战

程序里报错`Caused by: java.lang.NoSuchMethodError: com.fasterxml.jackson.databind.ObjectMapper.canSerialize(Ljava/lang/Class;Ljava/util/concurrent/atomic/AtomicReference;)Z`

但是无论从`mvn dependency:tree`, 还是运行时加载的 jar 包来看, 都是用了正确的 `jackson-databind-2.6.5.jar`. 问题就刁钻在它用的这个类, 其实不是 `jackson-databind` 里的, 而是其他的包里 shaed 但是又没有 relocation 的. 除非你把这个包给从依赖李去掉, 在这个包的里面的依赖里去掉, 或者最外面加正确版本的`jackson-databind-2.6.5.jar`都是没有用的, 见下图:

![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/18-9-14/40098244.jpg)

所以画框里他 exclusive 也是没有用的. 解决方法就是我们做成 external 的, 并且 exclude 掉.

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <artifactId>external-influxdb</artifactId>
    <packaging>jar</packaging>
    <name>External-InfluxDB</name>
    <url>http://kyligence.io</url>
    <description>Curator for KAP</description>

    <parent>
        <groupId>apache</groupId>
        <artifactId>kylin</artifactId>
        <version>3.0.0-SNAPSHOT</version>
        <relativePath>../../../pom.xml</relativePath>
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <shadeBase>org.apache.kylin.shaded.influxdb</shadeBase>
        <shaded.curator.version>2.12.0</shaded.curator.version>
    </properties>


    <dependencies>
        <dependency>
            <groupId>org.influxdb</groupId>
            <artifactId>influxdb-java</artifactId>
            <version>2.5</version>
            <scope>compile</scope>
        </dependency>
        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>20.0</version>
            <scope>compile</scope>
        </dependency>
        <!-- cover log4j from parent pom-->
        <dependency>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>jcl-over-slf4j</artifactId>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <scope>provided</scope>
        </dependency>
    </dependencies>
    <!--overwrite parent, need to upgrade this when upgrade grpc-->
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <artifactSet>
                                <includes>
                                    <include>*:*</include>
                                </includes>
                                <excludes>
                                    <exclude>log4j:*</exclude>
                                    <exclude>org.slf4j:*</exclude>
                                </excludes>
                            </artifactSet>
                            <relocations>
                                <relocation>
                                    <pattern>org.influxdb</pattern>
                                    <shadedPattern>${shadeBase}.org.influxdb</shadedPattern>
                                </relocation>
                                <relocation>
                                    <pattern>com.squareup.moshi</pattern>
                                    <shadedPattern>${shadeBase}.com.squareup.moshi</shadedPattern>
                                </relocation>
                                <relocation>
                                    <pattern>okhttp3</pattern>
                                    <shadedPattern>${shadeBase}.okhttp3</shadedPattern>
                                </relocation>
                                <relocation>
                                    <pattern>okio</pattern>
                                    <shadedPattern>${shadeBase}.okio</shadedPattern>
                                </relocation>
                                <relocation>
                                    <pattern>retrofit2</pattern>
                                    <shadedPattern>${shadeBase}.retrofit2</shadedPattern>
                                </relocation>
                                <relocation>
                                    <pattern>com.google</pattern>
                                    <shadedPattern>${shadeBase}.com.google.common</shadedPattern>
                                </relocation>
                            </relocations>
                            <filters>
                                <filter>
                                    <artifact>*:*</artifact>
                                    <excludes>
                                        <exclude>META-INF/*.SF</exclude>
                                        <exclude>META-INF/*.DSA</exclude>
                                        <exclude>META-INF/*.RSA</exclude>
                                    </excludes>
                                </filter>
                            </filters>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

#### 番外
maven 的不同 scope 的官方定义:

- compile

    This is the default scope. Compile dependencies are available in all classpaths of a project. Furthermore, those dependencies are propagated to dependent projects(会有依赖传递).

- provided

    This is much like compile, but indicates you expect the JDK or a container to provide the dependency at runtime. For example, when building a web application for the Java Enterprise Edition, you would set the dependency on the Servlet API and related Java EE APIs to scope provided because the web container provides those classes. This scope is only available on the compilation and test classpath, and is not transitive.

我们经常回用到 `-pl :moduleName`, 看着很奇怪, 其实:前面省略的是 groupId.
