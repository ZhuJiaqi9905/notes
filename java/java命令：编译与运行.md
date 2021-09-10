# java命令：编译与运行

指定classpath，中间用`:`分割

```shell
java -classpath /Users/zhujiaqi/Desktop/code/java/test_disni/target/classes:/Users/zhujiaqi/.m2/repository/com/ibm/disni/disni/1.5/disni-1.5.jar:/Users/zhujiaqi/.m2/repository/org/slf4j/slf4j-log4j12/1.7.12/slf4j-log4j12-1.7.12.jar:/Users/zhujiaqi/.m2/repository/org/slf4j/slf4j-api/1.7.12/slf4j-api-1.7.12.jar:/Users/zhujiaqi/.m2/repository/log4j/log4j/1.2.17/log4j-1.2.17.jar:/Users/zhujiaqi/.m2/repository/commons-cli/commons-cli/1.3.1/commons-cli-1.3.1.jar VerbsServer

```





```
/home/zhujiaqi/test_disni/target/classes:/home/zhujiaqi/.m2/repository/com/ibm/disni/disni/1.5/disni-1.5.jar:/home/zhujiaqi/.m2/repository/org/slf4j/slf4j-log4j12/1.7.12/slf4j-log4j12-1.7.12.jar:/home/zhujiaqi/.m2/repository/org/slf4j/slf4j-api/1.7.12/slf4j-api-1.7.12.jar:/home/zhujiaqi/.m2/repository/log4j/log4j/1.2.17/log4j-1.2.17.jar:/home/zhujiaqi/.m2/repository/commons-cli/commons-cli/1.3.1/commons-cli-1.3.1.jar
```

mvn dependency:build-classpath



```
mvn -DskipTests install
```

mvn和junit

https://blog.csdn.net/lfsfxy9/article/details/12201033

mvn设置java.library.path

参考这个stackoverflow：

只要把pom.xml文件改一下就行

https://stackoverflow.com/questions/35366035/set-java-library-path-for-testing

以及这个：

https://maven.apache.org/surefire/maven-surefire-plugin/examples/system-properties.html

