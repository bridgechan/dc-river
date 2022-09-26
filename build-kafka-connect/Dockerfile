FROM confluentinc/cp-kafka-connect:7.1.0-1-ubi8

ENV CONNECT_PLUGIN_PATH: "/usr/share/java,/usr/share/confluent-hub-components"

RUN confluent-hub install --no-prompt neo4j/kafka-connect-neo4j:2.0.2
