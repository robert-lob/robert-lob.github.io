---
title: K8S 二进制部署
categories: 
- linux
- 应用部署
tags: 
- 未完成
---

## 高可用
Master的kube-apiserver、kube-controller-manager和kube-scheduler服务至少有3个节点
Master启用基于CA认证的HTTPS安全机制
etcd至少3个节点
etcd集群启用基于CA认证的HTTPS安全机制
Master启用RBAC授权

## 设备准备
三台服务器
系统：Centos 7 2009
IP：
  192.168.100.123
  192.168.100.124
  192.168.100.125
VIP：
  192。168.100.251
依赖：
  openssl

## 创建CA证书
可以使用openssl、easyrsa、cfssl等工具完成
```shell
openssl genrsa -out ca.key 2048
# subj：“/CN”的值为master主机名或者IP
# day：设置证书有效期
openssl req -x509 -new -nodes -key ca.key -subj "/CN=192.168.100.123" -days 36500 -out ca.crt
mv ca.* /etc/kubernetes/pki/
```

## 部署etcd集群

# 下载etcd二进制文件，配置systemd服务
```shell
tar zxvf etcd-v3.5.1-linux-amd64.tar.gz
mv etcd etcdctl -t /usr/bin
vi /usr/lib/systemd/system/etcd.service
```

```shell
[Unit]
Description=etcd key-value store
Documentation=https://github.com/etcd-io/etcd
After=network.target

[Service]
EnvironmentFile=/etc/etcd/etcd.conf
ExecStart=/usr/bin/etcd
Restart=always

[install]
WantedBy=multi-user.target
```

# 创建etcd的CA证书
`vi etcd_ssl.cnf`
```shell
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name

[ req_distinguished_name ]

[ v3_req ]
# Extensions to add to a certificate request
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[alt_names]
IP.1 = 192.168.100.123
IP.2 = 192.168.100.124
IP.3 = 192.168.100.125
```
创建etcd服务端CA证书，包括key和crt文件，保存再/etc/etcd/pki目录下
```shell
openssl genrsa -out etcd_server.key 2048
openssl req -new -key etcd_server.key -config etcd_ssl.cnf -subj "/CN=etcd-server" -out etcd_server.csr
openssl x509 -req -in etcd_server.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -days 36500 -extensions v3_req -extfile etcd_ssl.cnf -out etcd_server.crt
```

再创建客户端CA证书，包括
```shell
openssl genrsa -out etcd_client.key 2048
openssl req -new -key etcd_client.key -config etcd_ssl.cnf -subj "/CN=etcd-client" -out etcd_client.csr
openssl x509 -req -in etcd_client.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -days 36500 -extensions v3_req -extfile etcd_ssl.cnf -out etcd_client.crt
```

# etcd参数配置说明
`vi /etc/etcd/etcd.conf`
官方示例配置文件
```shell
# This is the configuration file for the etcd server.
# Human-readable name for this member.
name: 'etcd1'
# Path to the data directory.
data-dir: "/etc/etcd/data"
# Path to the dedicated wal directory.
#wal-dir:
# Number of committed transactions to trigger a snapshot to disk.
#snapshot-count: 10000
# Time (in milliseconds) of a heartbeat interval.
#heartbeat-interval: 100
# Time (in milliseconds) for an election to timeout.
#election-timeout: 1000
# Raise alarms when backend size exceeds the given quota. 0 means use the
# default quota.
#quota-backend-bytes: 0
# List of comma separated URLs to listen on for peer traffic.
listen-peer-urls: "https://192.168.100.123:2380"
# List of comma separated URLs to listen on for client traffic.
listen-client-urls: "https://192.168.100.123:2379"
# Maximum number of snapshot files to retain (0 is unlimited).
#max-snapshots: 5
# Maximum number of wal files to retain (0 is unlimited).
#max-wals: 5
# Comma-separated white list of origins for CORS (cross-origin resource sharing).
#cors:
# List of this member's peer URLs to advertise to the rest of the cluster.
# The URLs needed to be a comma-separated list.
initial-advertise-peer-urls: "https://192.168.100.123:2380"
# List of this member's client URLs to advertise to the public.
# The URLs needed to be a comma-separated list.
advertise-client-urls: "https://192.168.100.123:2379"
# Discovery URL used to bootstrap the cluster.
#discovery:
# Valid values include 'exit', 'proxy'
#discovery-fallback: 'proxy'
# HTTP proxy to use for traffic to discovery service.
#discovery-proxy:
# DNS domain used to bootstrap initial cluster.
#discovery-srv:
# Initial cluster configuration for bootstrapping.
initial-cluster: "etcd1=https://192.168.100.123:2380,etcd2=https://192.168.100.124:2380,etcd3=https://192.168.100.125:2380"
# Initial cluster token for the etcd cluster during bootstrap.
initial-cluster-token: 'etcd-cluster'
# Initial cluster state ('new' or 'existing').
initial-cluster-state: 'new'
# Reject reconfiguration requests that would cause quorum loss.
#strict-reconfig-check: false
# Accept etcd V2 client requests
#enable-v2: true
# Enable runtime profiling data via HTTP server
#enable-pprof: true
# Valid values include 'on', 'readonly', 'off'
#proxy: 'off'
# Time (in milliseconds) an endpoint will be held in a failed state.
#proxy-failure-wait: 5000
# Time (in milliseconds) of the endpoints refresh interval.
#proxy-refresh-interval: 30000
# Time (in milliseconds) for a dial to timeout.
#proxy-dial-timeout: 1000
# Time (in milliseconds) for a write to timeout.
#proxy-write-timeout: 5000
# Time (in milliseconds) for a read to timeout.
#proxy-read-timeout: 0
client-transport-security:
  # Path to the client server TLS cert file.
  cert-file: "/etc/etcd/pki/etcd_server.crt"
  # Path to the client server TLS key file.
  key-file: "/etc/etcd/pki/etcd_server.key"
  # Enable client cert authentication.
  client-cert-auth: true
  # Path to the client server TLS trusted CA cert file.
  trusted-ca-file: "/etc/kubernetes/pki/ca.crt"
  # Client TLS using generated certificates
#  auto-tls: false
peer-transport-security:
  # Path to the peer server TLS cert file.
  cert-file: "/etc/etcd/pki/etcd_server.crt"
  # Path to the peer server TLS key file.
  key-file: "/etc/etcd/pki/etcd_server.key"
  # Enable peer client cert authentication.
  client-cert-auth: true
  # Path to the peer server TLS trusted CA cert file.
  trusted-ca-file: "/etc/kubernetes/pki/ca.crt"
  # Peer TLS using generated certificates.
#  auto-tls: false
# Enable debug-level logging for etcd.
debug: false
logger: zap
# Specify 'stdout' or 'stderr' to skip journald logging even when running under systemd.
log-outputs: [stderr]
# Force to create a new one member cluster.
#force-new-cluster: false
#auto-compaction-mode: periodic
#auto-compaction-retention: "1"
```

权威指南提供配置文件
```shell
# node1 conf实例文件
ETCD_NAME="etcd1"
ETCD_DATA_DIR="/etc/etcd/data"

ETCD_CERT_FILE="/etc/etcd/pki/etcd_server.crt"
ETCD_KEY_FILE="/etc/etcd/pki/etcd_server.key"
ETCD_TRUSTED_CA_FILE="/etc/kubernetes/pki/ca.crt"
ETCD_CLIENT_CERT_AUTH=true
ETCD_LISTEN_CLIENT_URLS="https://192.168.100.123:2379"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.100.123:2379"
ETCD_PEER_CERT_FILE="/etc/etcd/pki/etcd_server.crt"
ETCD_PEER_KEY_FILE="/etc/etcd/pki/etcd_server.key"
ETCD_PEER_TRUSTED_CA_FILE="/etc/kubernetes/pki/ca.crt"
ETCD_LISTEN_PEER_URLS="https://192.168.100.123:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.100.123:2380"

ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER="etcd1=https://192.168.100.123:2380,etcd2=https://192.168.100.124:2380,etcd3=https://192.168.100.125:2380"
ETCD_INITIAL_CLUSTER_STATE=new

```

# 验证etcd部署
```shell
# etcdctl endpoint health
127.0.0.1:2379 is healthy: successfully committed proposal: took = 1.
# etcdctl --cacert=/etc/kubernetes/pki/ca.crt --cert=/etc/etcd/pki/etcd_client.crt --key=/etc/etcd/pki/etcd_client.key --endpoints=https://192.168.100.123:2379,https://192.168.100.124:2379,https://192.168.100.125:2379 endpoint health
```