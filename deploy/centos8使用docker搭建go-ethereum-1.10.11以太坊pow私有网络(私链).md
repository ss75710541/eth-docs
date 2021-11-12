# centos8使用docker搭建go-ethereum-1.10.11以太坊pow私有网络(私链)

## 安装 docker

### 卸载centos8已经安装的容器服务

```
yum remove -y runc podman
```

### 安装 docker

```
# (安装 Docker CE)
## 设置仓库
### 安装所需包
yum install -y yum-utils device-mapper-persistent-data lvm2

### 新增 Docker 仓库
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
  
## 安装 Docker CE
sudo yum update -y && sudo yum install -y \
  containerd.io \
  docker-ce \
  docker-ce-cli
  
## 创建 /etc/docker 目录
sudo mkdir /etc/docker

# 设置 Docker daemon
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "registry-mirrors": [
  ]
}
EOF

# Create /etc/systemd/system/docker.service.d
sudo mkdir -p /etc/systemd/system/docker.service.d

# 设置开机启动 docker 并现在启动
sudo systemctl enable docker --now
```

## 定义私有创世状态

首先，需要创建网络的创世状态，所有节点都需要了解并同意该状态。这包括一个小的 JSON 文件（例如称它为 genesis.json）：

- chainId: 为任意正整数
- nonce: 显示从帐户发送的交易数量的计数器。 这将确保交易只处理一次。 在合约帐户中，这个数字代表该帐户创建的合约数量。建议将nonce更改为一些随机值，以防止未知的远程节点能够连接到您。
- 如果您想预先资助一些帐户以便于测试，可以创建帐户并使用它们的地址填充alloc字段。
  -  balance: 这个地址拥有的 Wei 数量。 Wei 是以太币的计数单位，每个 以太币 ETH 有 1e+18 个 Wei。


### pow 创世信息

genesis.json

```json
{
  "config": {
    "chainId": 15, 
    "homesteadBlock": 0,
    "eip150Block": 0,
    "eip155Block": 0,
    "eip158Block": 0,
    "byzantiumBlock": 0,
    "constantinopleBlock": 0,
    "petersburgBlock": 0,
    "istanbulBlock": 0,
    "berlinBlock": 0,
    "londonBlock": 0
  },
  "alloc": {
    "0x0000000000000000000000000000000000000001": {
    "balance": "111111111"
  	},
    "0x0000000000000000000000000000000000000002": {
    "balance": "222222222"
  	}
  },
  "coinbase": "0x0000000000000000000000000000000000000000",
  "difficulty": "0x0",
  "extraData": "",
  "gasLimit": "8000000",
  "nonce": "0x0000000000000123",
  "mixhash": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "timestamp": "0x00"
}
```

### 初始化创世数据

创建 init.sh

```
#!/bin/bash
cd $(dirname $0)
SCRIPTS_DIR=$(pwd)

docker run -it --rm \
           -v ${SCRIPTS_DIR}/genesis.json:/root/files/genesis.json \
           -v /data/geth/data/chain:/root/.ethereum \
           -v /data/geth/data/ethash:/root/.ethash \
	   ethereum/client-go:v1.10.11 \
           init /root/files/genesis.json
```
初始化创建信息

```
sh init.sh
```

### 启动pow创世节点
创建 run.sh

```
#!/bin/bash

cd $(dirname $0)
SCRIPTS_DIR=$(pwd)

docker rm -f ethereum-node

docker run -d --name ethereum-node \
           -v /data/geth/data/chain:/root/.ethereum \
           -v /data/geth/data/ethash:/root/.ethash \
           -v /etc/localtime:/etc/localtime \
           -p 8545:8545 -p 30303:30303 -p 30303:30303/udp \
           ethereum/client-go:v1.10.11 \
           --networkid 15 --http --http.addr 0.0.0.0 --nat extip:172.17.16.174 --http.api "eth,net,web3,personal,admin,miner,txpool" --allow-insecure-unlock --rpc.allow-unprotected-txs --gcmode=archive
```


运行pow创世节点

```
sh run.sh
```

### 创建账号

创建 exec.sh

```
#!/bin/bash

cmd=${1:-admin.nodeInfo}

docker run -it --rm \
           ethereum/client-go:v1.10.11 \
           attach /data/geth/data/geth.ipc --exec $cmd
           #attach http://172.17.16.174:8545 --exec $cmd
```

创建账号

```
sh exec.sh 'personal.newAccount("123456")'
```


### 设置启动 pow 挖矿


创建 miner.sh, 内容如下

```
sh exec.sh 'miner.setEtherbase(eth.accounts[0])'
sh exec.sh 'miner.start(1)'
sh exec.sh 'eth.getBalance(eth.accounts[0])'
sh exec.sh 'eth.blockNumber'
```

```
sh miner.sh
```

### 查看geth node 日志

```
docker logs ethereum-node --tail 100 -f
```

## 参考

[go-ethereum官方github](https://github.com/ethereum/go-ethereum)

[geth官方私有网络教程](https://geth.ethereum.org/docs/getting-started/private-net)

[geth官方安装文档](https://geth.ethereum.org/docs/install-and-build/installing-geth)

https://geth.ethereum.org/docs/interface/javascript-console

[搭建以太坊私链环境](https://yuan1028.github.io/ethereum-test-network/)

[以太坊(Ethereum)私链建立 、合约编译、部署完全教程(1)](http://liyuechun.com/188.html)

[Geth搭建私链](https://donaldhan.github.io/blockchain/2020/05/19/Geth%E6%90%AD%E5%BB%BA%E7%A7%81%E9%93%BE.html)

https://hub.docker.com/r/ethereum/client-go

https://learnblockchain.cn/article/2750

https://ethereum.stackexchange.com/questions/17051/how-to-select-a-network-id-or-is-there-a-list-of-network-ids/17101#17101

https://geth.ethereum.org/docs/interface/mining

https://chainlist.org/