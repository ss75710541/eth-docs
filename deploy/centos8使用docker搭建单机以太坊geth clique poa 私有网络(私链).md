# centos8使用 docker 搭建单机以太坊geth clique poa 私有网络(私链)

## 安装 docker

参考：[centos8使用docker搭建go-ethereum-1.10.11以太坊pow私有网络(私链).md](./centos8使用docker搭建go-ethereum-1.10.11以太坊pow私有网络(私链).md)

## 初始化创世信息

创建 init.sh

```
#!/bin/bash
set -e
cd $(dirname $0)
SCRIPTS_DIR=$(pwd)

chainId=65534
# poa 网络没有挖矿奖励，所以需要提交指定账号预置大量 eth
balance=0x200000000000000000000000000000000000000000000000000000000000000


clean(){
rm -f ethereum-node
rm -rf /data/geth/data
rm -f UTC*
}



initGenesis(){
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
      "period": 15,
      "epoch": 30000
    }
  },
  "extradata":"0x0000000000000000000000000000000000000000000000000000000000000000${initAccount}0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
  "gasLimit": "1",
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

    docker run -it --rm \
           -v ${SCRIPTS_DIR}/genesis.json:/root/files/genesis.json \
           -v /data/geth/data/chain:/root/.ethereum \
           -v /data/geth/data/ethash:/root/.ethash \
           ethereum/client-go:v1.10.11 \
           init /root/files/genesis.json
    cp UTC* /data/geth/data/chain/keystore/
    cp password.txt /data/geth/data/chain/
}

createAccount(){

    password=`< /dev/urandom tr -dc _A-Z-a-z-0-9 | head -c${1:-12};echo;`
    echo -n "$password" > password.txt
    mkdir -p /data/geth/data/chain/
    cp password.txt /data/geth/data/chain/
docker run -it --rm \
           -v /data/geth/data/chain:/root/.ethereum \
           -v /data/geth/data/ethash:/root/.ethash \
           -v /etc/localtime:/etc/localtime \
           -v /data/geth/data/chain/geth.ipc:/data/geth/data/chain/geth.ipc \
           ethereum/client-go:v1.10.11 account new --password /root/.ethereum/password.txt
    cp /data/geth/data/chain/keystore/* .

}

# 清理现有数据
clean
# 创建账号
createAccount
# 初始化
initGenesis
```

执行初始化

```
sh init.sh
```

## 运行 geth

创建 run.sh

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
           -p 8545:8545 -p 30303:30303 -p 30303:30303/udp \
           registry.hisun.netwarps.com/ethereum/client-go:v1.10.11 \
           --networkid 65534 --http --http.addr 0.0.0.0 --nat extip:0.0.0.0 --http.api "eth,net,web3,personal,miner,txpool,debug,clique" --allow-insecure-unlock --rpc.allow-unprotected-txs  --gcmode=archive --mine --unlock  ${initAccount} --password /root/.ethereum/password.txt --miner.etherbase ${initAccount}
```

## 测试交易

```
docker run -it --rm \
           ethereum/client-go:v1.10.11 \
           attach http://172.17.16.174:8545

> acc0=eth.accounts[0]
"0x2292e02dcc558f68e6029dd5cb0f95055c2bd4aa"
> acc1="0xD1F4c75484f9390D3b05BC993FE795F95f9F57e4"
"0x0000000000000000000000000000000000000000"
> amount = web3.toWei(0.01)
"10000000000000000"
> eth.sendTransaction({from: acc0, to: acc1, value: amount})
> eth.sendTransaction({from: acc0, to: acc1, value: amount, gas: 300000, gasPrice: 300000})
"0x2585bd04573256138fac59c8c90f2073ec84d807b5c64e589cc52bf132b3c985"
```
  
## 参考：
  
  https://geth.ethereum.org/docs/interface/private-network
  
  http://blog.hubwiz.com/2019/02/28/ethereum-POA-setup/
  
  https://www.daimajiaoliu.com/daima/4ed53f9eb10040c
  
  https://github.com/ethereum/go-ethereum/blob/master/consensus/clique/clique.go