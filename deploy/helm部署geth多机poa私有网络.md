# helm部署geth多机poa私有网络

## 主机规划

| 主机名             | 用途                             | CPU  | 内存 | 磁盘（rssd） | label标签                                         | taint污点(为了不受其它服务干扰) |
| ------------------ | -------------------------------- | ---- | ---- | ------------ | ------------------------------------------------- | ------------------------------- |
| node7.pldtest.k8s  | geth 签名节点1                   | 4C   | 8G   | 100G         | geth/gluster=tps,geth/node=signer,geth/role=node  | geth/cluster=tps:NoSchedule     |
| node8.pldtest.k8s  | geth 浏览器                      | 8C   | 8G   | 200G         | geth/gluster=tps                                  | browser=blockscout:NoSchedule   |
| node9.pldtest.k8s  | geth 压测客户端                  | 8C   | 16G  | 100G         | geth/gluster=tps                                  | geth/cluster=client:NoSchedule  |
| node10.pldtest.k8s | geth 签名节点2                   | 4C   | 8G   | 100G         | geth/gluster=tps,geth/node=signer,geth/role=node  | geth/cluster=tps:NoSchedule     |
| node11.pldtest.k8s | geth 签名节点3                   | 4C   | 8G   | 100G         | geth/gluster=tps,geth/node=signer,geth/role=node  | geth/cluster=tps:NoSchedule     |
| node12.pldtest.k8s | geth 普通节点1                   | 4C   | 8G   | 100G         | geth/gluster=tps,geth/role=node                   | geth/cluster=tps:NoSchedule     |
| node13.pldtest.k8s | geth 普通节点2，浏览器访问此节点 | 16C  | 16G  | 100G         | geth/gluster=tps,geth/node=browser,geth/role=node | geth/cluster=tps:NoSchedule     |

### node添加label taint

略

### 准备influxdb

发布`influxdb` 1.x， 如果不需要收集metrics 数据可以不准备

略

## 下载geth helm chart 

登录k8s master1 

```sh
mkdir ~/geth-poa-ha-tps
cd ~/geth-poa-ha-tps
git clone https://github.com/paradeum-team/helm-charts
```

## 生成初始账号

```shell
for i in {1..4}
do
  # linux bash 生成随机密码${i}
  < /dev/urandom tr -dc _A-Z-a-z-0-9 | head -c${1:-12} > password${i}.txt
  # 创建账号${i}
  docker run -it --rm \
    -v /data/geth/data/chain:/root/.ethereum \
    -v /etc/localtime:/etc/localtime \
    -v `pwd`/password${i}.txt:/root/.ethereum/password.txt \
    ethereum/client-go:v1.10.13 account new --password /root/.ethereum/password.txt
  mv /data/geth/data/chain/keystore/UTC--* account${i}.txt
done
rm -rf /data/geth
```

## 创建 tps1 geth

### 创建 tps1-values.yaml

```yaml
key=`cat account1.txt`
account="`jq -r '.address' account1.txt`"
account4="`jq -r '.address' account4.txt`"
password=`cat password1.txt`
chainid=65533
cluster=tps
period=2
gasLimit=30000000

cat > tps1-values.yaml <<EOF
# Default values for geth.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: ethereum/client-go
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "v1.10.17"
  command: ["geth"]
  args:
    - --config
    - /root/config/config.custom.toml
    - --gcmode=archive
    - --unlock
    - ${account}
    - --password
    - /root/.ethereum/password
    - --mine
    - --nodiscover
    - --metrics
    - --miner.gaslimit=${gasLimit}
    - --metrics.influxdb
    - --metrics.influxdb.endpoint=http://gethtps-influxdb.geth.svc:8086
    - --metrics.influxdb.tags=host=tps1-singer-node
    - --txpool.locals=0x${account4}

initContainer:
  command: ["geth"]
  args: ["init", "/root/config/genesis.custom.json"]


imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

volumes:
  data:
    mountPath: /root/.ethereum
  config:
    mountPath: /root/config

persistence:
  enabled: true
  storageClass: "local-path"
  annotations: {}
  accessModes:
    - ReadWriteOnce
  size: 50Gi # For mainnet must increase

  ## A manually managed Persistent Volume and Claim
  ## If defined, PVC must be created manually before volume will be bound
  ## The value is evaluated as a template, so, for example, the name can depend on .Release or .Chart
  ##
  # existingClaim:
  ## An existing directory in the node
  ## If defined, host directory must be created manually before volume will be bound
  ## See https://kubernetes.io/docs/concepts/storage/volumes/#hostpath
  ##
  # hostPath:
  #   path: /root/.local/share/io.parity.ethereum
  #   type: Directory

serviceAccount:
  # Specifies whether a service account should be created
  create: true
  # Annotations to add to the service account
  annotations: {}
  # The name of the service account to use.
  # If not set and create is true, a name is generated using the fullname template
  name: ""

podAnnotations: {}

podSecurityContext:
  {}
  # fsGroup: 2000

securityContext:
  {}
  # capabilities:
  #   drop:
  #   - ALL
  # readOnlyRootFilesystem: true
  # runAsNonRoot: true
  # runAsUser: 1000

container:
  ports:
    http: 8545
    ws: 8546
    prometheus: 6060
    listener: 30303
    discovery: 30303

service:
  type: ClusterIP
  http: 8545
  ws: 8546

serviceP2P:
  type: hostPort
  listener: 30303
  discovery: 30303

prometheus: false

autoscaling:
  enabled: false

resources: {}
cluster: ${cluster}
nodeSelector:
  geth/role: node
tolerations:
- key: "geth/cluster"
  value: "${cluster}"
  operator: "Equal"
  effect: "NoSchedule"
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: "geth/cluster"
          operator: In
          values:
          - "${cluster}"
        - key: "geth/node"
          operator: In
          values:
          - "signer"
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: "geth/cluster"
            operator: In
            values:
            - "${cluster}"
        topologyKey: "kubernetes.io/hostname"

config: |
  [Eth]
  NetworkId = ${chainid}
  SyncMode = "full"
  
  [Metrics]
  HTTP = "0.0.0.0"

  [Node.P2P]
  StaticNodes = []
  TrustedNodes = []

genesis: '{"config":{"chainId":${chainid},"homesteadBlock":0,"eip150Block":0,"eip150Hash":"0x0000000000000000000000000000000000000000000000000000000000000000","eip155Block":0,"eip158Block":0,"byzantiumBlock":0,"constantinopleBlock":0,"petersburgBlock":0,"istanbulBlock":0,"clique":{"period":${period},"epoch":30000}},"extradata":"0x0000000000000000000000000000000000000000000000000000000000000000${account}0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000","gasLimit":"${gasLimit}","coinbase":"0x0000000000000000000000000000000000000000","difficulty":"0x1","mixHash":"0x0000000000000000000000000000000000000000000000000000000000000000","alloc":{"0000000000000000000000000000000000000000":{"balance":"0x1"},"${account}":{"balance":"0x200000000000000000000000000000000000000000000000000000000000000"}},"nonce":"0x0000000000000123","number":"0x0","gasUsed":"0x0","parentHash":"0x0000000000000000000000000000000000000000000000000000000000000000","baseFeePerGas":null}'
key: '${key}'
password: "${password}"

#Additional Environment Variables
env: {}
EOF
```

### 执行安装tps1 geth

```shell
helm upgrade --install tps1-geth helm-charts/geth -f tps1-values.yaml -n geth --create-namespace
```

### 查看日志

```shell
kubectl -n geth logs tps1-geth-0
```

显示 🔗`block reached canonical chain`  等字样为正常

```
INFO [04-07|10:51:43.000] Successfully sealed new block            number=5 sealhash=7ab007..83faac hash=b5b4fe..c92613 elapsed=1.999s
INFO [04-07|10:51:43.000] 🔨 mined potential block                  number=5 hash=b5b4fe..c92613
INFO [04-07|10:51:43.001] Commit new sealing work                  number=6 sealhash=3a6b70..34c9b5 uncles=0 txs=0 gas=0 fees=0 elapsed="308.399µs"
INFO [04-07|10:51:43.001] Commit new sealing work                  number=6 sealhash=3a6b70..34c9b5 uncles=0 txs=0 gas=0 fees=0 elapsed="514.275µs"
```

### 查看 enode 信息

```shell
kubectl exec tps1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.nodeInfo.enode
```

## 创建tps2 geth

### 创建tps2-values.yaml

```yaml
# 获取tps1 geth enode
tps1_geth_hostip=`kubectl get pod tps1-geth-0 -n geth -o jsonpath="{.status.hostIP}"`
enode1=`kubectl exec tps1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.nodeInfo.enode 2>&1|grep enode|sed "s/127.0.0.1/${tps1_geth_hostip}/g"`
key=`cat account2.txt`
# 这里account1 是固定的，所有节点初始化信息中的预置账号都是account1
account1="`jq -r '.address' account1.txt`"
account="`jq -r '.address' account2.txt`"
account4="`jq -r '.address' account4.txt`"
password=`cat password2.txt`
chainid=65533
cluster=tps
period=2
gasLimit=30000000

cat > tps2-values.yaml <<EOF
# Default values for geth.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: ethereum/client-go
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "v1.10.17"
  command: ["geth"]
  args:
    - --config
    - /root/config/config.custom.toml
    - --gcmode=archive
    - --unlock
    - ${account}
    - --password
    - /root/.ethereum/password
    - --mine
    - --nodiscover
    - --metrics
    - --miner.gaslimit=${gasLimit}
    - --metrics.influxdb
    - --metrics.influxdb.endpoint=http://gethtps-influxdb.geth.svc:8086
    - --metrics.influxdb.tags=host=tps2-singer-node
    - --txpool.locals=0x${account4}

initContainer:
  command: ["geth"]
  args: ["init", "/root/config/genesis.custom.json"]


imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

volumes:
  data:
    mountPath: /root/.ethereum
  config:
    mountPath: /root/config

persistence:
  enabled: true
  storageClass: "local-path"
  annotations: {}
  accessModes:
    - ReadWriteOnce
  size: 50Gi # For mainnet must increase

  ## A manually managed Persistent Volume and Claim
  ## If defined, PVC must be created manually before volume will be bound
  ## The value is evaluated as a template, so, for example, the name can depend on .Release or .Chart
  ##
  # existingClaim:
  ## An existing directory in the node
  ## If defined, host directory must be created manually before volume will be bound
  ## See https://kubernetes.io/docs/concepts/storage/volumes/#hostpath
  ##
  # hostPath:
  #   path: /root/.local/share/io.parity.ethereum
  #   type: Directory

serviceAccount:
  # Specifies whether a service account should be created
  create: true
  # Annotations to add to the service account
  annotations: {}
  # The name of the service account to use.
  # If not set and create is true, a name is generated using the fullname template
  name: ""

podAnnotations: {}

podSecurityContext:
  {}
  # fsGroup: 2000

securityContext:
  {}
  # capabilities:
  #   drop:
  #   - ALL
  # readOnlyRootFilesystem: true
  # runAsNonRoot: true
  # runAsUser: 1000

container:
  ports:
    http: 8545
    ws: 8546
    prometheus: 6060
    listener: 30303
    discovery: 30303

service:
  type: ClusterIP
  http: 8545
  ws: 8546
  prometheus: 6060

serviceP2P:
  type: hostPort
  listener: 30303
  discovery: 30303

prometheus: false

autoscaling:
  enabled: false

resources: {}
cluster: ${cluster}
nodeSelector:
  geth/role: node
tolerations:
- key: "geth/cluster"
  value: "${cluster}"
  operator: "Equal"
  effect: "NoSchedule"
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: "geth/cluster"
          operator: In
          values:
          - "${cluster}"
        - key: "geth/node"
          operator: In
          values:
          - "signer"
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: "geth/cluster"
            operator: In
            values:
            - "${cluster}"
        topologyKey: "kubernetes.io/hostname"
        
config: |
  [Eth]
  NetworkId = ${chainid}
  SyncMode = "full"

  [Metrics]
  HTTP = "0.0.0.0"

  [Node.P2P]
  StaticNodes = [${enode1}]
  TrustedNodes = [${enode1}]

genesis: '{"config":{"chainId":${chainid},"homesteadBlock":0,"eip150Block":0,"eip150Hash":"0x0000000000000000000000000000000000000000000000000000000000000000","eip155Block":0,"eip158Block":0,"byzantiumBlock":0,"constantinopleBlock":0,"petersburgBlock":0,"istanbulBlock":0,"clique":{"period":${period},"epoch":30000}},"extradata":"0x0000000000000000000000000000000000000000000000000000000000000000${account1}0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000","gasLimit":"${gasLimit}","coinbase":"0x0000000000000000000000000000000000000000","difficulty":"0x1","mixHash":"0x0000000000000000000000000000000000000000000000000000000000000000","alloc":{"0000000000000000000000000000000000000000":{"balance":"0x1"},"${account1}":{"balance":"0x200000000000000000000000000000000000000000000000000000000000000"}},"nonce":"0x0000000000000123","number":"0x0","gasUsed":"0x0","parentHash":"0x0000000000000000000000000000000000000000000000000000000000000000","baseFeePerGas":null}'
key: '${key}'
password: "${password}"

#Additional Environment Variables
env: {}
EOF
```

### 安装tps2 geth

```shell
helm upgrade --install tps2-geth helm-charts/geth -f tps2-values.yaml -n geth --create-namespace
```

### 查看日志

```shell
kubectl -n geth logs tps2-geth-0
```

显示 `unauthorized signer` ` Imported new chain segment` 等字样为正常

```
INFO [04-07|10:53:03.993] Etherbase automatically configured       address=0xEE81F37cd0DA914eBd56292d9e8a374d23aAD263
INFO [04-07|10:53:03.994] Commit new sealing work                  number=1 sealhash=aa198b..db77a9 uncles=0 txs=0 gas=0 fees=0 elapsed="126.992µs"
WARN [04-07|10:53:03.994] Block sealing failed                     err="unauthorized signer"
INFO [04-07|10:53:03.994] Commit new sealing work                  number=1 sealhash=aa198b..db77a9 uncles=0 txs=0 gas=0 fees=0 elapsed="272.825µs"
INFO [04-07|10:53:13.259] Block synchronisation started
INFO [04-07|10:53:13.260] Mining aborted due to sync
INFO [04-07|10:53:13.264] Downloader queue stats                   receiptTasks=0 blockTasks=0 itemSize=645.70B throttle=8192
INFO [04-07|10:53:13.270] Imported new chain segment               blocks=48 txs=0 mgas=0.000 elapsed=5.886ms     mgasps=0.000 number=48 hash=e34495..4b271e dirty=0.00B
INFO [04-07|10:53:16.266] Imported new chain segment               blocks=3  txs=0 mgas=0.000 elapsed="710.523µs" mgasps=0.000 number=51 hash=05e2bf..4a2b7c dirty=0.00B
INFO [04-07|10:53:16.266] Commit new sealing work                  number=52 sealhash=9a99a4..6b05b7 uncles=0 txs=0 gas=0 fees=0 elapsed="31.011µs"
WARN [04-07|10:53:16.266] Block sealing failed                     err="unauthorized signer"
INFO [04-07|10:53:16.267] Commit new sealing work                  number=52 sealhash=9a99a4..6b05b7 uncles=0 txs=0 gas=0 fees=0 elapsed="110.464µs"
```

### 查看tps1 geth peers

```shell
kubectl exec tps1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.peers
```

有一项连接记录为正常

```shell
Defaulted container "geth" out of: geth, geth-init (init)
[{
    caps: ["eth/66", "snap/1"],
    enode: "enode://d0a462174092aee2d189b5f161d8f9c89f61594b0772f020b4349a3c8662cbb21ef17398f6435110704ec1e30a6eb69deee2bfbb1ed3fcce4c17138ed3af4e74@10.128.8.1:20806",
    id: "86c5839a64b9b94a021ca3f29a682eed9147b930d769d8cfeefb348ebf0fac1c",
    name: "Geth/v1.10.17-stable-25c9b49f/linux-amd64/go1.18",
    network: {
      inbound: true,
      localAddress: "10.128.8.93:30303",
      remoteAddress: "10.128.8.1:20806",
      static: false,
      trusted: false
    },
    protocols: {
      eth: {
        difficulty: 1,
        head: "0x86d61543de9d808da44f1b1b8b7ccb1a4e8f156aca7ad177e12f3dc678808e2a",
        version: 66
      },
      snap: {
        version: 1
      }
    }
}]
```

## 创建tps3 geth

### 创建tps3-values.yaml

```yaml
# 获取tps geth enode
tps1_geth_hostip=`kubectl get pod tps1-geth-0 -n geth -o jsonpath="{.status.hostIP}"`
enode1=`kubectl exec  tps1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.nodeInfo.enode 2>&1|grep enode|sed "s/127.0.0.1/${tps1_geth_hostip}/g"`
tps2_geth_hostip=`kubectl get pod tps2-geth-0 -n geth -o jsonpath="{.status.hostIP}"`
enode2=`kubectl exec  tps2-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.nodeInfo.enode 2>&1|grep enode|sed "s/127.0.0.1/${tps2_geth_hostip}/g"`
key=`cat account3.txt`
# 这里account1 是固定的，所有节点初始化信息中的预置账号都是account1
account1="`jq -r '.address' account1.txt`"
account="`jq -r '.address' account3.txt`"
account4="`jq -r '.address' account4.txt`"
password=`cat password3.txt`
chainid=65533
cluster=tps
period=2
gasLimit=30000000

cat > tps3-values.yaml <<EOF
# Default values for geth.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: ethereum/client-go
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "v1.10.17"
  command: ["geth"]
  args:
    - --config
    - /root/config/config.custom.toml
    - --gcmode=archive
    - --unlock
    - ${account}
    - --password
    - /root/.ethereum/password
    - --mine
    - --nodiscover
    - --metrics
    - --miner.gaslimit=${gasLimit}
    - --metrics.influxdb
    - --metrics.influxdb.endpoint=http://gethtps-influxdb.geth.svc:8086
    - --metrics.influxdb.tags=host=tps3-singer-node
    - --txpool.locals=0x${account4}

initContainer:
  command: ["geth"]
  args: ["init", "/root/config/genesis.custom.json"]


imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

volumes:
  data:
    mountPath: /root/.ethereum
  config:
    mountPath: /root/config

persistence:
  enabled: true
  storageClass: "local-path"
  annotations: {}
  accessModes:
    - ReadWriteOnce
  size: 50Gi # For mainnet must increase

  ## A manually managed Persistent Volume and Claim
  ## If defined, PVC must be created manually before volume will be bound
  ## The value is evaluated as a template, so, for example, the name can depend on .Release or .Chart
  ##
  # existingClaim:
  ## An existing directory in the node
  ## If defined, host directory must be created manually before volume will be bound
  ## See https://kubernetes.io/docs/concepts/storage/volumes/#hostpath
  ##
  # hostPath:
  #   path: /root/.local/share/io.parity.ethereum
  #   type: Directory

serviceAccount:
  # Specifies whether a service account should be created
  create: true
  # Annotations to add to the service account
  annotations: {}
  # The name of the service account to use.
  # If not set and create is true, a name is generated using the fullname template
  name: ""

podAnnotations: {}

podSecurityContext:
  {}
  # fsGroup: 2000

securityContext:
  {}
  # capabilities:
  #   drop:
  #   - ALL
  # readOnlyRootFilesystem: true
  # runAsNonRoot: true
  # runAsUser: 1000

container:
  ports:
    http: 8545
    ws: 8546
    prometheus: 6060
    listener: 30303
    discovery: 30303

service:
  type: ClusterIP
  http: 8545
  ws: 8546
  prometheus: 6060

serviceP2P:
  type: hostPort
  listener: 30303
  discovery: 30303

prometheus: false

autoscaling:
  enabled: false

resources: {}
cluster: ${cluster}
nodeSelector:
  geth/role: node
tolerations:
- key: "geth/cluster"
  value: "${cluster}"
  operator: "Equal"
  effect: "NoSchedule"
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: "geth/cluster"
          operator: In
          values:
          - "${cluster}"
        - key: "geth/node"
          operator: In
          values:
          - "signer"
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: "geth/cluster"
            operator: In
            values:
            - "${cluster}"
        topologyKey: "kubernetes.io/hostname"

config: |
  [Eth]
  NetworkId = ${chainid}
  SyncMode = "full"
  
  [Metrics]
  HTTP = "0.0.0.0"

  [Node.P2P]
  StaticNodes = [${enode1},${enode2}]
  TrustedNodes = [${enode1},${enode2}]

genesis: '{"config":{"chainId":${chainid},"homesteadBlock":0,"eip150Block":0,"eip150Hash":"0x0000000000000000000000000000000000000000000000000000000000000000","eip155Block":0,"eip158Block":0,"byzantiumBlock":0,"constantinopleBlock":0,"petersburgBlock":0,"istanbulBlock":0,"clique":{"period":${period},"epoch":30000}},"extradata":"0x0000000000000000000000000000000000000000000000000000000000000000${account1}0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000","gasLimit":"${gasLimit}","coinbase":"0x0000000000000000000000000000000000000000","difficulty":"0x1","mixHash":"0x0000000000000000000000000000000000000000000000000000000000000000","alloc":{"0000000000000000000000000000000000000000":{"balance":"0x1"},"${account1}":{"balance":"0x200000000000000000000000000000000000000000000000000000000000000"}},"nonce":"0x0000000000000123","number":"0x0","gasUsed":"0x0","parentHash":"0x0000000000000000000000000000000000000000000000000000000000000000","baseFeePerGas":null}'
key: '${key}'
password: "${password}"

#Additional Environment Variables
env: {}
EOF
```

### 安装tps3 geth

```shell
helm upgrade --install tps3-geth helm-charts/geth -f tps3-values.yaml -n geth
```

### 查看日志

```shell
kubectl -n geth logs tps3-geth-0
```

显示 `unauthorized signer` ` Imported new chain segment` 等字样为正常

```
INFO [04-07|11:02:46.521] Etherbase automatically configured       address=0xCd995671CA414996bd6Cd3C8ea989Da003B9DAF3
INFO [04-07|11:02:46.521] Commit new sealing work                  number=1 sealhash=7d5c3b..3ab7dc uncles=0 txs=0 gas=0 fees=0 elapsed="170.928µs"
WARN [04-07|11:02:46.521] Block sealing failed                     err="unauthorized signer"
INFO [04-07|11:02:46.521] Commit new sealing work                  number=1 sealhash=7d5c3b..3ab7dc uncles=0 txs=0 gas=0 fees=0 elapsed="328.644µs"
INFO [04-07|11:02:55.679] Block synchronisation started
INFO [04-07|11:02:55.680] Mining aborted due to sync
INFO [04-07|11:02:55.688] Downloader queue stats                   receiptTasks=0 blockTasks=0 itemSize=650.00B throttle=8192
INFO [04-07|11:02:55.716] Imported new chain segment               blocks=192 txs=0 mgas=0.000 elapsed=27.442ms    mgasps=0.000 number=192 hash=6f3766..115058 age=4m58s   dirty=0.00B
INFO [04-07|11:02:55.743] Imported new chain segment               blocks=147 txs=0 mgas=0.000 elapsed=25.935ms    mgasps=0.000 number=339 hash=0883b1..2eef22 dirty=0.00B
INFO [04-07|11:02:58.692] Imported new chain segment               blocks=3   txs=0 mgas=0.000 elapsed=1.093ms     mgasps=0.000 number=342 hash=4bc518..3a97f5 dirty=0.00B
INFO [04-07|11:02:58.692] Commit new sealing work                  number=343 sealhash=214ae8..ebe387 uncles=0 txs=0 gas=0 fees=0 elapsed="57.549µs"
WARN [04-07|11:02:58.693] Block sealing failed                     err="unauthorized signer"
INFO [04-07|11:02:58.693] Commit new sealing work                  number=343 sealhash=214ae8..ebe387 uncles=0 txs=0 gas=0 fees=0 elapsed="150.717µs"
INFO [04-07|11:02:59.002] Imported new chain segment               blocks=1   txs=0 mgas=0.000 elapsed="293.573µs" mgasps=0.000 number=343 hash=f13215..0e6daf dirty=0.00B
INFO [04-07|11:02:59.002] Commit new sealing work                  number=344 sealhash=a70275..bcb6f2 uncles=0 txs=0 gas=0 fees=0 elapsed="230.022µs"
WARN [04-07|11:02:59.002] Block sealing failed                     err="unauthorized signer"
INFO [04-07|11:02:59.003] Commit new sealing work                  number=344 sealhash=a70275..bcb6f2 uncles=0 txs=0 gas=0 fees=0 elapsed="377.474µs"
```

### 查看tps1 geth peers

```
kubectl exec tps1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.peers
```

有两项连接记录为正常

```shell
Defaulted container "geth" out of: geth, geth-init (init)
[{
    caps: ["eth/66", "snap/1"],
    enode: "enode://d0a462174092aee2d189b5f161d8f9c89f61594b0772f020b4349a3c8662cbb21ef17398f6435110704ec1e30a6eb69deee2bfbb1ed3fcce4c17138ed3af4e74@10.128.8.1:20806",
    id: "86c5839a64b9b94a021ca3f29a682eed9147b930d769d8cfeefb348ebf0fac1c",
    name: "Geth/v1.10.17-stable-25c9b49f/linux-amd64/go1.18",
    network: {
      inbound: true,
      localAddress: "10.128.8.93:30303",
      remoteAddress: "10.128.8.1:20806",
      static: false,
      trusted: false
    },
    protocols: {
      eth: {
        difficulty: 1,
        head: "0x86d61543de9d808da44f1b1b8b7ccb1a4e8f156aca7ad177e12f3dc678808e2a",
        version: 66
      },
      snap: {
        version: 1
      }
    }
}, {
    caps: ["eth/66", "snap/1"],
    enode: "enode://98250d502ffde7d799084e15c646e45b3152eb2274f642a4be17ccdda64e9c59cf8d885383f70f28a8db0acdbcd22e6b2c0aaf08108a3939d48d6838065ade42@10.128.8.1:61848",
    id: "95de6602c73dec191b78f9d3c9cd2110ad118033003f058a7d8c3514b9a6f5bb",
    name: "Geth/v1.10.17-stable-25c9b49f/linux-amd64/go1.18",
    network: {
      inbound: true,
      localAddress: "10.128.8.93:30303",
      remoteAddress: "10.128.8.1:61848",
      static: false,
      trusted: false
    },
    protocols: {
      eth: {
        difficulty: 1,
        head: "0x86d61543de9d808da44f1b1b8b7ccb1a4e8f156aca7ad177e12f3dc678808e2a",
        version: 66
      },
      snap: {
        version: 1
      }
    }
}]
```

### 修改三个geth 初始连接enode

#### 修改tps*-values.yaml

```shell
# 获取tps geth enode
tps1_geth_hostip=`kubectl get pod tps1-geth-0 -n geth -o jsonpath="{.status.hostIP}"`
enode1=`kubectl exec tps1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.nodeInfo.enode 2>&1|grep enode|sed "s/127.0.0.1/${tps1_geth_hostip}/g"`
tps2_geth_hostip=`kubectl get pod tps2-geth-0 -n geth -o jsonpath="{.status.hostIP}"`
enode2=`kubectl exec tps2-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.nodeInfo.enode 2>&1|grep enode|sed "s/127.0.0.1/${tps2_geth_hostip}/g"`
tps3_geth_hostip=`kubectl get pod tps3-geth-0 -n geth -o jsonpath="{.status.hostIP}"`
enode3=`kubectl exec tps3-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.nodeInfo.enode 2>&1|grep enode|sed "s/127.0.0.1/${tps3_geth_hostip}/g"`

if [ -z "$enode1" ] || [ -z "$enode2" ] || [ -z "$enode3" ];then
  echo "enodes empty"
	echo "enode1: $enode1"
	echo "enode2: $enode2"
	echo "enode3: $enode3"
else
  sed -i "s#StaticNodes.*#StaticNodes = [${enode1},${enode2},${enode3}]#g" tps{1,2,3}-values.yaml
  sed -i "s#TrustedNodes.*#TrustedNodes = [${enode1},${enode2},${enode3}]#g" tps{1,2,3}-values.yaml
fi
```

#### 更新tps*-geth

```shell
helm upgrade --install tps1-geth helm-charts/geth -f tps1-values.yaml -n geth
helm upgrade --install tps2-geth helm-charts/geth -f tps2-values.yaml -n geth
helm upgrade --install tps3-geth helm-charts/geth -f tps3-values.yaml -n geth
```

#### 查看peers

```shell
kubectl exec  tps1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.peers
kubectl exec  tps2-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.peers
kubectl exec  tps3-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.peers
```

从三个pod 跟查peers 都为两个连接为正常

## 设置签名账号

获取三个账号地址

```shell
acc1="0x`jq -r '.address' account1.txt`"
acc2="0x`jq -r '.address' account2.txt`"
acc3="0x`jq -r '.address' account3.txt`"
```

提名tps2-geth 初始账号为签名账号

```shell
# 在 tps1 geth 提名acc2 账号
kubectl exec  tps1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  "clique.propose('$acc2', true)"

# 输出
Defaulted container "geth" out of: geth, geth-init (init)
null

# 查看签名账号列表
kubectl exec  tps1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  'clique.getSnapshot().signers'

# 输出2个账号，分别为tps1-geth 和 tps2-geth 的初始账号
Defaulted container "geth" out of: geth, geth-init (init)
{
  0x0fbf567d26e9de3c30852d5158a5d1f8b980e115: {},
  0xee81f37cd0da914ebd56292d9e8a374d23aad263: {}
}
```

提名tps3-geth 初始账号为签名账号(第三个账号提名需要一半以上签名者提名)

```shell
kubectl exec  tps1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  "clique.propose('$acc3', true)"
kubectl exec  tps2-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  "clique.propose('$acc3', true)"

kubectl exec  tps1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  'clique.getSnapshot().signers'

# 输出三个签名者为正常
Defaulted container "geth" out of: geth, geth-init (init)
{
  0x2caf0dd917cd0bf69fef580faf718356b4e7a930: {},
  0x87962f20925156b4ccf20408d712d34d9f1c260b: {},
  0xb6e44ba48d742ae915567874ee91a4aeacded421: {}
}
```

查看各tps geth 日志，显示 出块信息为正常

```shell
# 查看各tps3 geth 日志
kubectl -n geth logs tps3-geth-0 --tail 100

# 输出
INFO [04-07|11:20:26.001] Successfully sealed new block            number=676 sealhash=5599da..61296c hash=e58bc2..01b7c7 elapsed=1.999s
INFO [04-07|11:20:26.001] 🔨 mined potential block                  number=676 hash=e58bc2..01b7c7
INFO [04-07|11:20:26.002] Commit new sealing work                  number=677 sealhash=ea90fd..5337f7 uncles=0 txs=0 gas=0 fees=0 elapsed="263.345µs"
WARN [04-07|11:20:26.002] Block sealing failed                     err="signed recently, must wait for others"
INFO [04-07|11:20:26.002] Commit new sealing work                  number=677 sealhash=ea90fd..5337f7 uncles=0 txs=0 gas=0 fees=0 elapsed="436.957µs"
INFO [04-07|11:20:28.002] Imported new chain segment               blocks=1 txs=0 mgas=0.000 elapsed="278.475µs" mgasps=0.000 number=677 hash=8e2470..8d267d dirty=0.00B
INFO [04-07|11:20:28.002] 🔗 block reached canonical chain          number=670 hash=bd2495..6157bf
INFO [04-07|11:20:28.002] Commit new sealing work                  number=678 sealhash=464232..b78710 uncles=0 txs=0 gas=0 fees=0 elapsed="162.328µs"
INFO [04-07|11:20:28.003] Commit new sealing work                  number=678 sealhash=464232..b78710 uncles=0 txs=0 gas=0 fees=0 elapsed="344.746µs"
INFO [04-07|11:20:30.002] Imported new chain segment               blocks=1 txs=0 mgas=0.000 elapsed="352.89µs"  mgasps=0.000 number=678 hash=a55ada..0e8cc0 dirty=0.00B
INFO [04-07|11:20:30.003] Commit new sealing work                  number=679 sealhash=e4a656..83a7a3 uncles=0 txs=0 gas=0 fees=0 elapsed="129.275µs"
INFO [04-07|11:20:30.003] Commit new sealing work                  number=679 sealhash=e4a656..83a7a3 uncles=0 txs=0 gas=0 fees=0 elapsed="324.732µs"
```

## 测试交易

```shell
acc1="0x`jq -r '.address' account1.txt`"
acc4="0x`jq -r '.address' account4.txt`"
kubectl exec -it tps1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec "amount = web3.toWei(100000000000); eth.sendTransaction({from: '$acc1', to: '$acc4', value: amount});"

# 输出
"0xa3e6a034ed918cf07a85cfb795986e15e8124eeef5e8ef443f07986b883d98d6"

kubectl exec -it tps1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec "eth.getBalance('$acc4')"

kubectl exec -it tps2-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec "eth.getBalance('$acc4')"

kubectl exec -it tps3-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec "eth.getBalance('$acc4')"

# 结果都输出为正常
Defaulted container "geth" out of: geth, geth-init (init)
10000000000000000
```

## 添加浏览器访问普通node

### 创建 tps-browser-node-values.yaml

```
# 获取tps geth enode
tps1_geth_hostip=`kubectl get pod tps1-geth-0 -n geth -o jsonpath="{.status.hostIP}"`
enode1=`kubectl exec tps1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.nodeInfo.enode 2>&1|grep enode|sed "s/127.0.0.1/${tps1_geth_hostip}/g"`
tps2_geth_hostip=`kubectl get pod tps2-geth-0 -n geth -o jsonpath="{.status.hostIP}"`
enode2=`kubectl exec tps2-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.nodeInfo.enode 2>&1|grep enode|sed "s/127.0.0.1/${tps2_geth_hostip}/g"`
tps3_geth_hostip=`kubectl get pod tps3-geth-0 -n geth -o jsonpath="{.status.hostIP}"`
enode3=`kubectl exec tps3-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.nodeInfo.enode 2>&1|grep enode|sed "s/127.0.0.1/${tps3_geth_hostip}/g"`

# 这里account1 是固定的，所有节点初始化信息中的预置账号都是account1
account1="`jq -r '.address' account1.txt`"
chainid=65533
cluster=tps
period=2
gasLimit=30000000

cat > tps-browser-node-values.yaml <<EOF
# Default values for geth.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: ethereum/client-go
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "v1.10.17"
  command: ["geth"]
  args:
    - --config
    - /root/config/config.custom.toml
    - --http
    - --ws
    - --ws.addr=0.0.0.0
    - --ws.origins=*
    - --gcmode=archive
    - --nodiscover
    - --metrics
    - --metrics.addr=0.0.0.0
    - --http.vhosts=*
    - --http.corsdomain=*
    - --metrics.influxdb
    - --metrics.influxdb.endpoint=http://gethtps-influxdb.geth.svc:8086
    - --metrics.influxdb.tags=host=tps-browser-node-blockscout

initContainer:
  command: ["geth"]
  args: ["init", "/root/config/genesis.custom.json"]


imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

volumes:
  data:
    mountPath: /root/.ethereum
  config:
    mountPath: /root/config

persistence:
  enabled: true
  storageClass: "local-path"
  annotations: {}
  accessModes:
    - ReadWriteOnce
  size: 50Gi # For mainnet must increase

  ## A manually managed Persistent Volume and Claim
  ## If defined, PVC must be created manually before volume will be bound
  ## The value is evaluated as a template, so, for example, the name can depend on .Release or .Chart
  ##
  # existingClaim:
  ## An existing directory in the node
  ## If defined, host directory must be created manually before volume will be bound
  ## See https://kubernetes.io/docs/concepts/storage/volumes/#hostpath
  ##
  # hostPath:
  #   path: /root/.local/share/io.parity.ethereum
  #   type: Directory

serviceAccount:
  # Specifies whether a service account should be created
  create: true
  # Annotations to add to the service account
  annotations: {}
  # The name of the service account to use.
  # If not set and create is true, a name is generated using the fullname template
  name: ""

podAnnotations: {}

podSecurityContext:
  {}
  # fsGroup: 2000

securityContext:
  {}
  # capabilities:
  #   drop:
  #   - ALL
  # readOnlyRootFilesystem: true
  # runAsNonRoot: true
  # runAsUser: 1000

container:
  ports:
    http: 8545
    ws: 8546
    prometheus: 6060
    listener: 30303
    discovery: 30303

service:
  type: ClusterIP
  http: 8545
  ws: 8546
  prometheus: 6060
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600

serviceP2P:
  type: hostPort
  listener: 30303
  discovery: 30303

prometheus: false

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80
  # targetMemoryUtilizationPercentage: 80

resources: {}
cluster: ${cluster}
nodeSelector:
  geth/role: node
tolerations:
- key: "geth/cluster"
  value: "${cluster}"
  operator: "Equal"
  effect: "NoSchedule"
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: "geth/cluster"
          operator: In
          values:
          - "${cluster}"
        - key: "geth/node"
          operator: In
          values:
          - "browser"
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: "geth/cluster"
            operator: In
            values:
            - "${cluster}"
        topologyKey: "kubernetes.io/hostname"

config: |
  [Eth]
  NetworkId = ${chainid}
  SyncMode = "full"

  [Node]
  HTTPHost = "0.0.0.0"
  HTTPModules = ["eth", "net", "web3", "txpool"]

  [Metrics]
  HTTP = "0.0.0.0"

  [Node.P2P]
  StaticNodes = [${enode1},${enode2},${enode3}]
  TrustedNodes = [${enode1},${enode2},${enode3}]

genesis: '{"config":{"chainId":${chainid},"homesteadBlock":0,"eip150Block":0,"eip150Hash":"0x0000000000000000000000000000000000000000000000000000000000000000","eip155Block":0,"eip158Block":0,"byzantiumBlock":0,"constantinopleBlock":0,"petersburgBlock":0,"istanbulBlock":0,"clique":{"period":${period},"epoch":30000}},"extradata":"0x0000000000000000000000000000000000000000000000000000000000000000${account1}0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000","gasLimit":"${gasLimit}","coinbase":"0x0000000000000000000000000000000000000000","difficulty":"0x1","mixHash":"0x0000000000000000000000000000000000000000000000000000000000000000","alloc":{"0000000000000000000000000000000000000000":{"balance":"0x1"},"${account1}":{"balance":"0x200000000000000000000000000000000000000000000000000000000000000"}},"nonce":"0x0000000000000123","number":"0x0","gasUsed":"0x0","parentHash":"0x0000000000000000000000000000000000000000000000000000000000000000","baseFeePerGas":null}'

#Additional Environment Variables
env: {}
EOF
```

### 安装浏览器访问node

```sh
helm upgrade --install tps-browser-node-geth helm-charts/geth -f tps-browser-node-values.yaml -n geth
```

查看日志

```
kubectl logs tps-browser-node-geth-0 -n geth --tail 100
```

输出`Imported new chain segment` 等为正常

```
INFO [04-07|11:46:54.879] Block synchronisation started
INFO [04-07|11:46:54.889] Downloader queue stats                   receiptTasks=0 blockTasks=0 itemSize=650.00B throttle=8192
INFO [04-07|11:46:54.955] Imported new chain segment               blocks=576 txs=0 mgas=0.000 elapsed=64.371ms    mgasps=0.000 number=576 hash=582353..8a78e9 age=29m51s  dirty=0.00B
INFO [04-07|11:46:55.058] Imported new chain segment               blocks=892 txs=1 mgas=0.021 elapsed=98.605ms    mgasps=0.213 number=1468 hash=ed0242..4d1a57 dirty=0.00B
INFO [04-07|11:46:57.899] Imported new chain segment               blocks=3   txs=0 mgas=0.000 elapsed="805.926µs" mgasps=0.000 number=1471 hash=3a2aa7..999ae0 dirty=0.00B
INFO [04-07|11:46:58.005] Imported new chain segment               blocks=1   txs=0 mgas=0.000 elapsed="269.761µs" mgasps=0.000 number=1472 hash=9972d0..5350a8 dirty=0.00B
INFO [04-07|11:47:00.003] Imported new chain segment               blocks=1   txs=0 mgas=0.000 elapsed="267.067µs" mgasps=0.000 number=1473 hash=28dd11..6432e1 dirty=0.00B
INFO [04-07|11:47:02.005] Imported new chain segment               blocks=1   txs=0 mgas=0.000 elapsed="239.511µs" mgasps=0.000 number=1474 hash=6db138..7e9bf9 dirty=0.00B
INFO [04-07|11:47:04.005] Imported new chain segment               blocks=1   txs=0 mgas=0.000 elapsed="279.908µs" mgasps=0.000 number=1475 hash=cc2187..7f58ba dirty=0.00B
INFO [04-07|11:47:06.001] Imported new chain segment               blocks=1   txs=0 mgas=0.000 elapsed="264.014µs" mgasps=0.000 number=1476 hash=252a7a..0d3c95 dirty=0.00B
INFO [04-07|11:47:08.004] Imported new chain segment               blocks=1   txs=0 mgas=0.000 elapsed="240.253µs" mgasps=0.000 number=1477 hash=29171f..bee6ee dirty=0.00B
```

## 添加普通node

### 创建 tps-node-values.yaml

```yaml
# 获取tps geth enode
tps1_geth_hostip=`kubectl get pod tps1-geth-0 -n geth -o jsonpath="{.status.hostIP}"`
enode1=`kubectl exec tps1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.nodeInfo.enode 2>&1|grep enode|sed "s/127.0.0.1/${tps1_geth_hostip}/g"`
tps2_geth_hostip=`kubectl get pod tps2-geth-0 -n geth -o jsonpath="{.status.hostIP}"`
enode2=`kubectl exec tps2-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.nodeInfo.enode 2>&1|grep enode|sed "s/127.0.0.1/${tps2_geth_hostip}/g"`
tps3_geth_hostip=`kubectl get pod tps3-geth-0 -n geth -o jsonpath="{.status.hostIP}"`
enode3=`kubectl exec tps3-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.nodeInfo.enode 2>&1|grep enode|sed "s/127.0.0.1/${tps3_geth_hostip}/g"`

key=`cat account4.txt`
password=`cat password4.txt`
# 这里account1 是固定的，所有节点初始化信息中的预置账号都是account1
account1="`jq -r '.address' account1.txt`"
chainid=65533
cluster=tps
period=2
gasLimit=30000000

cat > tps-node-values.yaml <<EOF
# Default values for geth.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: ethereum/client-go
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "v1.10.17"
  command: ["geth"]
  args:
    - --config
    - /root/config/config.custom.toml
    - --http
    - --ws
    - --ws.addr=0.0.0.0
    - --ws.origins=*
    - --allow-insecure-unlock # 压测使用，正常情况下不开启
    - --gcmode=archive
    - --nodiscover
    - --metrics
    - --http.vhosts=*
    - --http.corsdomain=*
    - --nodiscover
    - --metrics.influxdb
    - --metrics.influxdb.endpoint=http://gethtps-influxdb.geth.svc:8086
    - --metrics.influxdb.tags=host=tps-node

initContainer:
  command: ["geth"]
  args: ["init", "/root/config/genesis.custom.json"]


imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

volumes:
  data:
    mountPath: /root/.ethereum
  config:
    mountPath: /root/config

persistence:
  enabled: true
  storageClass: "local-path"
  annotations: {}
  accessModes:
    - ReadWriteOnce
  size: 50Gi # For mainnet must increase

  ## A manually managed Persistent Volume and Claim
  ## If defined, PVC must be created manually before volume will be bound
  ## The value is evaluated as a template, so, for example, the name can depend on .Release or .Chart
  ##
  # existingClaim:
  ## An existing directory in the node
  ## If defined, host directory must be created manually before volume will be bound
  ## See https://kubernetes.io/docs/concepts/storage/volumes/#hostpath
  ##
  # hostPath:
  #   path: /root/.local/share/io.parity.ethereum
  #   type: Directory

serviceAccount:
  # Specifies whether a service account should be created
  create: true
  # Annotations to add to the service account
  annotations: {}
  # The name of the service account to use.
  # If not set and create is true, a name is generated using the fullname template
  name: ""

podAnnotations: {}

podSecurityContext:
  {}
  # fsGroup: 2000

securityContext:
  {}
  # capabilities:
  #   drop:
  #   - ALL
  # readOnlyRootFilesystem: true
  # runAsNonRoot: true
  # runAsUser: 1000

container:
  ports:
    http: 8545
    ws: 8546
    prometheus: 6060
    listener: 30303
    discovery: 30303

service:
  type: ClusterIP
  http: 8545
  ws: 8546
  prometheus: 6060
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600

serviceP2P:
  type: hostPort
  listener: 30303
  discovery: 30303

prometheus: false

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80
  # targetMemoryUtilizationPercentage: 80

resources: {}
cluster: ${cluster}
nodeSelector:
  geth/role: node
tolerations:
- key: "geth/cluster"
  value: "${cluster}"
  operator: "Equal"
  effect: "NoSchedule"
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: "geth/cluster"
          operator: In
          values:
          - "${cluster}"
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: "geth/cluster"
            operator: In
            values:
            - "${cluster}"
        topologyKey: "kubernetes.io/hostname"

config: |
  [Eth]
  NetworkId = ${chainid}
  SyncMode = "full"

  [Node]
  HTTPHost = "0.0.0.0"
  HTTPModules = ["eth", "net", "web3", "personal", "txpool"]

  [Metrics]
  HTTP = "0.0.0.0"

  [Node.P2P]
  StaticNodes = [${enode1},${enode2},${enode3}]
  TrustedNodes = [${enode1},${enode2},${enode3}]

genesis: '{"config":{"chainId":${chainid},"homesteadBlock":0,"eip150Block":0,"eip150Hash":"0x0000000000000000000000000000000000000000000000000000000000000000","eip155Block":0,"eip158Block":0,"byzantiumBlock":0,"constantinopleBlock":0,"petersburgBlock":0,"istanbulBlock":0,"clique":{"period":${period},"epoch":30000}},"extradata":"0x0000000000000000000000000000000000000000000000000000000000000000${account1}0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000","gasLimit":"${gasLimit}","coinbase":"0x0000000000000000000000000000000000000000","difficulty":"0x1","mixHash":"0x0000000000000000000000000000000000000000000000000000000000000000","alloc":{"0000000000000000000000000000000000000000":{"balance":"0x1"},"${account1}":{"balance":"0x200000000000000000000000000000000000000000000000000000000000000"}},"nonce":"0x0000000000000123","number":"0x0","gasUsed":"0x0","parentHash":"0x0000000000000000000000000000000000000000000000000000000000000000","baseFeePerGas":null}'
key: '${key}'
password: "${password}"

#Additional Environment Variables
env: {}
EOF
```

### 安装普通node

```sh
helm upgrade --install tps-node-geth helm-charts/geth -f tps-node-values.yaml -n geth
```

查看日志

```
kubectl logs tps-node-geth-0 -n geth --tail 100
```

输出`Imported new chain segment` 等为正常

```
INFO [04-07|11:50:13.126] Downloader queue stats                   receiptTasks=0 blockTasks=0 itemSize=339.11B throttle=8192
INFO [04-07|11:50:13.127] Imported new chain segment               blocks=7 txs=0 mgas=0.000 elapsed=1.270ms mgasps=0.000 number=1569 hash=daa064..6cfb68 dirty=0.00B
INFO [04-07|11:50:14.003] Imported new chain segment               blocks=1 txs=0 mgas=0.000 elapsed="257.308µs" mgasps=0.000 number=1570 hash=69b422..5c59c1 dirty=0.00B
```

## 参考

https://liujinye.gitbook.io/eth-docs/deploy/centos8-shi-yong-docker-da-jian-yi-tai-fang-geth-clique-poa-duo-ji-si-you-wang-luo-si-lian

https://clydedcruz.medium.com/geths-insecure-unlock-c28b79dce923