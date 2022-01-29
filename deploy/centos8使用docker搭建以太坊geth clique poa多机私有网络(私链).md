# centos8使用 docker 搭建高可用以太坊geth clique poa多机私有网络(私链)


## 主机规划

ip|用途
-----|-----
172.17.103.114|node1、初始创世节点、共识签名节点
172.17.53.207|node2、共识签名节点
172.17.66.205|node3、共识签名节点

## 初始化节点

登录 node1节点

### 创建工作目录 

```
mkdir ~/geth
cd ~/geth
```

### 创建账号

```
# 生成随机密码
password=`< /dev/urandom tr -dc _A-Z-a-z-0-9 | head -c${1:-12};echo;`
echo -n "$password" > password.txt
mkdir -p /data/geth/data/chain/
cp password.txt /data/geth/data/chain/
# 根据密码文件创建账号
docker run -it --rm \
           -v /data/geth/data/chain:/root/.ethereum \
           -v /data/geth/data/ethash:/root/.ethash \
           -v /etc/localtime:/etc/localtime \
           ethereum/client-go:v1.10.11 account new --password /root/.ethereum/password.txt
# 账号密钥信息复制到当前目录
cp /data/geth/data/chain/keystore/* .
```

### 创建创世信息文件

```
# 私有网络 id
chainId=65534
# 账号预置 eth 数量 
balance=0x200000000000000000000000000000000000000000000000000000000000000
# 根据创建的账号查询账号地址，该账号设置为默认的签名账号
initAccount=`jq -r '.address' $(ls -1 UTC*|head -1)`

cat > genesis.json<<EOF
{
  "config": {
    "chainId": 65534,
    "homesteadBlock": 0,
    "eip150Block": 0,
    "eip150Hash": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "eip155Block": 0,
    "eip158Block": 0,
    "byzantiumBlock": 0,
    "constantinopleBlock": 0,
    "petersburgBlock": 0,
    "istanbulBlock": 0,
    "clique": {
      "period": 5,
      "epoch": 30000
    }
  },
  "extradata":"0x0000000000000000000000000000000000000000000000000000000000000000${initAccount}0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
  "gasLimit": "8000000",
  "coinbase": "0x0000000000000000000000000000000000000000",
  "difficulty": "0x1",
  "mixHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "alloc": {
    "0000000000000000000000000000000000000000": {
      "balance": "0x1"
    },
    "${initAccount}": {
      "balance": "${balance}"
    }
  },
  "nonce": "0x0000000000000123",
  "number": "0x0",
  "gasUsed": "0x0",
  "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "baseFeePerGas": null
}
EOF
```


### 执行初始化创世信息

```
SCRIPTS_DIR=$(pwd)
docker run -it --rm \
           -v ${SCRIPTS_DIR}/genesis.json:/root/files/genesis.json \
           -v /data/geth/data/chain:/root/.ethereum \
           -v /data/geth/data/ethash:/root/.ethash \
           ethereum/client-go:v1.10.11 \
           init /root/files/genesis.json
```

## 运行创世节点

创建 run.sh, 酌情修改 extip, 如果是外网连接填写外网 ip。启动时unlock 的账号默认为 poa 签名账号

```
#!/bin/bash

cd $(dirname $0)
SCRIPTS_DIR=$(pwd)

initAccount=`jq -r '.address' $(ls -1 UTC*|head -1)`

docker rm -f ethereum-node

docker run -d --name ethereum-node \
           -v /data/geth/data/chain:/root/.ethereum \
           -v /data/geth/data/ethash:/root/.ethash \
           -v /etc/localtime:/etc/localtime \
	   -p 8545:8545 -p 30303:30303 \
           registry.hisun.netwarps.com/ethereum/client-go:v1.10.11 \
           --networkid 65534 \
           --http --http.addr 0.0.0.0 --nat extip:172.17.16.174 --http.api "eth,net,web3,txpool,debug" \
	   --syncmode 'full' --allow-insecure-unlock --rpc.allow-unprotected-txs  --gcmode=archive \
	   --unlock "$initAccount"  --password /root/.ethereum/password.txt --miner.etherbase ${initAccount} --mine \
	   -verbosity 3  --nodiscover
```

启动创世节点

```
# sh run.sh
```

查看签名账号列表

```
docker run -it --rm \
           -v /data/geth/data/chain:/root/.ethereum \
           ethereum/client-go:v1.10.11 \
           attach /root/.ethereum/geth.ipc --exec  admin.nodeInfo.enode 'clique.getSnapshot().signers'
{
  0xc715d67b1cb004fa90cfcdc6026e464cd982ee81: {}
}
```


## 启动其它 peer 节点

### 创建工作目录

在 node2 / node3 执行

```
mkdir -p ~/geth
cd ~/geth
```

复制 genesis.json 到 node2、node3 节点, 所有节点的 创世信息必须相同，否则不能添加 peer


### 执行初始化创世信息

```
# 清理历史数据
rm -f UTC*
rm -rf /data/geth/data/

SCRIPTS_DIR=$(pwd)
docker run -it --rm \
           -v ${SCRIPTS_DIR}/genesis.json:/root/files/genesis.json \
           -v /data/geth/data/chain:/root/.ethereum \
           -v /data/geth/data/ethash:/root/.ethash \
           ethereum/client-go:v1.10.11 \
           init /root/files/genesis.json
```

### 创建账号

```
# 生成随机密码
password=`< /dev/urandom tr -dc _A-Z-a-z-0-9 | head -c${1:-12};echo;`
echo -n "$password" > password.txt
\cp password.txt /data/geth/data/chain/
# 根据密码文件创建账号
docker run -it --rm \
           -v /data/geth/data/chain:/root/.ethereum \
           -v /data/geth/data/ethash:/root/.ethash \
           -v /etc/localtime:/etc/localtime \
           ethereum/client-go:v1.10.11 account new --password /root/.ethereum/password.txt
# 账号密钥信息复制到当前目录
cp /data/geth/data/chain/keystore/* .
```

### 启动 peer 节点

创建 run.sh, 酌情修改 extip, 如果是外网连接填写外网 ip。默认 unlock 的账号，后面通过创世节点添加为签名账号

```
#!/bin/bash

cd $(dirname $0)
SCRIPTS_DIR=$(pwd)

initAccount=`jq -r '.address' $(ls -1 UTC*|head -1)`

docker rm -f ethereum-node

docker run -d --name ethereum-node \
           -v /data/geth/data/chain:/root/.ethereum \
           -v /data/geth/data/ethash:/root/.ethash \
           -v /etc/localtime:/etc/localtime \
	   -p 8545:8545 -p 30303:30303 \
           registry.hisun.netwarps.com/ethereum/client-go:v1.10.11 \
           --networkid 65534 \
           --http --http.addr 0.0.0.0 --nat extip:172.17.66.205 --http.api "eth,net,web3,txpool,debug" \
           --syncmode 'full' --allow-insecure-unlock --rpc.allow-unprotected-txs  --gcmode=archive \
           --unlock  ${initAccount} --password /root/.ethereum/password.txt --miner.etherbase ${initAccount} \
           --nodiscover
```

运行 peer 节点

```
sh run.sh
```

查看日志

```
docker logs ethereum-node
```

### 连接 peer 节点

查看所有节点的 enode 连接信息

```
docker run -it --rm \
           -v /data/geth/data/chain:/root/.ethereum \
           ethereum/client-go:v1.10.11 \
           attach /root/.ethereum/geth.ipc --exec  admin.nodeInfo.enode
         
# 三个peer 节点分别显示如下内容  
"enode://c75ceb06b0af77b88da08d6a66fafb8e0aab2ae840785799864fa05deb78a3234204b93fc7807aefd1cc0749483682863fae945cb66e0fd2109bc210b85b8cfd@172.17.16.174:30303?discport=0"

"enode://aedb373c5b0ff92275de4685140a7c8e6f9545f530645f8386b5d9c3f069a8a5ffe03117fb2863f48705222860d834ce590ca251f4f63031fde270889ba9c132@172.17.53.207:30303?discport=0"

"enode://9f6828477034970dbf896298a1e8a906536fbe6365bd4e732210d0f77737fbcf456151cee72a8745f85f08f2c5575ef3a896f427282ba59caf34efa75ccbbf98@172.17.66.205:30303?discport=0"
```

创建 config.toml, enode 信息参考上面, 

```
cat > cofnig.toml<<EOF
[Node.P2P]
# 最大 peer 数量
MaxPeers = 50
# 自动发现其它 peer
NoDiscovery = false
# 静态节点，默认连接
StaticNodes = ["enode://c75ceb06b0af77b88da08d6a66fafb8e0aab2ae840785799864fa05deb78a3234204b93fc7807aefd1cc0749483682863fae945cb66e0fd2109bc210b85b8cfd@172.17.16.174:30303?discport=0","enode://aedb373c5b0ff92275de4685140a7c8e6f9545f530645f8386b5d9c3f069a8a5ffe03117fb2863f48705222860d834ce590ca251f4f63031fde270889ba9c132@172.17.53.207:30303?discport=0","enode://9f6828477034970dbf896298a1e8a906536fbe6365bd4e732210d0f77737fbcf456151cee72a8745f85f08f2c5575ef3a896f427282ba59caf34efa75ccbbf98@1172.17.66.205:30303?discport=0"]
# 可信节点，超出最大 peer 数量，也会连接
TrustedNodes = ["enode://c75ceb06b0af77b88da08d6a66fafb8e0aab2ae840785799864fa05deb78a3234204b93fc7807aefd1cc0749483682863fae945cb66e0fd2109bc210b85b8cfd@172.17.16.174:30303?discport=0","enode://aedb373c5b0ff92275de4685140a7c8e6f9545f530645f8386b5d9c3f069a8a5ffe03117fb2863f48705222860d834ce590ca251f4f63031fde270889ba9c132@172.17.53.207:30303?discport=0","enode://9f6828477034970dbf896298a1e8a906536fbe6365bd4e732210d0f77737fbcf456151cee72a8745f85f08f2c5575ef3a896f427282ba59caf34efa75ccbbf98@1172.17.66.205:30303?discport=0"]
ListenAddr = ":30303"
EnableMsgEvents = false
EOF
```

修改所有节点run.sh 增加 toml 配置,示例如下

```
#!/bin/bash

cd $(dirname $0)
SCRIPTS_DIR=$(pwd)


initAccount=`jq -r '.address' $(ls -1 UTC*|head -1)`

docker rm -f ethereum-node

docker run -d --name ethereum-node \
	   -v ${SCRIPTS_DIR}/config.toml:/root/config.toml \
           -v /data/geth/data/chain:/root/.ethereum \
           -v /data/geth/data/ethash:/root/.ethash \
           -v /etc/localtime:/etc/localtime \
	   -p 8545:8545 -p 30303:30303 -p 30303:30303/udp \
           ethereum/client-go:v1.10.12 \
           --networkid 65534 \
           --http --http.addr 0.0.0.0 --nat extip:117.50.18.201 --http.api "eth,net,web3,txpool,debug" \
	   --syncmode 'full' --allow-insecure-unlock --rpc.allow-unprotected-txs  --gcmode=archive \
	   --unlock "$initAccount"  --password /root/.ethereum/password.txt --miner.etherbase ${initAccount} --mine \
	   -verbosity 3  --nodiscover --config /root/config.toml
```

执行 run.sh 重启节点

```
sh run.sh
```

所有节点查看 已经连接的 peers，三个节点都显示两个连接的 peer 信息表示正常

```
docker run -it --rm \
           -v /data/geth/data/chain:/root/.ethereum \
           ethereum/client-go:v1.10.11 \
           attach /root/.ethereum/geth.ipc --exec admin.peers
           
# 显示如下内容          
[{
    caps: ["eth/66", "snap/1"],
    enode: "enode://aedb373c5b0ff92275de4685140a7c8e6f9545f530645f8386b5d9c3f069a8a5ffe03117fb2863f48705222860d834ce590ca251f4f63031fde270889ba9c132@172.17.53.207:30303?discport=0",
    id: "339f032b16f80ccb0b07dd62fa1f91352e477974afffc7359dc1d7c57f49cee6",
    name: "Geth/v1.10.11-stable-7231b3ef/linux-amd64/go1.17.2",
    network: {
      inbound: false,
      localAddress: "172.18.0.2:53254",
      remoteAddress: "172.17.53.207:30303",
      static: true,
      trusted: true
    },
    protocols: {
      eth: {
        difficulty: 209,
        head: "0x725de819ae8fa2e426cb7d3ebd5cd7c27f16c50910fda4f0c22b3eb3f402e0e8",
        version: 66
      },
      snap: {
        version: 1
      }
    }
}, {
    caps: ["eth/66", "snap/1"],
    enode: "enode://9f6828477034970dbf896298a1e8a906536fbe6365bd4e732210d0f77737fbcf456151cee72a8745f85f08f2c5575ef3a896f427282ba59caf34efa75ccbbf98@172.17.66.205:30303?discport=0",
    id: "94b0aee8a94d49537c62253cf1a2d105fca4e494b41444165bad7ebc16f2213f",
    name: "Geth/v1.10.11-stable-7231b3ef/linux-amd64/go1.17.2",
    network: {
      inbound: false,
      localAddress: "172.18.0.2:45490",
      remoteAddress: "172.17.66.205:30303",
      static: true,
      trusted: true
    },
    protocols: {
      eth: {
        difficulty: 207,
        head: "0x81b94425c579bc58f2f97d171787e82d3b5b6ab1e2e2c024500c9ce8f54ca917",
        version: 66
      },
      snap: {
        version: 1
      }
    }
}]
```

### 提名其它签名账号

在 node2 查看初始的 unlock 账号

```
docker run -it --rm \
           -v /data/geth/data/chain:/root/.ethereum \
           ethereum/client-go:v1.10.11 \
           attach /root/.ethereum/geth.ipc --exec eth.accounts[0]

"0x2dac2c0b6e3d6e0cf11c5a6028230ef2ffc82565"
```

在 node1 节点执行, 提名 node2 peer 启动时 unlock 的账号为签名者，并查看签名者列表(签名者列表更新有延迟，可以等2s 再执行)

```
docker run -it --rm \
           -v /data/geth/data/chain:/root/.ethereum \
           ethereum/client-go:v1.10.11 \
           attach /root/.ethereum/geth.ipc

> acc="0x2dac2c0b6e3d6e0cf11c5a6028230ef2ffc82565"
> clique.propose(acc, true)
null
>
> clique.getSnapshot().signers
{
  0xae1556f5f826d8c7257b4d1113428f4a02910d6e: {},
  0xc715d67b1cb004fa90cfcdc6026e464cd982ee81: {}
}
```

在 node2 节点修改启动脚本 run.sh ，添加 --mine 参数默认启动矿工, (只有启动矿工签名账号才能签名)

```
#!/bin/bash

cd $(dirname $0)

initAccount=`jq -r '.address' $(ls -1 UTC*|head -1)`

docker rm -f ethereum-node

docker run -d --name ethereum-node \
           -v /data/geth/data/chain:/root/.ethereum \
           -v /data/geth/data/ethash:/root/.ethash \
           -v /etc/localtime:/etc/localtime \
	   -p 8545:8545 -p 30303:30303 \
           registry.hisun.netwarps.com/ethereum/client-go:v1.10.11 \
           --networkid 65534 \
           --http --http.addr 0.0.0.0 --nat extip:172.17.53.207 --http.api "eth,net,web3,txpool,debug" \
           --syncmode 'full' --allow-insecure-unlock --rpc.allow-unprotected-txs  --gcmode=archive \
           --unlock  ${initAccount} --password /root/.ethereum/password.txt --miner.etherbase ${initAccount} --mine \
           --nodiscover
```

重新启动节点

```
sh run.sh
```

添加第三个签名账号，需要前两个签名账号提名(总签名账号数量/2+1)，删除第三个账号也需要前两个账号提名(总签名账号数量/2+1)

提名删除签名命令参考

```
> acc="0x008deabd65f32efbca5fdf2e8e19a88ef7710ee5"
> clique.propose(acc, false)
```

## 参考

http://blog.hubwiz.com/2019/02/28/ethereum-POA-setup/

https://geth.ethereum.org/docs/interface/private-network