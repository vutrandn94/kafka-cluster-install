# Kafka installation with systemd
> [!TIP]
> **Minimum requirement: 3 node controller & 3 node broker**

## Lab info (3 controller, 3 broker)
| Hostname | IP Address | OS | Role | Node ID |
| :--- | :--- | :--- | :--- | :--- |
| kafka-controller-01 | 172.31.16.254 | Ubuntu 22.04.5 LTS | controller | 1 |
| kafka-controller-02 | 172.31.17.55 | Ubuntu 22.04.5 LTS | controller | 2 |
| kafka-controller-03 | 172.31.31.137 | Ubuntu 22.04.5 LTS | controller | 3 |
| kafka-broker-01 | 172.31.31.153 | Ubuntu 22.04.5 LTS | broker | 4 |
| kafka-broker-02 | 172.31.16.244 | Ubuntu 22.04.5 LTS | broker | 5 |
| kafka-broker-03 | 172.31.16.215 | Ubuntu 22.04.5 LTS | broker | 6 |

## Config hosts file on all nodes (controller & broker)
```
# vi /etc/hosts

172.31.16.254 kafka-01 kafka-controller-01
172.31.17.55 kafka-02 kafka-controller-02
172.31.31.137 kafka-03 kafka-controller-03
172.31.31.153 kafka-04 kafka-broker-01
172.31.16.244 kafka-05 kafka-broker-02
172.31.16.215 kafka-06 kafka-broker-03
```

## Install & pre-config on all nodes (controller & broker)
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

**Prepare package components:**
```
# apt update && apt install openjdk-17-jdk -y
# useradd kafka -m
# cd /opt
# wget https://downloads.apache.org/kafka/4.1.1/kafka_2.13-4.1.1.tgz
# tar -xvf kafka_2.13-4.1.1.tgz
# mv kafka_2.13-4.1.1 kafka
# chown -R kafka:kafka /opt/kafka
# mkdir -p /var/lib/kafka/controller/{data,logs} /var/lib/kafka/broker/{data,logs}
# chown -R kafka:kafka /var/lib/kafka
```

## Config controller nodes
### On node kafka-controller-01
**Generate Cluster ID (only generate once and reuse for all remain nodes)**
```
# /opt/kafka/bin/kafka-storage.sh random-uuid

421j_LOSTrioyErqtEZMVA
```

> [!NOTE]
> **Cluster ID only generate once and reuse for all remain nodes**

**Apply controller config:**
```
# mv /opt/kafka/config/controller.properties /opt/kafka/config/controller.properties.ori
```

```
# vi /opt/kafka/config/controller.properties

listeners=CONTROLLER://:9094
listener.security.protocol.map=INTERNAL:SASL_PLAINTEXT,CONTROLLER:PLAINTEXT
process.roles=controller
node.id=1
controller.listener.names=CONTROLLER
controller.quorum.voters=1@kafka-controller-01:9094,2@kafka-controller-02:9094,3@kafka-controller-03:9094
controller.quorum.bootstrap.servers=kafka-controller-01:9094,kafka-controller-02:9094,kafka-controller-03:9094
log.dir=/var/lib/kafka/controller/data
logs.dir=/var/lib/kafka/controller/logs
num.network.threads=3
num.io.threads=8
inter.broker.listener.name=INTERNAL
group.initial.rebalance.delay.ms=0
offsets.topic.replication.factor=3
default.replication.factor=3
min.insync.replicas=2
```

```
# chown kafka:kafka /opt/kafka/config/controller.properties
```

```
# /opt/kafka/bin/kafka-storage.sh format --config /opt/kafka/config/controller.properties --cluster-id <CLUSTER ID>

Example: 
# /opt/kafka/bin/kafka-storage.sh format --config /opt/kafka/config/controller.properties --cluster-id 421j_LOSTrioyErqtEZMVA
Formatting metadata directory /var/lib/kafka/controller/data with metadata.version 4.1-IV1.
```

**Defined kafka-control systemd service:**
```
# vi /etc/systemd/system/kafka-controller.service

[Unit]
Description=Kafka KRaft Controller
After=network.target

[Service]
Environment="KAFKA_OPTS=-Dcom.sun.management.jmxremote \
 -Dcom.sun.management.jmxremote.port=9999 \
 -Dcom.sun.management.jmxremote.rmi.port=9999 \
 -Dcom.sun.management.jmxremote.authenticate=false \
 -Dcom.sun.management.jmxremote.ssl=false \
 -Djava.rmi.server.hostname=kafka-controller-01"
User=kafka
WorkingDirectory=/opt/kafka
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/controller.properties
Restart=on-failure
LimitNOFILE=100000

[Install]
WantedBy=multi-user.target
```

```
# systemctl daemon-reload

# systemctl start kafka-controller && systemctl enable kafka-controller

# systemctl status kafka-controller
● kafka-controller.service - Kafka KRaft Controller
     Loaded: loaded (/etc/systemd/system/kafka-controller.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2025-11-19 04:06:36 UTC; 36s ago
   Main PID: 4865 (java)
      Tasks: 52 (limit: 65000)
     Memory: 208.0M
        CPU: 5.743s
     CGroup: /system.slice/kafka-controller.service
             └─4865 java -Xmx1G -Xms1G -server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+ExplicitGCInvokesConcurrent -XX:MaxInlineLevel=15 -Djava.awt.headless=true "-Xlog>

Nov 19 04:07:11 kafka-controller-01 kafka-server-start.sh[4865]: [2025-11-19 04:07:11,564] INFO [MetadataLoader id=1] initializeNewPublishers: the loader is still catching up because we still don't know the h>
Nov 19 04:07:11 kafka-controller-01 kafka-server-start.sh[4865]: [2025-11-19 04:07:11,664] INFO [MetadataLoader id=1] initializeNewPublishers: the loader is still catching up because we still don't know the h>
Nov 19 04:07:11 kafka-controller-01 kafka-server-start.sh[4865]: [2025-11-19 04:07:11,739] INFO [RaftManager id=1] Node 2 disconnected. (org.apache.kafka.clients.NetworkClient)
Nov 19 04:07:11 kafka-controller-01 kafka-server-start.sh[4865]: [2025-11-19 04:07:11,739] WARN [RaftManager id=1] Connection to node 2 (kafka-controller-02/172.31.17.55:9094) could not be established. Node m>
Nov 19 04:07:11 kafka-controller-01 kafka-server-start.sh[4865]: [2025-11-19 04:07:11,739] INFO [RaftManager id=1] Node 3 disconnected. (org.apache.kafka.clients.NetworkClient)
Nov 19 04:07:11 kafka-controller-01 kafka-server-start.sh[4865]: [2025-11-19 04:07:11,739] WARN [RaftManager id=1] Connection to node 3 (kafka-controller-03/172.31.31.137:9094) could not be established. Node >
Nov 19 04:07:11 kafka-controller-01 kafka-server-start.sh[4865]: [2025-11-19 04:07:11,764] INFO [MetadataLoader id=1] initializeNewPublishers: the loader is still catching up because we still don't know the h>
Nov 19 04:07:11 kafka-controller-01 kafka-server-start.sh[4865]: [2025-11-19 04:07:11,865] INFO [MetadataLoader id=1] initializeNewPublishers: the loader is still catching up because we still don't know the h>
Nov 19 04:07:11 kafka-controller-01 kafka-server-start.sh[4865]: [2025-11-19 04:07:11,965] INFO [MetadataLoader id=1] initializeNewPublishers: the loader is still catching up because we still don't know the h>
Nov 19 04:07:12 kafka-controller-01 kafka-server-start.sh[4865]: [2025-11-19 04:07:12,066] INFO [MetadataLoader id=1] initializeNewPublishers: the loader is still catching up because we still don't know the h>
```

### On node kafka-controller-02
**Apply controller config:**
```
# mv /opt/kafka/config/controller.properties /opt/kafka/config/controller.properties.ori
```

```
# vi /opt/kafka/config/controller.properties

listeners=CONTROLLER://:9094
listener.security.protocol.map=INTERNAL:SASL_PLAINTEXT,CONTROLLER:PLAINTEXT
process.roles=controller
node.id=2
controller.listener.names=CONTROLLER
controller.quorum.voters=1@kafka-controller-01:9094,2@kafka-controller-02:9094,3@kafka-controller-03:9094
controller.quorum.bootstrap.servers=kafka-controller-01:9094,kafka-controller-02:9094,kafka-controller-03:9094
log.dir=/var/lib/kafka/controller/data
logs.dir=/var/lib/kafka/controller/logs
num.network.threads=3
num.io.threads=8
inter.broker.listener.name=INTERNAL
group.initial.rebalance.delay.ms=0
offsets.topic.replication.factor=3
default.replication.factor=3
min.insync.replicas=2
```

> [!NOTE]
> **Keep all configs on kafka-controller-01. Only increase "node.id"**

```
# chown kafka:kafka /opt/kafka/config/controller.properties
```

```
# /opt/kafka/bin/kafka-storage.sh format --config /opt/kafka/config/controller.properties --cluster-id <CLUSTER ID>

Example:
# /opt/kafka/bin/kafka-storage.sh format --config /opt/kafka/config/controller.properties --cluster-id 421j_LOSTrioyErqtEZMVA
Formatting metadata directory /var/lib/kafka/controller/data with metadata.version 4.1-IV1.
```

**Defined kafka-control systemd service:**
```
# vi /etc/systemd/system/kafka-controller.service

[Unit]
Description=Kafka KRaft Controller
After=network.target

[Service]
Environment="KAFKA_OPTS=-Dcom.sun.management.jmxremote \
 -Dcom.sun.management.jmxremote.port=9999 \
 -Dcom.sun.management.jmxremote.rmi.port=9999 \
 -Dcom.sun.management.jmxremote.authenticate=false \
 -Dcom.sun.management.jmxremote.ssl=false \
 -Djava.rmi.server.hostname=kafka-controller-02"
User=kafka
WorkingDirectory=/opt/kafka
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/controller.properties
Restart=on-failure
LimitNOFILE=100000

[Install]
WantedBy=multi-user.target
```

```
```
# systemctl daemon-reload

# systemctl start kafka-controller && systemctl enable kafka-controller

# systemctl status kafka-controller
● kafka-controller.service - Kafka KRaft Controller
     Loaded: loaded (/etc/systemd/system/kafka-controller.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2025-11-19 04:14:02 UTC; 4s ago
   Main PID: 3246 (java)
      Tasks: 52 (limit: 65000)
     Memory: 224.3M
        CPU: 5.433s
     CGroup: /system.slice/kafka-controller.service
             └─3246 java -Xmx1G -Xms1G -server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+ExplicitGCInvokesConcurrent -XX:MaxInlineLevel=15 -Djava.awt.headless=true "-Xlog>

Nov 19 04:14:05 kafka-controller-02 kafka-server-start.sh[3246]:  (org.apache.kafka.common.config.AbstractConfig)
Nov 19 04:14:05 kafka-controller-02 kafka-server-start.sh[3246]: [2025-11-19 04:14:05,734] INFO [ControllerRegistrationManager id=2 incarnation=DkfHoW8ESHC2iyI7XRV2KA] sendControllerRegistration: attempting t>
Nov 19 04:14:05 kafka-controller-02 kafka-server-start.sh[3246]: [2025-11-19 04:14:05,749] INFO [MetadataLoader id=2] InitializeNewPublishers: initializing DynamicClientQuotaPublisher controller id=2 with a s>
Nov 19 04:14:05 kafka-controller-02 kafka-server-start.sh[3246]: [2025-11-19 04:14:05,754] INFO [MetadataLoader id=2] InitializeNewPublishers: initializing DynamicTopicClusterQuotaPublisher controller id=2 wi>
Nov 19 04:14:05 kafka-controller-02 kafka-server-start.sh[3246]: [2025-11-19 04:14:05,755] INFO [MetadataLoader id=2] InitializeNewPublishers: initializing ScramPublisher controller id=2 with a snapshot at of>
Nov 19 04:14:05 kafka-controller-02 kafka-server-start.sh[3246]: [2025-11-19 04:14:05,757] INFO [MetadataLoader id=2] InitializeNewPublishers: initializing DelegationTokenPublisher controller id=2 with a snap>
Nov 19 04:14:05 kafka-controller-02 kafka-server-start.sh[3246]: [2025-11-19 04:14:05,760] INFO [MetadataLoader id=2] InitializeNewPublishers: initializing ControllerMetadataMetricsPublisher with a snapshot a>
Nov 19 04:14:05 kafka-controller-02 kafka-server-start.sh[3246]: [2025-11-19 04:14:05,761] INFO [MetadataLoader id=2] InitializeNewPublishers: initializing AclPublisher controller id=2 with a snapshot at offs>
Nov 19 04:14:05 kafka-controller-02 kafka-server-start.sh[3246]: [2025-11-19 04:14:05,793] INFO [ControllerRegistrationManager id=2 incarnation=DkfHoW8ESHC2iyI7XRV2KA] Our registration has been persisted to t>
Nov 19 04:14:05 kafka-controller-02 kafka-server-start.sh[3246]: [2025-11-19 04:14:05,798] INFO [ControllerRegistrationManager id=2 incarnation=DkfHoW8ESHC2iyI7XRV2KA] RegistrationResponseHandler: controller 
```

### On node kafka-controller-03
**Apply controller config:**
```
# mv /opt/kafka/config/controller.properties /opt/kafka/config/controller.properties.ori
```

```
# vi /opt/kafka/config/controller.properties

listeners=CONTROLLER://:9094
listener.security.protocol.map=INTERNAL:SASL_PLAINTEXT,CONTROLLER:PLAINTEXT
process.roles=controller
node.id=3
controller.listener.names=CONTROLLER
controller.quorum.voters=1@kafka-controller-01:9094,2@kafka-controller-02:9094,3@kafka-controller-03:9094
controller.quorum.bootstrap.servers=kafka-controller-01:9094,kafka-controller-02:9094,kafka-controller-03:9094
log.dir=/var/lib/kafka/controller/data
logs.dir=/var/lib/kafka/controller/logs
num.network.threads=3
num.io.threads=8
inter.broker.listener.name=INTERNAL
group.initial.rebalance.delay.ms=0
offsets.topic.replication.factor=3
default.replication.factor=3
min.insync.replicas=2
```

> [!NOTE]
> **Keep all configs on kafka-controller-02. Only increase "node.id"**

```
# chown kafka:kafka /opt/kafka/config/controller.properties
```

```
# /opt/kafka/bin/kafka-storage.sh format --config /opt/kafka/config/controller.properties --cluster-id <CLUSTER ID>

Example:
# /opt/kafka/bin/kafka-storage.sh format --config /opt/kafka/config/controller.properties --cluster-id 421j_LOSTrioyErqtEZMVA
Formatting metadata directory /var/lib/kafka/controller/data with metadata.version 4.1-IV1.
```

**Defined kafka-control systemd service:**
```
# vi /etc/systemd/system/kafka-controller.service

[Unit]
Description=Kafka KRaft Controller
After=network.target

[Service]
Environment="KAFKA_OPTS=-Dcom.sun.management.jmxremote \
 -Dcom.sun.management.jmxremote.port=9999 \
 -Dcom.sun.management.jmxremote.rmi.port=9999 \
 -Dcom.sun.management.jmxremote.authenticate=false \
 -Dcom.sun.management.jmxremote.ssl=false \
 -Djava.rmi.server.hostname=kafka-controller-03"
User=kafka
WorkingDirectory=/opt/kafka
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/controller.properties
Restart=on-failure
LimitNOFILE=100000

[Install]
WantedBy=multi-user.target
```

```
```
# systemctl daemon-reload

# systemctl start kafka-controller && systemctl enable kafka-controller

# systemctl status kafka-controller
● kafka-controller.service - Kafka KRaft Controller
     Loaded: loaded (/etc/systemd/system/kafka-controller.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2025-11-19 04:18:58 UTC; 4s ago
   Main PID: 3334 (java)
      Tasks: 52 (limit: 65000)
     Memory: 220.9M
        CPU: 4.953s
     CGroup: /system.slice/kafka-controller.service
             └─3334 java -Xmx1G -Xms1G -server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+ExplicitGCInvokesConcurrent -XX:MaxInlineLevel=15 -Djava.awt.headless=true "-Xlog>

Nov 19 04:19:02 kafka-controller-03 kafka-server-start.sh[3334]:         unstable.feature.versions.enable = false
Nov 19 04:19:02 kafka-controller-03 kafka-server-start.sh[3334]:  (org.apache.kafka.common.config.AbstractConfig)
Nov 19 04:19:02 kafka-controller-03 kafka-server-start.sh[3334]: [2025-11-19 04:19:02,134] INFO [MetadataLoader id=3] InitializeNewPublishers: initializing DynamicClientQuotaPublisher controller id=3 with a s>
Nov 19 04:19:02 kafka-controller-03 kafka-server-start.sh[3334]: [2025-11-19 04:19:02,135] INFO [MetadataLoader id=3] InitializeNewPublishers: initializing DynamicTopicClusterQuotaPublisher controller id=3 wi>
Nov 19 04:19:02 kafka-controller-03 kafka-server-start.sh[3334]: [2025-11-19 04:19:02,136] INFO [MetadataLoader id=3] InitializeNewPublishers: initializing ScramPublisher controller id=3 with a snapshot at of>
Nov 19 04:19:02 kafka-controller-03 kafka-server-start.sh[3334]: [2025-11-19 04:19:02,137] INFO [MetadataLoader id=3] InitializeNewPublishers: initializing DelegationTokenPublisher controller id=3 with a snap>
Nov 19 04:19:02 kafka-controller-03 kafka-server-start.sh[3334]: [2025-11-19 04:19:02,139] INFO [MetadataLoader id=3] InitializeNewPublishers: initializing ControllerMetadataMetricsPublisher with a snapshot a>
Nov 19 04:19:02 kafka-controller-03 kafka-server-start.sh[3334]: [2025-11-19 04:19:02,140] INFO [MetadataLoader id=3] InitializeNewPublishers: initializing AclPublisher controller id=3 with a snapshot at offs>
Nov 19 04:19:02 kafka-controller-03 kafka-server-start.sh[3334]: [2025-11-19 04:19:02,171] INFO [ControllerRegistrationManager id=3 incarnation=LEPEIxz_Sf-x7KyyPypmdA] RegistrationResponseHandler: controller >
Nov 19 04:19:02 kafka-controller-03 kafka-server-start.sh[3334]: [2025-11-19 04:19:02,174] INFO [ControllerRegistrationManager id=3 incarnation=LEPEIxz_Sf-x7KyyPypmdA] Our registration has been persisted to t
```
