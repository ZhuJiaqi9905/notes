# disni踩坑总结

## 使用maven

### 添加依赖

在pom.xml中加入下面的依赖，注意到disni已经有2.1了，不需要用它README.md中的1.5

```xml
<dependencies>
	...
    <dependency>
        <groupId>com.ibm.disni</groupId>
        <artifactId>disni</artifactId>
        <version>2.1</version>
    </dependency>
</dependencies
```

### 编译项目

在写完自己的代码逻辑后，编译项目：

```shell
mvn -DskipTests install
```

### 运行项目

#### libdisni的编译

在运行项目之前，要确保你的机器上有libdisni的链接库，目前dl24和dl25都已经有了。libdisni的编译参考https://github.com/zrlio/disni就行。

编译好的链接文件在`/usr/local/lib`下。

#### 运行java项目

先用mvn查找你的dependency的classpath:`mvn dependency:build-classpath`

进入到生成class文件的文件夹中：`cd target/classes`

运行项目的命令行参数：

- `-classpath`: 运行时要指定classpath参数,不然java虚拟机找不到你import的文件。而且除了上面你dependency的classpath，还要加上你自己项目的jar包的classpath

- `-Djava.library.path`:运行时要指定libdisni的链接文件的位置

最终如下：

```shell
java -classpath /home/zhujiaqi/.m2/repository/com/ibm/disni/disni/2.1/disni-2.1.jar:/home/zhujiaqi/.m2/repository/org/slf4j/slf4j-log4j12/1.7.12/slf4j-log4j12-1.7.12.jar:/home/zhujiaqi/.m2/repository/org/slf4j/slf4j-api/1.7.12/slf4j-api-1.7.12.jar:/home/zhujiaqi/.m2/repository/log4j/log4j/1.2.17/log4j-1.2.17.jar:/home/zhujiaqi/.m2/repository/commons-cli/commons-cli/1.3.1/commons-cli-1.3.1.jar:/home/zhujiaqi/test_disni/target/test_disni-1.0-SNAPSHOT.jar -Djava.library.path=/usr/local/lib/ VerbsClient
```

然后就跑起来了！