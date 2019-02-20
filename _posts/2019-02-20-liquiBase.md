---
layout: post
title: Liquibase使用入门
categories: [LiquiBase]
description: LiquiBase用法
keywords: LiquiBase
---
### 1、LiquiBase简介
LiquiBase是一个用于数据库重构和迁移的开源工具，通过日志文件的形式记录数据库的变更，然后执行日志文件中的修改，将数据库更新或回滚到一致的状态。LiquiBase的主要特点有：

- 支持几乎所有主流的数据库，如MySQL, PostgreSQL, Oracle, Sql Server, DB2等；
- 支持多开发者的协作维护；
- 日志文件支持多种格式，如XML, YAML, JSON, SQL等；
- 支持多种运行方式，如命令行、Spring集成、Maven插件、Gradle插件等；

### 2、日志文件changeLog
changelog是LiquiBase用来记录数据库的变更，一般放在CLASSPATH下，然后配置到执行路径中。 

changelog支持多种格式，主要有XML/JSON/YAML/SQL，推荐使用xml格式。[官网格式](http://www.liquibase.org/documentation/databasechangelog.html)

一个<changeSet>标签对应一个变更集，由属性id、name，以及changelog的文件路径唯一标识。 

changelog在执行的时候并不是按照id的顺序，而是按照changeSet在changelog中出现的顺序。

changelog中的一个changeSet对应一个事务，在changeSet执行完后commit，如果出现错误则rollback。

### 3、简单入门使用
1）在application.properties中配置changeLog路径

```
# liquibase配置
liquibase.enabled=true
#默认位置
#liquibase.change-log=classpath:/db/changelog/db.changelog-master.yaml

liquibase.change-log=classpath:/db/changelog/sqlData.xml
```

2）xml配置sample

```
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog" 
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-2.0.xsd">

	<changeSet author="yejg" id="sql-01">
		<sqlFile path="classpath:db/changelog/sqlfile/init.sql" encoding="UTF-8" />
		<sqlFile path="classpath:db/changelog/sqlfile/users.sql" encoding="UTF-8" />
	</changeSet>

	<changeSet author="yejg" id="sql-02">
		<sqlFile path="classpath:db/changelog/sqlfile/users2.sql" encoding="UTF-8" />
	</changeSet>
 
</databaseChangeLog>
```


3）要执行的sql
```
--- init.sql
CREATE TABLE usersTest(
  user_id                   varchar2(14)    DEFAULT ' '          NOT NULL,
  user_name             varchar2(128)   DEFAULT ' '          NOT NULL
  )  STORAGE(FREELISTS 20 FREELIST GROUPS 2) NOLOGGING TABLESPACE USER_DATA;

insert into usersTest(user_id,user_name) values ('0','测试');
```

4）启动项目时，日志效果
```
2019-02-19 15:15:22,425[INFO][main][][][] {dataSource-1} inited [com.alibaba.druid.pool.DruidDataSource.init(DruidDataSource.java:721)] 
2019-02-19 15:15:23,065[INFO][main][][][] Could not set remarks reporting on OracleDatabase: com.alibaba.druid.pool.DruidPooledConnection.setRemarksReporting(boolean) [org.springframework.boot.liquibase.CommonsLoggingLiquibaseLogger.info(CommonsLoggingLiquibaseLogger.java:92)] 
2019-02-19 15:15:24,036[INFO][main][][][] Successfully acquired change log lock [org.springframework.boot.liquibase.CommonsLoggingLiquibaseLogger.info(CommonsLoggingLiquibaseLogger.java:92)] 
2019-02-19 15:15:25,469[INFO][main][][][] Creating database history table with name: DATABASECHANGELOG [org.springframework.boot.liquibase.CommonsLoggingLiquibaseLogger.info(CommonsLoggingLiquibaseLogger.java:92)] 
2019-02-19 15:15:25,496[INFO][main][][][] Reading from DATABASECHANGELOG [org.springframework.boot.liquibase.CommonsLoggingLiquibaseLogger.info(CommonsLoggingLiquibaseLogger.java:92)] 
2019-02-19 15:15:25,574[INFO][main][][][] classpath:/db/changelog/sqlData.xml: classpath:/db/changelog/sqlData.xml::sql-01::yejg: SQL in file classpath:db/changelog/sqlfile/init.sql executed [org.springframework.boot.liquibase.CommonsLoggingLiquibaseLogger.info(CommonsLoggingLiquibaseLogger.java:92)] 
2019-02-19 15:15:25,581[INFO][main][][][] classpath:/db/changelog/sqlData.xml: classpath:/db/changelog/sqlData.xml::sql-01::yejg: SQL in file classpath:db/changelog/sqlfile/users.sql executed [org.springframework.boot.liquibase.CommonsLoggingLiquibaseLogger.info(CommonsLoggingLiquibaseLogger.java:92)] 
2019-02-19 15:15:25,586[INFO][main][][][] classpath:/db/changelog/sqlData.xml: classpath:/db/changelog/sqlData.xml::sql-01::yejg: ChangeSet classpath:/db/changelog/sqlData.xml::sql-01::yejg ran successfully in 58ms [org.springframework.boot.liquibase.CommonsLoggingLiquibaseLogger.info(CommonsLoggingLiquibaseLogger.java:92)] 
2019-02-19 15:15:25,624[INFO][main][][][] classpath:/db/changelog/sqlData.xml: classpath:/db/changelog/sqlData.xml::sql-02::yejg: SQL in file classpath:db/changelog/sqlfile/users2.sql executed [org.springframework.boot.liquibase.CommonsLoggingLiquibaseLogger.info(CommonsLoggingLiquibaseLogger.java:92)] 
2019-02-19 15:15:25,627[INFO][main][][][] classpath:/db/changelog/sqlData.xml: classpath:/db/changelog/sqlData.xml::sql-02::yejg: ChangeSet classpath:/db/changelog/sqlData.xml::sql-02::yejg ran successfully in 10ms [org.springframework.boot.liquibase.CommonsLoggingLiquibaseLogger.info(CommonsLoggingLiquibaseLogger.java:92)] 
2019-02-19 15:15:25,649[INFO][main][][][] Successfully released change log lock [org.springframework.boot.liquibase.CommonsLoggingLiquibaseLogger.info(CommonsLoggingLiquibaseLogger.java:92)] 
```

5）简单原理分析

在执行changelog的时候，liquibase会在数据库中新建2张表，写执行记录：
- databasechangelog
- databasechangeloglock


6）生成已有数据库的changeLog 

首先，在pom文件中增加如下配置
```
<build>
	<plugins>
		<plugin>
			<groupId>org.liquibase</groupId>
			<artifactId>liquibase-maven-plugin</artifactId>
			<version>3.4.2</version>
			<configuration>
				<propertyFile>src/main/resources/liquibase.properties</propertyFile>
				<propertyFileWillOverride>true</propertyFileWillOverride>
				<!--生成文件的路径-->
				<outputChangeLogFile>src/main/resources/changelog_dev.xml</outputChangeLogFile>
			</configuration>
		</plugin>
	</plugins>
</build>
```
其中，liquibase.properties的配置如下
```
changeLogFile=src/main/resources/db/changelog/sqlData.xml
driver=oracle.jdbc.driver.OracleDriver
url=jdbc:oracle:thin:@XXXX
username=XXX
password=XXX
verbose=true
## 生成文件的路径
outputChangeLogFile=src/main/resources/changelog_dev.xml
```

然后，执行【mvn liquibase:generateChangeLog】命令，就会生成changelog_dev.xml文件


### 4、其他应用
为减少部署实施的工作量，可以利用liquibase在项目中配置好sql和changeLog，让项目启动的时候，就自动去打sql脚本，实现自动化处理。

不过，目前发现liquibase不支持执行语句块，暂未找到解决办法。

### 5、参考 
https://www.cnblogs.com/tonyq/p/8039770.html
https://blog.csdn.net/qq_26000415/article/details/79076567