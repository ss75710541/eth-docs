# helm部署geth多机poa私有网络

## 下载geth helm chart 

```sh
git clone https://github.com/paradeum-team/helm-charts
cd helm-charts/geth
```

## 生成初始账号

```shell
for i in {1..3}
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

## 创建 test1 geth

### 创建 test1-values.yaml

```yaml
key=`cat account1.txt`
account="`jq -r '.address' account1.txt`"
password=`cat password1.txt`
cat > test1-values.yaml <<EOF
# Default values for geth.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: ethereum/client-go
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "v1.10.13"
  command: ["geth"]
  args:
    - --config
    - /root/config/config.custom.toml
    - --http
    - --ws
    - --allow-insecure-unlock
    - --rpc.allow-unprotected-txs
    - --gcmode=archive
    - --unlock
    - ${account}
    - --password
    - /root/.ethereum/password
    - --mine
    - --nodiscover
    - --metrics

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
  size: 20Gi # For mainnet must increase

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
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80
  # targetMemoryUtilizationPercentage: 80

resources: {}
cluster: test
nodeSelector:
  geth/cluster: test
tolerations: []
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: "geth/cluster"
            operator: In
            values:
            - "test"
        topologyKey: "kubernetes.io/hostname"

config: |
  [Eth]
  NetworkId = 65544
  SyncMode = "full"

  [Node]
  HTTPHost = "0.0.0.0"
  HTTPModules = ["eth", "net", "web3", "txpool"]

  [Metrics]
  HTTP = "0.0.0.0"

  [Node.P2P]
  StaticNodes = []
  TrustedNodes = []

genesis: '{"config":{"chainId":65534,"homesteadBlock":0,"eip150Block":0,"eip150Hash":"0x0000000000000000000000000000000000000000000000000000000000000000","eip155Block":0,"eip158Block":0,"byzantiumBlock":0,"constantinopleBlock":0,"petersburgBlock":0,"istanbulBlock":0,"clique":{"period":5,"epoch":30000}},"extradata":"0x0000000000000000000000000000000000000000000000000000000000000000${account}0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000","gasLimit":"8000000","coinbase":"0x0000000000000000000000000000000000000000","difficulty":"0x1","mixHash":"0x0000000000000000000000000000000000000000000000000000000000000000","alloc":{"0000000000000000000000000000000000000000":{"balance":"0x1"},"${account}":{"balance":"0x200000000000000000000000000000000000000000000000000000000000000"}},"nonce":"0x0000000000000123","number":"0x0","gasUsed":"0x0","parentHash":"0x0000000000000000000000000000000000000000000000000000000000000000","baseFeePerGas":null}'
key: '${key}'
password: "${password}"

#Additional Environment Variables
env: {}
EOF
```

### 执行安装test1 geth

```shell
helm upgrade --install test1-geth geth -f test1-values.yaml -n geth --create-namespace
```

### 查看日志

```shell
kubectl -n geth logs test1-geth-0
```

显示 🔗`block reached canonical chain`  等字样为正常

```
INFO [12-23|09:03:09.000] 🔗 block reached canonical chain          number=280 hash=acef2d..e384ab
INFO [12-23|09:03:09.000] 🔨 mined potential block                  number=287 hash=694829..02f045
INFO [12-23|09:03:09.001] Commit new mining work                   number=288 sealhash=a71eb4..7d13ad uncles=0 txs=0 gas=0 fees=0 elapsed="403.942µs"
INFO [12-23|09:03:14.000] Successfully sealed new block            number=288 sealhash=a71eb4..7d13ad hash=80719a..591ec6 elapsed=4.999s
INFO [12-23|09:03:14.000] 🔗 block reached canonical chain          number=281 hash=fd8583..08b2a7
INFO [12-23|09:03:14.000] 🔨 mined potential block                  number=288 hash=80719a..591ec6
INFO [12-23|09:03:14.001] Commit new mining work                   number=289 sealhash=fa20e6..d6280f uncles=0 txs=0 gas=0 fees=0 elapsed="333.05µs"
INFO [12-23|09:03:19.000] Successfully sealed new block            number=289 sealhash=fa20e6..d6280f hash=4805c3..5eb5cf elapsed=4.999s
```

### 查看 enode 信息

```shell
kubectl exec  test1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.nodeInfo.enode
```

## 创建test2 geth

### 创建test2-values.yaml

```yaml
# 获取test1 geth enode
test1_geth_hostip=`kubectl get pod test1-geth-0 -o jsonpath="{.status.hostIP}"`
enode1=`kubectl exec  test1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.nodeInfo.enode 2>&1|grep enode|sed "s/127.0.0.1/${test1_geth_hostip}/g"`
key=`cat account2.txt`
# 这里account1 是固定的，所有节点初始化信息中的预置账号都是account1
account1="`jq -r '.address' account1.txt`"
account="`jq -r '.address' account2.txt`"
password=`cat password2.txt`
cat > test2-values.yaml <<EOF
# Default values for geth.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: ethereum/client-go
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "v1.10.13"
  command: ["geth"]
  args:
    - --config
    - /root/config/config.custom.toml
    - --http
    - --ws
    - --allow-insecure-unlock
    - --rpc.allow-unprotected-txs
    - --gcmode=archive
    - --unlock
    - ${account}
    - --password
    - /root/.ethereum/password
    - --mine
    - --nodiscover
    - --metrics

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
  size: 20Gi # For mainnet must increase

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
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80
  # targetMemoryUtilizationPercentage: 80

resources: {}
cluster: test
nodeSelector:
  geth/cluster: test
tolerations: []
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: "geth/cluster"
            operator: In
            values:
            - "test"
        topologyKey: "kubernetes.io/hostname"

config: |
  [Eth]
  NetworkId = 65544
  SyncMode = "full"

  [Node]
  HTTPHost = "0.0.0.0"
  HTTPModules = ["eth", "net", "web3", "txpool"]

  [Metrics]
  HTTP = "0.0.0.0"

  [Node.P2P]
  StaticNodes = [${enode1}]
  TrustedNodes = [${enode1}]

genesis: '{"config":{"chainId":65534,"homesteadBlock":0,"eip150Block":0,"eip150Hash":"0x0000000000000000000000000000000000000000000000000000000000000000","eip155Block":0,"eip158Block":0,"byzantiumBlock":0,"constantinopleBlock":0,"petersburgBlock":0,"istanbulBlock":0,"clique":{"period":5,"epoch":30000}},"extradata":"0x0000000000000000000000000000000000000000000000000000000000000000${account1}0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000","gasLimit":"8000000","coinbase":"0x0000000000000000000000000000000000000000","difficulty":"0x1","mixHash":"0x0000000000000000000000000000000000000000000000000000000000000000","alloc":{"0000000000000000000000000000000000000000":{"balance":"0x1"},"${account1}":{"balance":"0x200000000000000000000000000000000000000000000000000000000000000"}},"nonce":"0x0000000000000123","number":"0x0","gasUsed":"0x0","parentHash":"0x0000000000000000000000000000000000000000000000000000000000000000","baseFeePerGas":null}'
key: '${key}'
password: "${password}"

#Additional Environment Variables
env: {}
EOF
```

### 安装test2 geth

```shell
helm upgrade --install test2-geth geth -f test2-values.yaml -n geth
```

### 查看日志

```shell
kubectl -n geth logs test2-geth-0
```

显示 `unauthorized signer` ` Imported new chain segment` 等字样为正常

```
WARN [12-23|09:02:13.917] Block sealing failed                     err="unauthorized signer"
INFO [12-23|09:02:23.113] Block synchronisation started
INFO [12-23|09:02:23.114] Mining aborted due to sync
INFO [12-23|09:02:23.223] Downloader queue stats                   receiptTasks=0 blockTasks=0 itemSize=650.00B throttle=8192
INFO [12-23|09:02:23.247] Imported new chain segment               blocks=192 txs=0 mgas=0.000 elapsed=23.500ms    mgasps=0.000 number=192 hash=4e50e5..d11949 age=7m9s     dirty=0.00B
INFO [12-23|09:02:26.239] Imported new chain segment               blocks=84  txs=0 mgas=0.000 elapsed=13.882ms    mgasps=0.000 number=276 hash=e07bc7..6da4fa dirty=0.00B
INFO [12-23|09:02:29.228] Imported new chain segment               blocks=3   txs=0 mgas=0.000 elapsed=1.124ms     mgasps=0.000 number=279 hash=188c5f..f57ce0 dirty=0.00B
```

### 查看test1 geth peers

```shell
kubectl exec  test1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.peers
```

有一项连接记录为正常

```shell
Defaulted container "geth" out of: geth, geth-init (init)
[{
    caps: ["eth/66", "snap/1"],
    enode: "enode://a1fba8291cd46ed9f902d7f1548828ce884142be31999b36de3f990d5320580c48b652917400f2d1600394e26d411e180a364ace797e53152c97c770298617dd@172.17.8.171:48108",
    id: "45d06af81f4a4af63747312592794848274c017b375221412e0e432bac898820",
    name: "Geth/v1.10.13-stable-7a0c19f8/linux-amd64/go1.17.3",
    network: {
      inbound: true,
      localAddress: "10.128.2.30:30303",
      remoteAddress: "172.17.8.171:48108",
      static: false,
      trusted: false
    },
    protocols: {
      eth: {
        difficulty: 1,
        head: "0x0abda3a91b68db87018d217b7987791e8dc74a211a42849c56457c1c249b7f32",
        version: 66
      },
      snap: {
        version: 1
      }
    }
}]
```

## 创建test3 geth

### 创建test3-values.yaml

```yaml
# 获取test geth enode
test1_geth_hostip=`kubectl get pod test1-geth-0 -o jsonpath="{.status.hostIP}"`
enode1=`kubectl exec  test1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.nodeInfo.enode 2>&1|grep enode|sed "s/127.0.0.1/${test1_geth_hostip}/g"`
test2_geth_hostip=`kubectl get pod test2-geth-0 -o jsonpath="{.status.hostIP}"`
enode2=`kubectl exec  test2-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.nodeInfo.enode 2>&1|grep enode|sed "s/127.0.0.1/${test2_geth_hostip}/g"`
key=`cat account3.txt`
# 这里account1 是固定的，所有节点初始化信息中的预置账号都是account1
account1="`jq -r '.address' account1.txt`"
account="`jq -r '.address' account3.txt`"
password=`cat password3.txt`
cat > test3-values.yaml <<EOF
# Default values for geth.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: ethereum/client-go
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "v1.10.13"
  command: ["geth"]
  args:
    - --config
    - /root/config/config.custom.toml
    - --http
    - --ws
    - --allow-insecure-unlock
    - --rpc.allow-unprotected-txs
    - --gcmode=archive
    - --unlock
    - ${account}
    - --password
    - /root/.ethereum/password
    - --mine
    - --nodiscover
    - --metrics

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
  size: 20Gi # For mainnet must increase

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
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80
  # targetMemoryUtilizationPercentage: 80

resources: {}
cluster: test
nodeSelector:
  geth/cluster: test
tolerations: []
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: "geth/cluster"
            operator: In
            values:
            - "test"
        topologyKey: "kubernetes.io/hostname"

config: |
  [Eth]
  NetworkId = 65544
  SyncMode = "full"

  [Node]
  HTTPHost = "0.0.0.0"
  HTTPModules = ["eth", "net", "web3", "txpool"]

  [Metrics]
  HTTP = "0.0.0.0"

  [Node.P2P]
  StaticNodes = [${enode1},${enode2}]
  TrustedNodes = [${enode1},${enode2}]

genesis: '{"config":{"chainId":65534,"homesteadBlock":0,"eip150Block":0,"eip150Hash":"0x0000000000000000000000000000000000000000000000000000000000000000","eip155Block":0,"eip158Block":0,"byzantiumBlock":0,"constantinopleBlock":0,"petersburgBlock":0,"istanbulBlock":0,"clique":{"period":5,"epoch":30000}},"extradata":"0x0000000000000000000000000000000000000000000000000000000000000000${account1}0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000","gasLimit":"8000000","coinbase":"0x0000000000000000000000000000000000000000","difficulty":"0x1","mixHash":"0x0000000000000000000000000000000000000000000000000000000000000000","alloc":{"0000000000000000000000000000000000000000":{"balance":"0x1"},"${account1}":{"balance":"0x200000000000000000000000000000000000000000000000000000000000000"}},"nonce":"0x0000000000000123","number":"0x0","gasUsed":"0x0","parentHash":"0x0000000000000000000000000000000000000000000000000000000000000000","baseFeePerGas":null}'
key: '${key}'
password: "${password}"

#Additional Environment Variables
env: {}
EOF
```

### 安装test3 geth

```shell
helm upgrade --install test3-geth geth -f test3-values.yaml -n geth
```

### 查看日志

```shell
kubectl -n geth logs test3-geth-0
```

显示 `unauthorized signer` ` Imported new chain segment` 等字样为正常

```
WARN [12-23|09:02:13.917] Block sealing failed                     err="unauthorized signer"
INFO [12-23|09:02:23.113] Block synchronisation started
INFO [12-23|09:02:23.114] Mining aborted due to sync
INFO [12-23|09:02:23.223] Downloader queue stats                   receiptTasks=0 blockTasks=0 itemSize=650.00B throttle=8192
INFO [12-23|09:02:23.247] Imported new chain segment               blocks=192 txs=0 mgas=0.000 elapsed=23.500ms    mgasps=0.000 number=192 hash=4e50e5..d11949 age=7m9s     dirty=0.00B
INFO [12-23|09:02:26.239] Imported new chain segment               blocks=84  txs=0 mgas=0.000 elapsed=13.882ms    mgasps=0.000 number=276 hash=e07bc7..6da4fa dirty=0.00B
INFO [12-23|09:02:29.228] Imported new chain segment               blocks=3   txs=0 mgas=0.000 elapsed=1.124ms     mgasps=0.000 number=279 hash=188c5f..f57ce0 dirty=0.00B
```

### 查看test1 geth peers

```
kubectl exec  test1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.peers
```

有两项连接记录为正常

```shell
Defaulted container "geth" out of: geth, geth-init (init)
[{
    caps: ["eth/66", "snap/1"],
    enode: "enode://a1fba8291cd46ed9f902d7f1548828ce884142be31999b36de3f990d5320580c48b652917400f2d1600394e26d411e180a364ace797e53152c97c770298617dd@172.17.8.171:48108",
    id: "45d06af81f4a4af63747312592794848274c017b375221412e0e432bac898820",
    name: "Geth/v1.10.13-stable-7a0c19f8/linux-amd64/go1.17.3",
    network: {
      inbound: true,
      localAddress: "10.128.2.30:30303",
      remoteAddress: "172.17.8.171:48108",
      static: false,
      trusted: false
    },
    protocols: {
      eth: {
        difficulty: 1,
        head: "0x0abda3a91b68db87018d217b7987791e8dc74a211a42849c56457c1c249b7f32",
        version: 66
      },
      snap: {
        version: 1
      }
    }
}]
```

### 修改三个geth 初始连接enode

#### 修改test*-values.yaml

```shell
# 获取test geth enode
test1_geth_hostip=`kubectl get pod test1-geth-0 -o jsonpath="{.status.hostIP}"`
enode1=`kubectl exec  test1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.nodeInfo.enode 2>&1|grep enode|sed "s/127.0.0.1/${test1_geth_hostip}/g"`
test2_geth_hostip=`kubectl get pod test2-geth-0 -o jsonpath="{.status.hostIP}"`
enode2=`kubectl exec  test2-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.nodeInfo.enode 2>&1|grep enode|sed "s/127.0.0.1/${test2_geth_hostip}/g"`
test3_geth_hostip=`kubectl get pod test3-geth-0 -o jsonpath="{.status.hostIP}"`
enode3=`kubectl exec  test3-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.nodeInfo.enode 2>&1|grep enode|sed "s/127.0.0.1/${test3_geth_hostip}/g"`

sed -i "s#StaticNodes.*#StaticNodes = [${enode1},${enode2},${enode3}]#g" test{1,2,3}-values.yaml
sed -i "s#TrustedNodes.*#TrustedNodes = [${enode1},${enode2},${enode3}]#g" test{1,2,3}-values.yaml
```

#### 更新test*-geth

```shell
helm upgrade --install test1-geth geth -f test1-values.yaml -n geth --create-namespace
helm upgrade --install test2-geth geth -f test2-values.yaml -n geth --create-namespace
helm upgrade --install test3-geth geth -f test3-values.yaml -n geth --create-namespace
```

#### 查看peers

```shell
kubectl exec  test1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.peers
kubectl exec  test2-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.peers
kubectl exec  test3-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.peers
```

从三个pod 跟查peers 都为两个连接为正常

## 设置签名账号

获取三个账号地址

```shell
acc1="0x`jq -r '.address' account1.txt`"
acc2="0x`jq -r '.address' account2.txt`"
acc3="0x`jq -r '.address' account3.txt`"
```

提名test2-geth 初始账号为签名账号

```shell
# 在 test1 geth 提名acc2 账号
kubectl exec  test1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  "clique.propose('$acc2', true)"

# 输出
Defaulted container "geth" out of: geth, geth-init (init)
null

# 查看签名账号列表
kubectl exec  test1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  'clique.getSnapshot().signers'

# 输出2个账号，分别为test1-geth 和 test2-geth 的初始账号
Defaulted container "geth" out of: geth, geth-init (init)
{
  0x2caf0dd917cd0bf69fef580faf718356b4e7a930: {},
  0xb6e44ba48d742ae915567874ee91a4aeacded421: {}
}
```

提名test3-geth 初始账号为签名账号(第三个账号提名需要一半以上签名者提名)

```shell
kubectl exec  test1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  "clique.propose('$acc3', true)"
kubectl exec  test2-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  "clique.propose('$acc3', true)"

kubectl exec  test1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  'clique.getSnapshot().signers'

# 输出三个签名者为正常
Defaulted container "geth" out of: geth, geth-init (init)
{
  0x2caf0dd917cd0bf69fef580faf718356b4e7a930: {},
  0x87962f20925156b4ccf20408d712d34d9f1c260b: {},
  0xb6e44ba48d742ae915567874ee91a4aeacded421: {}
}
```

查看各test geth 日志，显示 出块信息为正常

```shell
# 查看各test3 geth 日志
kubectl -n geth logs test3-geth-0 

# 输出
INFO [12-23|09:57:24.000] 🔨 mined potential block                  number=938 hash=2711d7..5b8678
INFO [12-23|09:57:24.001] Commit new mining work                   number=939 sealhash=16af20..87521d uncles=0 txs=0 gas=0 fees=0 elapsed="200.786µs"
WARN [12-23|09:57:24.001] Block sealing failed                     err="signed recently, must wait for others"
INFO [12-23|09:57:29.006] Imported new chain segment               blocks=1 txs=0 mgas=0.000 elapsed="253.698µs" mgasps=0.000 number=939 hash=eab9ba..89ef7f dirty=0.00B
INFO [12-23|09:57:29.006] 🔗 block reached canonical chain          number=932 hash=3efb62..1fbc6c
INFO [12-23|09:57:29.006] Commit new mining work                   number=940 sealhash=0e0009..8af088 uncles=0 txs=0 gas=0 fees=0 elapsed="135.103µs"
INFO [12-23|09:57:34.005] Imported new chain segment               blocks=1 txs=0 mgas=0.000 elapsed="219.807µs" mgasps=0.000 number=940 hash=6eba55..00e5ec dirty=0.00B
INFO [12-23|09:57:34.006] Commit new mining work                   number=941 sealhash=493afa..0826b0 uncles=0 txs=0 gas=0 fees=0 elapsed="128.214µs"
```

## 测试交易

```shell
acc1="0x`jq -r '.address' account1.txt`"
acc2="0x`jq -r '.address' account2.txt`"
kubectl exec -it test1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec "amount = web3.toWei(0.01); eth.sendTransaction({from: '$acc1', to: '$acc2', value: amount});"

# 输出
"0xae9a921e9d5998ec6a4064bd8f5fc2bbc499a793d7476e9b11800b8868a6a048"

kubectl exec -it test1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec "eth.getBalance('$acc2')"
kubectl exec -it test2-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec "eth.getBalance('$acc2')"
kubectl exec -it test3-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec "eth.getBalance('$acc2')"

# 结果都输出为正常
Defaulted container "geth" out of: geth, geth-init (init)
10021000000000000
```

## 添加普通node

### 创建 test-node-values.yaml

```yaml
# 获取test geth enode
test1_geth_hostip=`kubectl get pod test1-geth-0 -o jsonpath="{.status.hostIP}"`
enode1=`kubectl exec  test1-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.nodeInfo.enode 2>&1|grep enode|sed "s/127.0.0.1/${test1_geth_hostip}/g"`
test2_geth_hostip=`kubectl get pod test2-geth-0 -o jsonpath="{.status.hostIP}"`
enode2=`kubectl exec  test2-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.nodeInfo.enode 2>&1|grep enode|sed "s/127.0.0.1/${test2_geth_hostip}/g"`
test3_geth_hostip=`kubectl get pod test3-geth-0 -o jsonpath="{.status.hostIP}"`
enode3=`kubectl exec  test3-geth-0 -n geth -- geth attach /root/.ethereum/geth.ipc --exec  admin.nodeInfo.enode 2>&1|grep enode|sed "s/127.0.0.1/${test3_geth_hostip}/g"`

cat > test-node-values.yaml <<EOF
# Default values for geth.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 2

image:
  repository: ethereum/client-go
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "v1.10.13"
  command: ["geth"]
  args:
    - --config
    - /root/config/config.custom.toml
    - --http
    - --ws
    - --ws.addr=0.0.0.0
    - --ws.origins=*
    - --allow-insecure-unlock
    - --rpc.allow-unprotected-txs
    - --gcmode=archive
    - --nodiscover
    - --metrics
    - --metrics.addr=0.0.0.0
    - --http.vhosts=*
    - --http.corsdomain=*

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
  size: 20Gi # For mainnet must increase

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

#serviceP2P:
#  type: hostPort
#  listener: 30303
#  discovery: 30303

prometheus: false

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80
  # targetMemoryUtilizationPercentage: 80

resources: {}
cluster: test
nodeSelector:
  geth/cluster: test
tolerations: []
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: "geth/cluster"
            operator: In
            values:
            - "test"
        topologyKey: "kubernetes.io/hostname"

config: |
  [Eth]
  NetworkId = 65534
  SyncMode = "full"

  [Node]
  HTTPHost = "0.0.0.0"
  HTTPModules = ["eth", "net", "web3", "txpool"]

  [Metrics]
  HTTP = "0.0.0.0"

  [Node.P2P]
  StaticNodes = [${enode1},${enode2},${enode3}]
  TrustedNodes = [${enode1},${enode2},${enode3}]

genesis: '{"config":{"chainId":65534,"homesteadBlock":0,"eip150Block":0,"eip150Hash":"0x0000000000000000000000000000000000000000000000000000000000000000","eip155Block":0,"eip158Block":0,"byzantiumBlock":0,"constantinopleBlock":0,"petersburgBlock":0,"istanbulBlock":0,"clique":{"period":5,"epoch":30000}},"extradata":"0x0000000000000000000000000000000000000000000000000000000000000000b6e44ba48d742ae915567874ee91a4aeacded4210000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000","gasLimit":"8000000","coinbase":"0x0000000000000000000000000000000000000000","difficulty":"0x1","mixHash":"0x0000000000000000000000000000000000000000000000000000000000000000","alloc":{"0000000000000000000000000000000000000000":{"balance":"0x1"},"b6e44ba48d742ae915567874ee91a4aeacded421":{"balance":"0x200000000000000000000000000000000000000000000000000000000000000"}},"nonce":"0x0000000000000123","number":"0x0","gasUsed":"0x0","parentHash":"0x0000000000000000000000000000000000000000000000000000000000000000","baseFeePerGas":null}'

#Additional Environment Variables
env: {}
EOF
```

### 安装普通node

```sh
helm upgrade --install test-node-geth geth -f test-node-values.yaml -n geth
```