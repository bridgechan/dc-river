# dc-river

# 环境准备
1. 安装docker
2. 安装docker-compose
3. 安装confluent-hub
```
wget http://client.hub.confluent.io/confluent-hub-client-latest.tar.gz
tar zxvf confluent-hub-client-latest.tar.gz -C ../confluent-hub-client/
vi ~/.bash_profile # 添加环境变量
```
4. 安装插件confluent和debezium插件
```
confluent-hub install --component-dir confluent-hub-components --worker-configs ./temp/xxx.conf --no-prompt confluentinc/kafka-connect-influxdb:latest
```
5. 编辑docker-compose的yaml文件docker-compose.yml
    - 必选：zookeeper、broker、schema-registry、ksqldb-server、ksqldb-cli
    - 可选：kafka-connect、kafkacat、rest-proxy、kafka-manager

# 部署流程
1. 拉取相关镜像并运行容器
```
docker-compose up -d
```
2. 查看错误并处理

# 开发实时流处理任务
1. 日志流接入kafka并在ksqldb中分析处理
2. 实时数据流接入kafka并分发至不同目标端
