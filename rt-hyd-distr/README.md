# 实时数据分发
## Mysql连接器
### 确保mysql配置项中有以下这些内容

```ini
[mysqld]  
server-id = 唯一编码
log_bin = mysql-bin   
binlog_format = ROW   
binlog_row_image = FULL   
expire_logs_days = 10  
gtid_mode = ON  
enforce_gtid_consistency = ON  
```

### 全表增删改同步
1. **使用Debezium连接器接入数据**
```sql_more=
CREATE SOURCE CONNECTOR dbz_st_river_r WITH (   
    "connector.class" = 'io.debezium.connector.mysql.MySqlConnector',   
    "database.hostname" = 'ip地址',   
    "database.port" = '端口',   
    "database.user" = 'username',   
    "database.password" = 'password',   
    "database.allowPublicKeyRetrieval" = 'true',   
    "key.converter"='io.confluent.connect.avro.AvroConverter',   
    "key.converter.schema.registry.url"   = 'http://schema-registry:8081',   
    "value.converter"='io.confluent.connect.avro.AvroConverter',   
    "value.converter.schema.registry.url" = 'http://schema-registry:8081',   
    "database.server.id"='206',   
    "database.server.name" = 'river_r',   
    "database.whitelist" = 'zjsl_rain',   
    "database.history.kafka.bootstrap.servers"='broker:9092',   
    "database.history.kafka.topic"='st_river_r_his',   
    "table.whitelist" = 'zjsl_rain.st_river_r',   
    "include.schema.changes"='false',   
    "decimal.handling.mode"='string',   
    "topic.creation.default.replication.factor"='-1',   
    "topic.creation.default.partitions"='20',   
    "time.precision.mode"='connect',   
    "transforms"='unwrap',   
    "transforms.unwrap.type"='io.debezium.transforms.ExtractNewRecordState',
    "transforms.unwrap.drop.tombstones"='false'   
);   
```
 
2. **sink到mysql实现数据的增删改同步**
```sql_more=
CREATE SINK CONNECTOR `sink_st_river_r` WITH(
    "connector.class"='io.confluent.connect.jdbc.JdbcSinkConnector',   
    "connection.url"='jdbc:mysql://ip地址:3306/目标库?user=username&password=password',   
    "insert.mode"='upsert',   
    "tasks.max"='1',   
    "topics"='river_r.zjsl_rain.st_river_r ',    
    "key.converter" = 'io.confluent.connect.avro.AvroConverter',   
    "key.converter.schema.registry.url"='http://schema-registry:8081',   
    "value.converter"= 'io.confluent.connect.avro.AvroConverter',      
    "value.converter.schema.registry.url"='http://schema-registry:8081',   
    "table.name.format"='st_river_r',   
    "pk.mode"='record_key',   
    'pk.fields'= 'STCD,TM',
    "delete.enabled"='true'
);   
```
###  根据测站切分实时数据并分发
1. **通过视图获取数据**
    ```mermaid
      flowchart  LR;
      st(开始) -->
      step1[建立对应筛选视图]-->
      step2[接入视图数据]--> 
      step3[sink到目标表]-->
      ed(结束)
    ```
    1. **建立对应筛选视图 例如筛选嘉兴部分河道水情数据**
    ```sql_more=
    CREATE VIEW `vw_st_river_r_3304` AS SELECT
        `b`.`STCD` AS `STCD`,
        `b`.`TM` AS `TM`,
        `b`.`Z` AS `Z`,
        `b`.`Q` AS `Q`,
        `b`.`XSA` AS `XSA`,
        `b`.`XSAVV` AS `XSAVV`,
        `b`.`XSMXV` AS `XSMXV`,
        `b`.`FLWCHRCD` AS `FLWCHRCD`,
        `b`.`WPTN` AS `WPTN`,
        `b`.`MSQMT` AS `MSQMT`,
        `b`.`MSAMT` AS `MSAMT`,
        `b`.`MSVMT` AS `MSVMT`,
        `b`.`TONG_TIME` AS `TONG_TIME`,
        `b`.`OP` AS `OP` 
    FROM
        (
        `st_stbprp_b` `a`
        JOIN `st_river_r` `b` ON ((
        `a`.`STCD` = `b`.`STCD` 
        ))) 
    WHERE
        (
        `a`.`ADDVCD` LIKE '3304%')
    ```

    2. **使用jdbc-mysql连接接入视图数据**
    ```sql_more=
    DROP CONNECTOR IF EXISTS `jdbc_vw_st_river_r_3304`;
    CREATE SOURCE CONNECTOR `jdbc_vw_st_river_r_3304` WITH(
        "connector.class"='io.confluent.connect.jdbc.JdbcSourceConnector',
        "connection.url"='jdbc:mysql://ip地址:3308/zjsl_rain?user=username&password=password',
        "mode"='timestamp',
        "key.converter"='io.confluent.connect.avro.AvroConverter',
        "key.converter.schema.registry.url"='http://schema-registry:8081',
        "value.converter"='io.confluent.connect.avro.AvroConverter',
        "value.converter.schema.registry.url"='http://schema-registry:8081',
        "timestamp.column.name"='tong_time',
        "numeric.mapping"='best_fit',
        "topic.prefix"='jdbc_',
        "table.types"='view',
        "table.whitelist"='vw_st_river_r_3304'
    );
    ```
    3. **将数据sink到目标表 （jdbc连接器不能捕获删除 只能插入、更新）**
    ```sql_more=
    CREATE SINK CONNECTOR `sink_vw_river_r_3304` WITH(
        "connector.class"='io.confluent.connect.jdbc.JdbcSinkConnector',
        "connection.url"='jdbc:mysql://ip地址:端口/test?user=username&password=password',
        "insert.mode"='upsert',
        "topics"='jdbc_vw_st_river_r_3304', 
        "key.converter" = 'io.confluent.connect.avro.AvroConverter',
        "key.converter.schema.registry.url"='http://schema-registry:8081',
        "value.converter"= 'io.confluent.connect.avro.AvroConverter',
        "value.converter.schema.registry.url"='http://schema-registry:8081',
        "table.name.format"='st_river_r_3304',
        "pk.mode"='record_value',
        "pk.fields"='STCD,TM'
    );
    ```
2. **通过将部分测站信息接入后关联实时数据**
    ```mermaid
      flowchart  LR;
      st(开始) -->
      step1[接入某地测站数据]-->
      step2[创建对应流表]--> 
      step3[关联测站与实时数据]-->
      step5[sink到目标端]--> 
      ed(结束)
    ```
    1. **接入嘉兴测站数据**
    ```sql_more=
     CREATE SOURCE CONNECTOR `jdbc_st_stbprp_b_3304` WITH (
        "connector.class"='io.confluent.connect.jdbc.JdbcSourceConnector',
        "connection.url"='jdbc:mysql://ip地址:端口/zjsl_rain?user=username&password=password',
        "mode"='bulk',
        "incrementing.column.name"='STCD',
        "key.converter"='io.confluent.connect.avro.AvroConverter',
        "key.converter.schema.registry.url"='http://schema-registry:8081',
        "value.converter"='io.confluent.connect.avro.AvroConverter',
        "value.converter.schema.registry.url"='http://schema-registry:8081',
        "numeric.mapping"='best_fit',
        "poll.interval.ms"='360000',
        "topic.prefix"='jdbc_st_stbprp_b',
        "query"='select stcd,stnm,addvcd,tong_time from st_stbprp_b where ADDVCD like "3304%";',
        "transforms"='ValueToKey',
        "transforms.ValueToKey.type"='org.apache.kafka.connect.transforms.ValueToKey',
        "transforms.ValueToKey.fields"='stcd'
    );
    ```
    2. **建立和topic结构一样的流表**
    ```sql_more
    CREATE STREAM JDBC_ST_STBPRP_B_3304 WITH (
        FORMAT='avro',
        KAFKA_TOPIC='jdbc_st_stbprp_b'
    );
    ```
    3. **关联测站与实时数据**
    ```sql_more=
    CREATE STREAM DBZ_ST_RIVER_R_3304_5 AS SELECT   
        B.ROWKEY,   
        B.*
    FROM JDBC_ST_STBPRP_B_3304 A 
    INNER JOIN DBZ_ST_RIVER_R B WITHIN 5 MINUTES 
    ON ((A.ROWKEY->STCD = B.ROWKEY->STCD)) 
    WHERE (B.OP <> 'd') PARTITION BY B.ROWKEY EMIT CHANGES;

    -- 过滤删除部分 因为在sink的时候会把删除部分update成主键有值其余列为空的形式

    ```
    4. **sink到目标端（仅插入更新）**
    ```sql_more
    CREATE SINK CONNECTOR `ods_rain_river_r_3304` WITH (
        "connector.class"='io.confluent.connect.jdbc.JdbcSinkConnector',
        "connection.url"='jdbc:mysql://ip地址:3308/zjslods_rain?user=username&password=passowrd',
        "insert.mode"='upsert',
        "topics"='DBZ_ST_RIVER_R_3304_AFTER_2', 
        "key.converter" = 'io.confluent.connect.avro.AvroConverter',
        "key.converter.schema.registry.url"='http://schema-registry:8081',
        "value.converter"= 'io.confluent.connect.avro.AvroConverter',
        "value.converter.schema.registry.url"='http://schema-registry:8081',
        "table.name.format"='st_river_r_3304',
        "pk.mode"='record_key',
        "pk.fields"= 'STCD,TM',
        "delete.enabled"='false'
    );
    ```
3. **将测站信息全部接入后在ksql中筛选和关联实时数据**
    ```mermaid
      flowchart  LR;
      st(开始) -->
      step1[接入测站基本信息]-->
      step2[创建对应流表]--> 
      step3[根据行政区划切分测站]-->
      step4[与实时数据关联]--> 
      step5[sink到目标端]--> 
      ed(结束)
    ```
    1. **接入测站基本信息**
    ```sql_more=
    drop CONNECTOR if exists `jdbc_st_stbprp_b_all`;
    create SOURCE CONNECTOR `jdbc_st_stbprp_b_all` WITH(
        "connector.class"='io.confluent.connect.jdbc.JdbcSourceConnector',
        "connection.url"='jdbc:mysql://192.168.2.174:3308/zjsl_rain?user=kafka_pro&password=dcxx@1234',
        "mode"='bulk',
        "incrementing.column.name"='STCD',
        "key.converter" = 'io.confluent.connect.avro.AvroConverter',
        "key.converter.schema.registry.url" = 'http://schema-registry:8081',
        "value.converter"= 'io.confluent.connect.avro.AvroConverter',
        "value.converter.schema.registry.url" = 'http://schema-registry:8081',
        "numeric.mapping"='best_fit',
        "poll.interval.ms"='360000',
        "topic.prefix"='jdbc_all_',
        "table.whitelist"='st_stbprp_b',
        "transforms"='ValueToKey',
        "transforms.ValueToKey.type"='org.apache.kafka.connect.transforms.ValueToKey',
        "transforms.ValueToKey.fields"='STCD'
        );
    ```
    2. **创建对应流表**
    ```sql_more= 
    create stream "jdbc_st_stbprp_b_all" with (kafka_topic='jdbc_all_st_stbprp_b',format='avro');
    ```
    3. **根据行政区划切分测站**
    ```sql_more=
    create stream jdbc_st_stbprp_b_3301   as select * from "jdbc_st_stbprp_b_all" where addvcd like '3301%' partition by stcd emit changes;
    ```
    4. **与实时数据关联**
    ```sql_more=
    CREATE STREAM DBZ_ST_RIVER_R_3301
    AS SELECT   
    B.ROWKEY ,   
    B.*
    FROM jdbc_st_stbprp_b_3301 A 
    INNER JOIN DBZ_ST_RIVER_R B WITHIN 5 MINUTES 
    ON ((A.ROWKEY->STCD = B.ROWKEY->STCD)) 
    WHERE (B.OP <> 'd') PARTITION BY B.ROWKEY EMIT CHANGES;
    ```
    5. **sink到目标端**
    ```sql_more=
    CREATE sink CONNECTOR `ods_rain_river_r_3301` WITH(
        "connector.class"='io.confluent.connect.jdbc.JdbcSinkConnector',
        "connection.url"='jdbc:mysql://192.168.2.174:3308/zjslods_rain?user=kafka_pro&password=dcxx@1234',
        "insert.mode"='upsert',
        "topics"='DBZ_ST_RIVER_R_3301_AFTER', 
        "key.converter" = 'io.confluent.connect.avro.AvroConverter',
        "key.converter.schema.registry.url" = 'http://schema-registry:8081',
        "value.converter"= 'io.confluent.connect.avro.AvroConverter',
        "value.converter.schema.registry.url" = 'http://schema-registry:8081',
        "table.name.format"='st_river_r_3301',
        "pk.mode"='record_key',
        "pk.fields"= 'STCD,TM',
        "delete.enabled"='false');
    ```


## PG连接器测试
1. **SOURCE CONNECTOR**   
    ```sql_more=
    DROP CONNECTOR IF EXISTS DBZ_PGZ_ZJSL_RAIN;
    CREATE SOURCE CONNECTOR dbz_pg_zjsl_rain WITH (
        "connector.class" = 'io.debezium.connector.postgresql.PostgresConnector',
        "database.hostname" = 'ip地址',
        "database.port" = '5432',
        "database.user" = 'postgres',
        "database.password" = 'postgres',
        "database.dbname" ='test_kafka',
        "plugin.name"='pgoutput',
        "key.converter"='io.confluent.connect.avro.AvroConverter',
        "key.converter.schema.registry.url"='http://schema-registry:8081',
        "value.converter"='io.confluent.connect.avro.AvroConverter',
        "value.converter.schema.registry.url"='http://schema-registry:8081',
        "database.server.name" = 'pg_river_r',
        "table.include.list"='zjsl_rain.st_river_r',
        "decimal.handling.mode"='string',
        "time.precision.mode"='connect',
        "transforms"='unwrap',    "transforms.unwrap.type"='io.debezium.transforms.ExtractNewRecordState',
        "transforms.unwrap.drop.tombstones"='false'
    );
    ```   

   
2. **SINK CONNETCOR**   
    1. **SINK PG**
    ```sql_more=
    DROP CONNECTOR IF EXISTS SINK_PG_RIVER_R;
    CREATE SINK CONNECTOR SINK_pg_river_r WITH (
        'connector.class'= 'io.confluent.connect.jdbc.JdbcSinkConnector',
        'connection.url'= 'jdbc:postgresql://192.168.2.176:5432/customers',
        'connection.user'= 'postgres',
        'connection.password'= 'postgres',
        'topics'= 'pg_river_r.zjsl_rain.st_river_r',
        'key.converter'= 'io.confluent.connect.avro.AvroConverter',
        'key.converter.schema.registry.url'   = 'http://schema-registry:8081',
        'value.converter'= 'io.confluent.connect.avro.AvroConverter',
        'value.converter.schema.registry.url' = 'http://schema-registry:8081',
        "table.name.format"='"public"."st_river_r"',
        "auto.create"='true',
        'pk.mode'= 'record_key',
        'pk.fields'= 'STCD,TM',
        'insert.mode'= 'upsert'
        );
    ```
    2. **SINK Mysql**
    ```sql_more=
    drop CONNECTOR `sink_mysql_st_river_r`;
    CREATE sink CONNECTOR `sink_mysql_st_river_r` WITH(
        "connector.class"='io.confluent.connect.jdbc.JdbcSinkConnector',
        "connection.url"='jdbc:mysql://192.168.2.174:3308/zjslods_rain?user=kafka_pro&password=dcxx@1234',
        "insert.mode"='upsert',
        "topics"='pg_river_r.zjsl_rain.st_river_r', 
        "key.converter" = 'io.confluent.connect.avro.AvroConverter',
        "key.converter.schema.registry.url" = 'http://schema-registry:8081',
        "value.converter"= 'io.confluent.connect.avro.AvroConverter',
        "value.converter.schema.registry.url" = 'http://schema-registry:8081',
        "table.name.format"='st_river_r',
        "pk.mode"='record_key',
        'pk.fields'= 'STCD,TM'
    );
    ```
