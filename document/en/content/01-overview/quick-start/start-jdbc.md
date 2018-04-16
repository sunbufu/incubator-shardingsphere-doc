+++
toc = true
title = "Sharding-JDBC Quick Start"
weight = 1
+++

## 1. Import maven dependency

```xml
<dependency>
    <groupId>io.shardingjdbc</groupId>
    <artifactId>sharding-jdbc-core</artifactId>
    <version>${latest.release.version}</version>
</dependency>
```

## 2. Configure sharding rule configuration

Sharding-JDBC support 4 types for sharding rule configuration, they are `Java`, `YAML`, `Spring namespace` and `Spring boot starter`. Developers can choose any one for best suitable situation. More details please reference [Configuration Manual](/06-sharding-jdbc/02-configuration/)。

## 3. Create DataSource

Use ShardingDataSourceFactory to create ShardingDataSource, which is a standard JDBC DataSource. Then developers can use it for raw JDBC, JPA, MyBatis or Other JDBC based ORM frameworks.

```java
DataSource dataSource = ShardingDataSourceFactory.createDataSource(dataSourceMap, shardingRuleConfig);
```