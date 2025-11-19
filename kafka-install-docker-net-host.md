# Kafka installation with docker (Network mode: host)

## Menu
- [Lab info](#lab-info)
- [Config hosts file](#Config-hosts-file)
- [Install & Pre-config](#Install--Pre-config)
- [Config on node kafka-01](#Config-on-node-kafka-01)
- [Config on node kafka-02](#Config-on-node-kafka-02)
- [Config on node kafka-03](#Config-on-node-kafka-03)

> [!TIP]
> Minimum requirement: 3 node (Combine between controller role and broker role), docker & docker-compose installed on server.
## Lab info (3 node)
| Hostname | IP Address | OS | Role | Node ID | Proxy Public IP |
| :--- | :--- | :--- | :--- | :--- | :--- |
| kafka-01 | 172.31.16.254 | Ubuntu 22.04.5 LTS | combine (controller & broker) | 1 | 52.221.180.79 |
| kafka-02 | 172.31.17.55 | Ubuntu 22.04.5 LTS | combine (controller & broker) | 2 | 52.221.180.79 |
| kafka-03 | 172.31.31.137 | Ubuntu 22.04.5 LTS | combine (controller & broker) | 3 | 52.221.180.79 |

## Config hosts file on all nodes
```
# vi /etc/hosts

172.31.16.254 kafka-01
172.31.17.55 kafka-02
172.31.31.137 kafka-03
```

## Install & pre-config on all nodes
**Change default values system limit**
```
# vi /etc/systemd/system.conf

DefaultLimitNOFILE=65000
DefaultLimitNPROC=65000
DefaultTasksMax=65000
```

**Config sysctl.conf**
```
# vi /etc/sysctl.conf

net.ipv4.ip_forward=1
kernel.randomize_va_space=2
fs.suid_dumpable=0
kernel.keys.root_maxbytes=25000000
kernel.keys.root_maxkeys=1000000
kernel.panic=10
kernel.panic_on_oops=1
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=524288
fs.inotify.max_user_instances=1024
```

```
# sysctl -p --system
```

**Define "kafka_server_jaas.conf" to set authentication for Kafka cluster**
```
# vi kafka_server_jaas.conf

KafkaServer {
  org.apache.kafka.common.security.plain.PlainLoginModule required
  username="admin"
  password="Enjoyd@y2025"
  user_admin="Enjoyd@y2025";
};
```

## Config on node kafka-01
**Generate JMX config for each kafka node**
> [!TIP]
> **"jmx_config_kafka-template.yml" attached on repository**

```
# for host in kafka-01; do cp jmx_config_kafka-template.yml jmx_config_$host.yml; sed -i "s/^hostPort: <KAFKA_HOST>:<KAFKA_JMX_PORT>$/hostPort: ${host}:9999/" jmx_config_$host.yml; done
```

**Define docker-compose.yaml**
```
services:
  kafka-01:
    image: confluentinc/cp-kafka:8.0.2
    hostname: kafka-01
    container_name: kafka-01
    network_mode: host
    user: "0:0"
    restart: always
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_BROKER_ID: 1
      KAFKA_PROCESS_ROLES: 'broker,controller'
      KAFKA_CONTROLLER_QUORUM_VOTERS: '1@kafka-01:9094,2@kafka-02:9094,3@kafka-03:9094'
      KAFKA_LISTENERS: 'CLIENT://:9092,INTERNAL://:9093,CONTROLLER://:9094'
      KAFKA_ADVERTISED_LISTENERS: 'INTERNAL://:9093,CLIENT://52.221.180.79:9092'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: 'CLIENT:SASL_PLAINTEXT,INTERNAL:SASL_PLAINTEXT,CONTROLLER:PLAINTEXT'
      KAFKA_CONTROLLER_LISTENER_NAMES: 'CONTROLLER'
      KAFKA_INTER_BROKER_LISTENER_NAME: 'INTERNAL'
      CLUSTER_ID: 'a50a310c-c42f-11f0-9c0d-02f2d2730ed1'
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2
      KAFKA_SASL_ENABLED_MECHANISMS: 'PLAIN'
      KAFKA_SASL_MECHANISM_INTER_BROKER_PROTOCOL: 'PLAIN'
      KAFKA_OPTS: "-Djava.security.auth.login.config=/etc/kafka/kafka_server_jaas.conf"
      KAFKA_JMX_OPTS: "-Dcom.sun.management.jmxremote=true -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.port=9999 -Dcom.sun.management.jmxremote.rmi.port=9999 -Djava.rmi.server.hostname=kafka-01"
    volumes:
      - ./data:/var/lib/kafka/data
      - ./kafka_server_jaas.conf:/etc/kafka/kafka_server_jaas.conf:ro
  kafka-01-jmx:
    image: bitnamilegacy/jmx-exporter:1.3.0-debian-12-r1
    container_name: kafka-01-jmx
    hostname: kafka-01-jmx
    network_mode: host
    restart: always
    volumes:
      - ./jmx_config_kafka-01.yml:/opt/bitnami/jmx-exporter/examples/standalone_sample_config.yml:ro
    depends_on:
      - kafka-01
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-cluster-ui
    network_mode: host
    restart: always
    environment:
      KAFKA_CLUSTERS_0_NAME: 'Test'
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: 'kafka-01:9093,kafka-02:9093,kafka-03:9093,kafka-04:9093'
      KAFKA_CLUSTERS_0_PROPERTIES_SASL_JAAS_CONFIG: "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"admin\" password=\"VUTD@123\";"
      KAFKA_CLUSTERS_0_PROPERTIES_SASL_MECHANISM: 'PLAIN'
      KAFKA_CLUSTERS_0_PROPERTIES_SECURITY_PROTOCOL: 'SASL_PLAINTEXT'
      MANAGEMENT_HEALTH_LDAP_ENABLED: 'FALSE'
    depends_on:
      - kafka-01
```

**Start service**
```
# docker-compose up -d
[+] Running 3/3
 ✔ Container kafka-01          Started                                                                                                                                                                      0.4s 
 ✔ Container kafka-cluster-ui  Started                                                                                                                                                                      0.4s 
 ✔ Container kafka-01-jmx      Started
```

## Config on node kafka-02
**Generate JMX config for each kafka node**
> [!TIP]
> **"jmx_config_kafka-template.yml" attached on repository**

```
# for host in kafka-02; do cp jmx_config_kafka-template.yml jmx_config_$host.yml; sed -i "s/^hostPort: <KAFKA_HOST>:<KAFKA_JMX_PORT>$/hostPort: ${host}:9999/" jmx_config_$host.yml; done
```

**Define docker-compose.yaml**
```
services:
  kafka-02:
    image: confluentinc/cp-kafka:8.0.2
    hostname: kafka-02
    container_name: kafka-02
    network_mode: host
    user: "0:0"
    restart: always
    environment:
      KAFKA_NODE_ID: 2
      KAFKA_BROKER_ID: 2
      KAFKA_PROCESS_ROLES: 'broker,controller'
      KAFKA_CONTROLLER_QUORUM_VOTERS: '1@kafka-01:9094,2@kafka-02:9094,3@kafka-03:9094'
      KAFKA_LISTENERS: 'CLIENT://:9092,INTERNAL://:9093,CONTROLLER://:9094'
      KAFKA_ADVERTISED_LISTENERS: 'INTERNAL://:9093,CLIENT://52.221.180.79:9092'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: 'CLIENT:SASL_PLAINTEXT,INTERNAL:SASL_PLAINTEXT,CONTROLLER:PLAINTEXT'
      KAFKA_CONTROLLER_LISTENER_NAMES: 'CONTROLLER'
      KAFKA_INTER_BROKER_LISTENER_NAME: 'INTERNAL'
      CLUSTER_ID: 'a50a310c-c42f-11f0-9c0d-02f2d2730ed1'
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2
      KAFKA_SASL_ENABLED_MECHANISMS: 'PLAIN'
      KAFKA_SASL_MECHANISM_INTER_BROKER_PROTOCOL: 'PLAIN'
      KAFKA_OPTS: "-Djava.security.auth.login.config=/etc/kafka/kafka_server_jaas.conf"
      KAFKA_JMX_OPTS: "-Dcom.sun.management.jmxremote=true -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.port=9999 -Dcom.sun.management.jmxremote.rmi.port=9999 -Djava.rmi.server.hostname=kafka-02"
    volumes:
      - ./data:/var/lib/kafka/data
      - ./kafka_server_jaas.conf:/etc/kafka/kafka_server_jaas.conf:ro
  kafka-02-jmx:
    image: bitnamilegacy/jmx-exporter:1.3.0-debian-12-r1
    container_name: kafka-02-jmx
    hostname: kafka-02-jmx
    network_mode: host
    restart: always
    volumes:
      - ./jmx_config_kafka-02.yml:/opt/bitnami/jmx-exporter/examples/standalone_sample_config.yml:ro
    depends_on:
      - kafka-02
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-cluster-ui
    network_mode: host
    restart: always
    environment:
      KAFKA_CLUSTERS_0_NAME: 'Test'
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: 'kafka-01:9093,kafka-02:9093,kafka-03:9093'
      KAFKA_CLUSTERS_0_PROPERTIES_SASL_JAAS_CONFIG: "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"admin\" password=\"Enjoyd@y2025\";"
      KAFKA_CLUSTERS_0_PROPERTIES_SASL_MECHANISM: 'PLAIN'
      KAFKA_CLUSTERS_0_PROPERTIES_SECURITY_PROTOCOL: 'SASL_PLAINTEXT'
      MANAGEMENT_HEALTH_LDAP_ENABLED: 'FALSE'
    depends_on:
      - kafka-02
```

**Start service**
```
# docker-compose up -d
[+] Running 3/3
 ✔ Container kafka-02          Started                                                                                                                                                                      0.4s 
 ✔ Container kafka-cluster-ui  Started                                                                                                                                                                      0.4s 
 ✔ Container kafka-02-jmx      Started
```

## Config on node kafka-03
**Generate JMX config for each kafka node**
> [!TIP]
> **"jmx_config_kafka-template.yml" attached on repository**

```
# for host in kafka-03; do cp jmx_config_kafka-template.yml jmx_config_$host.yml; sed -i "s/^hostPort: <KAFKA_HOST>:<KAFKA_JMX_PORT>$/hostPort: ${host}:9999/" jmx_config_$host.yml; done
```

**Define docker-compose.yaml**
```
services:
  kafka-03:
    image: confluentinc/cp-kafka:8.0.2
    hostname: kafka-03
    container_name: kafka-03
    network_mode: host
    user: "0:0"
    restart: always
    environment:
      KAFKA_NODE_ID: 3
      KAFKA_BROKER_ID: 3
      KAFKA_PROCESS_ROLES: 'broker,controller'
      KAFKA_CONTROLLER_QUORUM_VOTERS: '1@kafka-01:9094,2@kafka-02:9094,3@kafka-03:9094'
      KAFKA_LISTENERS: 'CLIENT://:9092,INTERNAL://:9093,CONTROLLER://:9094'
      KAFKA_ADVERTISED_LISTENERS: 'INTERNAL://:9093,CLIENT://52.221.180.79:9092'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: 'CLIENT:SASL_PLAINTEXT,INTERNAL:SASL_PLAINTEXT,CONTROLLER:PLAINTEXT'
      KAFKA_CONTROLLER_LISTENER_NAMES: 'CONTROLLER'
      KAFKA_INTER_BROKER_LISTENER_NAME: 'INTERNAL'
      CLUSTER_ID: 'a50a310c-c42f-11f0-9c0d-02f2d2730ed1'
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2
      KAFKA_SASL_ENABLED_MECHANISMS: 'PLAIN'
      KAFKA_SASL_MECHANISM_INTER_BROKER_PROTOCOL: 'PLAIN'
      KAFKA_OPTS: "-Djava.security.auth.login.config=/etc/kafka/kafka_server_jaas.conf"
      KAFKA_JMX_OPTS: "-Dcom.sun.management.jmxremote=true -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.port=9999 -Dcom.sun.management.jmxremote.rmi.port=9999 -Djava.rmi.server.hostname=kafka-03"
    volumes:
      - ./data:/var/lib/kafka/data
      - ./kafka_server_jaas.conf:/etc/kafka/kafka_server_jaas.conf:ro
  kafka-03-jmx:
    image: bitnamilegacy/jmx-exporter:1.3.0-debian-12-r1
    container_name: kafka-03-jmx
    hostname: kafka-03-jmx
    network_mode: host
    restart: always
    volumes:
      - ./jmx_config_kafka-03.yml:/opt/bitnami/jmx-exporter/examples/standalone_sample_config.yml:ro
    depends_on:
      - kafka-03
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-cluster-ui
    network_mode: host
    restart: always
    environment:
      KAFKA_CLUSTERS_0_NAME: 'Test'
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: 'kafka-01:9093,kafka-02:9093,kafka-03:9093'
      KAFKA_CLUSTERS_0_PROPERTIES_SASL_JAAS_CONFIG: "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"admin\" password=\"Enjoyd@y2025\";"
      KAFKA_CLUSTERS_0_PROPERTIES_SASL_MECHANISM: 'PLAIN'
      KAFKA_CLUSTERS_0_PROPERTIES_SECURITY_PROTOCOL: 'SASL_PLAINTEXT'
      MANAGEMENT_HEALTH_LDAP_ENABLED: 'FALSE'
    depends_on:
      - kafka-03
```

**Start service**
```
# docker-compose up -d

[+] Running 3/3
 ✔ Container kafka-03          Started                                                                                                                                                                      0.4s 
 ✔ Container kafka-cluster-ui  Started                                                                                                                                                                      0.4s 
 ✔ Container kafka-03-jmx      Started  
```

## Kafka UI expose info

| URL ACCESS |
| :--- |
| http://172.31.16.254:8080 |
| http://172.31.17.55:8080 |
| http://172.31.31.137:8080 |

## Kafka expose Kafka JMX metrics
| KAFKA NODE | URL ACCESS |
| :--- | :--- |
| kafka-01 | http://172.31.16.254:5556/metrics |
| kafka-02 | http://172.31.17.55:5556/metrics |
| kafka-03 | http://172.31.31.137:5556/metrics |
