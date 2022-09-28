# 日志流处理

## MySQL审计日志

### 确认开启MySQL日志审计功能
``` 
[mysqld] 
server_audit_logging=on  #开启审计功能
server_audit_file_path =/opt/data/mysql/server_audit.log  #指定审计日志文件存放路径 
server_audit_file_rotate_size=10000000
server_audit=FORCE_PLUS_PERMANENT  #防止server_audit 插件被卸载
```

```sql_more=
mysql> show variables like '%audit%';
```

### 配置Fluentd  .conf文件
``` 
<source>
  @type tail
  path /tmp/test/server_audit.log
  pos_file /fluentd/log/pos/server_audit311.pos
  follow_inodes true
  #read_from_head true
  tag audit311
<parse>
  @type regexp
  expression (?<logtime>[^,]*),(?<serverhost>[^,]*),(?<username>[^,]*),(?<host>[^,]*),(?<connectionid>[^,]*),(?<queryid>[^,]*),(?<operation>[^,]*),(?<database>[^,]*),(?<object>'[^?]*'),(?<retcode>\d*)
  time_key logtime
  keep_time_key true
  time_format %Y%m%d %H:%M:%S
</parse>
</source>

<filter audit311>
  @type grep
  <exclude>
    key operation
    pattern /CONNECT/
  </exclude>

  <exclude>
    key object
    pattern /commit|java|show|SHOW/
  </exclude>
</filter>


<match audit311>
  @type kafka2
  brokers 192.168.2.175:29092
  #use_event_time true
  # buffer settings
  <buffer>
    @type file
    path /fluentd/log/audit/kafka2
    chunk_limit_size 9MB
    flush_interval 3s
  </buffer>
  # data type settings
  <format>
    @type json
  </format>
  # topic settings
  default_topic audit311
  # producer settings
  required_acks -1
  # compression_codec gzip
</match>
``` 

进入fluentd容器
```shell=
docker exec -it fluentd-test sh
```
检查配置文件是否正确
```
fluentd --dry-run -c /fluentd/etc/mysql_audit.conf 
```
执行已配置的.conf
```
fluentd  -c /fluentd/etc/mysql_audit.conf
```

### ksql流处理
```shell=
docker exec -it ksqldb-cli ksql http://ksqldb-server:8088
```

**1.基于audit311创建流**
```sql_more=
CREATE STREAM AUDIT_LOG_JSON311 
	(LOGTIME STRING, 
	SERVERHOST STRING, 
	USERNAME STRING, 
	HOST STRING, 
	CONNECTIONID INTEGER,
	QUERYID INTEGER, 
	OPERATION STRING, 
	DATABASE STRING,
	OBJECT STRING, 
	RETCODE INTEGER) 
WITH (KAFKA_TOPIC='audit311',KEY_FORMAT='KAFKA', PARTITIONS=1,VALUE_FORMAT='JSON');
```
**2.基于AUDIT_LOG_JSON311解析出sql_type并添加key**
```sql_more=
CREATE STREAM audit_stream311 WITH (FORMAT='AVRO') AS SELECT 
    host,
    username,
    LOGTIME,
    database,
    UCASE(REGEXP_EXTRACT('([A-Za-z]+) .*',object,1)) as sql_type 
FROM AUDIT_LOG_JSON31
PARTITION BY host, username
EMIT CHANGES;
```
**3.创建实时查询表：根据IP,用户名，数据库，sql_type维度进行统计并取最新一条sql的执行时间**
```sql_more=
CREATE table audit_sum311 AS SELECT 
    host,
    username,
    database,
    sql_type, 
    count(sql_type) as cnt,
LATEST_BY_OFFSET(LOGTIME) as last_time 
FROM audit_stream311 
group BY host, username,database,sql_type
EMIT CHANGES;
```
**4.创建connector将实时表推到目标数据库**
```sql_more=
CREATE sink CONNECTOR `to_audit_sum311` WITH(
    "connector.class"= 'io.confluent.connect.jdbc.JdbcSinkConnector',
    "connection.url" = 'jdbc:mysql://192.168.2.174:3307/test?user=kafka_pro&password=******',
    "key.converter" = "io.confluent.connect.avro.AvroConverter",
    "key.converter.schema.registry.url" = 'http://schema-registry:8081',
    "value.converter" = 'io.confluent.connect.avro.AvroConverter',
    "value.converter.schema.registry.url" = 'http://schema-registry:8081',
    "insert.mode"='upsert',
    "topics"='AUDIT_SUM311', 
    "table.name.format"='audit_sum311',
    "pk.mode"='record_key',
    "pk.fields"= 'HOST,USERNAME,DATABASE,SQL_TYPE',
    "delete.enabled"='true');
```



