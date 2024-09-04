---
title: JOOQ Start
date: 2016-11-07 17:16:03
tags:
  - JOOQ
---
JOOQ 简单的使用

毕业以后进入的公司数据库持久层用的JOOQ，流式的API敲起来很爽。这里记录一下。具体的用法可以直接查看官网文档 *http://www.jooq.org/*

### Code Generation
JOOQ是根据数据库自动生成的表对象。这里我们使用maven的插件来生成。官方的maven配置* http://www.jooq.org/doc/3.8/manual-single-page/#codegen-configuration*

  ```xml

  <plugin>
    <!-- Specify the maven code generator plugin -->
    <groupId>org.jooq</groupId>
    <artifactId>jooq-codegen-maven</artifactId>
    <!--<version>${jooq.version}</version>-->
    <!-- The plugin should hook into the generate goal -->
    <executions>
        <execution>
            <id>generate-mysql</id>
               <phase>generate-sources</phase>
            <goals>
                <goal>generate</goal>
            </goals>
        </execution>
    </executions>
    <!-- Manage the plugin's dependency. In this example, we'll use a MySQL database -->
    <dependencies>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.37</version>
        </dependency>
    </dependencies>

    <!-- Specify the plugin configuration.
    The configuration format is the same as for the standalone code generator -->
    <configuration>
        <!-- JDBC connection parameters -->
        <jdbc>
            <driver>com.mysql.jdbc.Driver</driver>
            <url>jdbc:mysql://localhost:3306/test</url>
            <user>root</user>
            <password>root</password>
        </jdbc>
        <!-- Generator parameters -->
        <generator>
            <database>
                <name>org.jooq.util.mysql.MySQLDatabase</name>
                <includes>.*</includes>
                <inputSchema>test</inputSchema>  <!--database name-->
            </database>
            <target>
                <packageName>me.lty.data</packageName>
                <directory>src/main/java</directory>
            </target>
        </generator>
    </configuration>
</plugin>
  ```
然后我们导入JOOQ的包

```xml
  <dependency>
     <groupId>org.jooq</groupId>
     <artifactId>jooq</artifactId>
     <version>${jooq.version}</version>
 </dependency>
```
### 简单的代码

```java
//这使用jOOQ默认的配置
//或者 DSLContext create = new DefaultDSLContext(SQLDialect.MYSQL);
DSLContext create = DSL.using(new DefaultConfiguration().derive(SQLDialect.MYSQL));
//这里就是简单的根据ID获取的操作。更多的用法请查看官方文档
MainRecord record  = create.selectFrom(MAIN).where(MAIN.NUMBER.eq(id)).fetchOne();
```

---
<p align='center'><font color='blue'>书籍把我们引入最美好的社会，使我们认识各个时代的伟大智者。</font></p><p align='right'>——史美尔斯</p>

---
