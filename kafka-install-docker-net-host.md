# Kafka installation with docker (Network mode: host)

## Menu
- [Lab info](#lab-info)
- [Config hosts file](#Config-hosts-file)
- [Install & Pre-config](#Install--Pre-config)

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
