---
title: etcd集群部署指南
date: 2026-10-12 16:57:10
tags:
  - Kubernetes
categories:
  - K8S
description: 从零搭建高可用 etcd 集群，涵盖 cfssl 证书签发、节点配置、集群启动与健康检查，适用于 Kubernetes 控制平面基础设施部署。
---

本文记录了在 Linux 环境下使用 `cfssl` 工具从零开始签发证书并部署高可用 `etcd` 集群的完整步骤。

<!-- more -->

## 一、证书签发与配置

### 1. 安装 cfssl 工具

```shell
curl -s -L -o /usr/bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 
curl -s -L -o /usr/bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 
curl -s -L -o /usr/bin/cfssl-certinfo https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x /usr/bin/cfssl*
```

### 2. 签发 CA 证书

创建 CA 配置文件 `ca-csr.json`：

```shell
cat > ca-csr.json << EOF
{
    "CN": "kubernetes-ca",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
        "C": "CN",
        "ST": "GD",
        "L": "GZ",
        "O": "K8S",
        "OU": "OP"
        }
    ],
    "ca": {
        "expiry": "876000h"
    }
}
EOF
```

生成 CA 证书：

```shell
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

### 3. 创建 CA 证书配置文件

创建 `ca-config.json`：

```shell
cat > ca-config.json << EOF
{
    "signing": {
        "default": {
            "expiry": "876000h"
        },
        "profiles": {
            "server": {
                "expiry": "876000h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth"
                ]
            },
            "client": {
                "expiry": "876000h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "client auth"
                ]
            },
            "peer": {
                "expiry": "876000h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
EOF
```

### 4. 签发 etcd 证书

创建 etcd 证书请求文件 `etcd-peer-csr.json`（请根据实际集群节点的 IP 地址进行修改）：

```shell
cat > etcd-peer-csr.json << EOF
{
    "CN": "k8s-etcd",
    "hosts": [
        "127.0.0.1",
        "192.168.19.100",
        "192.168.19.101",
        "192.168.19.102",
        "192.168.19.103"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
        "C": "CN",
        "ST": "GD",
        "L": "GZ",
        "O": "K8S",
        "OU": "OP"
        }
    ]
}
EOF
```

生成 etcd 证书：

```shell
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=peer etcd-peer-csr.json | cfssljson -bare etcd-peer
```

> **注意**：生成完成后，需要将 `ca.pem`、`etcd-peer-key.pem`、`etcd-peer.pem` 拷贝到其它节点的 `/etc/etcd/ssl/` 目录下。

---

## 二、部署 etcd 服务

### 1. 下载并安装 etcd

```shell
curl -L https://github.com/coreos/etcd/releases/download/v3.5.1/etcd-v3.5.1-linux-amd64.tar.gz -o etcd-v3.5.1-linux-amd64.tar.gz
tar xf etcd-v3.5.1-linux-amd64.tar.gz
cp etcd* /usr/bin/
```

### 2. 配置 etcd 系统服务

创建 systemd 服务文件 `/usr/lib/systemd/system/etcd.service`：

```ini
cat > /usr/lib/systemd/system/etcd.service << EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=/etc/etcd/etcd.conf
ExecStart=/usr/bin/etcd \
    --cert-file=/etc/etcd/ssl/etcd-peer.pem \
    --key-file=/etc/etcd/ssl/etcd-peer-key.pem \
    --peer-cert-file=/etc/etcd/ssl/etcd-peer.pem \
    --peer-key-file=/etc/etcd/ssl/etcd-peer-key.pem \
    --trusted-ca-file=/etc/etcd/ssl/ca.pem \
    --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
    --logger=zap
Restart=on-failure
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

### 3. 修改 etcd 配置文件

在 `/etc/etcd/` 目录下创建配置文件 `etcd.conf`（注意修改各节点对应的 IP 与主机名）：

```ini
cat > etcd.conf <<EOF
#[Member]
ETCD_NAME="node01"
ETCD_DATA_DIR="/var/lib/etcd/"
ETCD_LISTEN_PEER_URLS="https://192.168.22.101:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.22.101:2379,https://127.0.0.1:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.22.101:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.22.101:2379,https://127.0.0.1:2379"
ETCD_INITIAL_CLUSTER="node01=https://192.168.22.101:2380,node02=https://192.168.22.102:2380,node03=https://192.168.22.103:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF
```

---

## 三、常用运维命令

集群启动后，可以使用以下命令检查集群状态和成员列表：

```shell
ENDPOINTS=https://192.168.22.101:2379,https://192.168.22.102:2379,https://192.168.22.103:2379

# 查看集群成员列表
ETCDCTL_API=3 etcdctl --endpoints=$ENDPOINTS --cacert=/usr/local/etcd/ssl/ca.pem --cert=/usr/local/etcd/ssl/etcd-peer.pem --key=/usr/local/etcd/ssl/etcd-peer-key.pem member list --write-out=table

# 检查集群端点状态
etcdctl --endpoints=$ENDPOINTS --cacert=/usr/local/etcd/ssl/ca.pem --cert=/usr/local/etcd/ssl/etcd-peer.pem --key=/usr/local/etcd/ssl/etcd-peer-key.pem endpoint status --write-out=table
```
