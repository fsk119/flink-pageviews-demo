# flink-pageviews-demo

## 环境准备
(a) 下载 flink 安装包

Flink 安装包: https://mirrors.tuna.tsinghua.edu.cn/apache/flink/flink-1.12.2/flink-1.12.2-bin-scala_2.11.tgz

(b) 下载 Kafka connector jar * Kafka connector jar: https://repo1.maven.org/maven2/org/apache/flink/flink-sql-connector-kafka_2.11/1.12.2/flink-sql-connector-kafka_2.11-1.12.2.jar

(c) 下载 mysql-cdc connector jar

MySQL CDC jar： https://repo1.maven.org/maven2/com/alibaba/ververica/flink-sql-connector-mysql-cdc/1.2.0/flink-sql-connector-mysql-cdc-1.2.0.jar

(d) 下载 JDBC connector jar

JDBC connector jar: https://repo1.maven.org/maven2/org/apache/flink/flink-connector-jdbc_2.11/1.12.2/flink-connector-jdbc_2.11-1.12.2.jar
MySQL driver jar： https://repo.maven.apache.org/maven2/mysql/mysql-connector-java/5.1.48/mysql-connector-java-5.1.48.jar

## 测试数据准备

我们首先用命令`docker-compose up -d`启动docker。我们可以利用以下命令从 Terminal 进入 Mysql 容器之中，并插入相应的数据。

```
docker exec -it flink-pageviews-demo_debz-mysql_1 bash
mysql -uroot -p123456
```
在 Mysql 中执行以下命令：
```
CREATE DATABASE flink;
USE flink;

CREATE TABLE users (
  user_id BIGINT,
  user_name VARCHAR(1000),
  region VARCHAR(1000)
);

INSERT INTO users VALUES 
(1, 'Timo', 'Berlin'),
(2, 'Tom', 'Beijing'),
(3, 'Apple', 'Beijing');
```

现在，我们利用Sql client在Flink中创建相应的表。
```
CREATE TABLE users (
  user_id BIGINT,
  user_name STRING,
  region STRING
) WITH (
  'connector' = 'mysql-cdc',
  'hostname' = 'localhost',
  'database-name' = 'flink',
  'table-name' = 'users',
  'username' = 'root',
  'password' = '123456'
);

CREATE TABLE pageviews (
  user_id BIGINT,
  page_id BIGINT,
  view_time TIMESTAMP(3),
  proctime AS PROCTIME()
) WITH (
  'connector' = 'kafka',
  'topic' = 'pageviews',
  'properties.bootstrap.servers' = 'localhost:9092',
  'scan.startup.mode' = 'earliest-offset',
  'format' = 'json'
);

```

并利用Flink 往 Kafka中灌入相应的数据

```
INSERT INTO pageviews VALUES
  (1, 101, TO_TIMESTAMP('2020-11-23 15:00:00')),
  (2, 104, TO_TIMESTAMP('2020-11-23 15:00:01.00'));
```

## 将 left join 结果写入 Kafka

我们首先测试是否能将Left join的结果灌入到 Kafka 之中。

首先，我们在 Sql client 中创建相应的表

```
CREATE TABLE enriched_pageviews (
  user_id BIGINT,
  user_region STRING,
  page_id BIGINT,
  view_time TIMESTAMP(3),
  WATERMARK FOR view_time as view_time - INTERVAL '5' SECOND,
  PRIMARY KEY (user_id, page_id) NOT ENFORCED
) WITH (
  'connector' = 'upsert-kafka',
  'topic' = 'enriched_pageviews',
  'properties.bootstrap.servers' = 'localhost:9092',
  'key.format' = 'json',
  'value.format' = 'json'
);
```

并利用以下语句将left join的结果插入到kafka对应的topic之中。

```
INSERT INTO enriched_pageviews
SELECT pageviews.user_id, region, pageviews.page_id, pageviews.view_time
FROM pageviews
LEFT JOIN users ON pageviews.user_id = users.user_id;
```

当作业跑起来后，我们可以另起一个 Terminal 利用命令`docker exec -it flink-pageviews-demo_kafka_1 bash` 进入kafka所在的容器之中。
Kafka的安装路径在于`/opt/kafka`,利用以下命令，我们可以打印topic内的数据`./kafka-console-consumer.sh --bootstrap-server kafka:9094 --topic "enriched_pageviews" --from-beginning --property print.key=true`

```
#预期结果
{"user_id":1,"page_id":101}	{"user_id":1,"user_region":null,"page_id":101,"view_time":"2020-11-23 15:00:00"}
{"user_id":2,"page_id":104}	{"user_id":2,"user_region":null,"page_id":104,"view_time":"2020-11-23 15:00:01"}
{"user_id":1,"page_id":101}	null
{"user_id":1,"page_id":101}	{"user_id":1,"user_region":"Berlin","page_id":101,"view_time":"2020-11-23 15:00:00"}
{"user_id":2,"page_id":104}	null
{"user_id":2,"page_id":104}	{"user_id":2,"user_region":"Beijing","page_id":104,"view_time":"2020-11-23 15:00:01"}

```
<b>Left join</b>中，右流发现左流没有join上但已经发射了，此时会发送`DELETE`消息，而非`UPDATE-BEFORE`消息清理之前发送的消息。详见`org.apache.flink.table.runtime.operators.join.stream.StreamingJoinOperator#processElement`

我们可以进一步在mysql中删除或者修改一些数据，来观察进一步的变化。

```
UPDATE users SET region = 'Beijing' WHERE user_id = 1;

DELETE FROM users WHERE user_id = 1;
```

## 将聚合结果写入 Kafka

我们进一步测试将聚合的结果写入到 Kafka 之中。

在Sql client 中构建以下表
```
CREATE TABLE pageviews_per_region (
  user_region STRING,
  cnt BIGINT,
  PRIMARY KEY (user_region) NOT ENFORCED
) WITH (
  'connector' = 'upsert-kafka',
  'topic' = 'pageviews_per_region',
  'properties.bootstrap.servers' = 'localhost:9092',
  'key.format' = 'json',
  'value.format' = 'json'
)
```

我们再用以下命令将数据插入到upsert-kafka之中。

```
INSERT INTO pageviews_per_region
SELECT
  user_region,
  COUNT(*)
FROM enriched_pageviews
WHERE user_region is not null
GROUP BY user_region;
```

我们可以通过以下命令查看 Kafka 中对应的数据

```
./kafka-console-consumer.sh --bootstrap-server kafka:9094 --topic "pageviews_per_region" --from-beginning --property print.key=true

# 预期结果
{"user_region":"Berlin"}	{"user_region":"Berlin","cnt":1}
{"user_region":"Beijing"}	{"user_region":"Beijing","cnt":1}
{"user_region":"Berlin"}	null
{"user_region":"Beijing"}	{"user_region":"Beijing","cnt":2}
{"user_region":"Beijing"}	{"user_region":"Beijing","cnt":1}
```
