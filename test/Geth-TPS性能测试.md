# Geth-TPS性能测试

## 环境依赖

### Geth 环境配置

| 版本     | Geth RPC Address          | ChainID | NetworkID | consensus(共识算法) | CPU  | 内存 | 磁盘(rssd) |
| -------- | ------------------------- | ------- | --------- | ------------------- | ---- | ---- | ---------- |
| v1.10.17 | http://172.17.57.222:8545 | 65533   | 65533     | clique              | 4C   | 8G   | 100G       |

### Geth 启动命令

```yaml
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
    - 631cb5db2476e55e5886e0a7d1cdc02701c4a581
    - --password
    - /root/.ethereum/password
    - --mine
    - --nodiscover
    - --metrics
    - --metrics.addr=0.0.0.0
    - --miner.gaslimit=30000000
```

### Geth 启动配置

```yaml
config: |
  [Eth]
  NetworkId = 65533
  SyncMode = "full"

  [Node]
  HTTPHost = "0.0.0.0"
  HTTPModules = ["eth", "net", "web3", "personal", "txpool"]

  [Metrics]
  HTTP = "0.0.0.0"

  [Node.P2P]
  StaticNodes = []
  TrustedNodes = []
```

### Geth 初始信息

```
genesis: '{"config":{"chainId": 65533,"homesteadBlock":0,"eip150Block":0,"eip150Hash":"0x0000000000000000000000000000000000000000000000000000000000000000","eip155Block":0,"eip158Block":0,"byzantiumBlock":0,"constantinopleBlock":0,"petersburgBlock":0,"istanbulBlock":0,"clique":{"period":2,"epoch":30000}},"extradata":"0x0000000000000000000000000000000000000000000000000000000000000000631cb5db2476e55e5886e0a7d1cdc02701c4a5810000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000","gasLimit":"30000000","coinbase":"0x0000000000000000000000000000000000000000","difficulty":"0x1","mixHash":"0x0000000000000000000000000000000000000000000000000000000000000000","alloc":{"0000000000000000000000000000000000000000":{"balance":"0x1"},"631cb5db2476e55e5886e0a7d1cdc02701c4a581":{"balance":"0x200000000000000000000000000000000000000000000000000000000000000"}},"nonce":"0x0000000000000123","number":"0x0","gasUsed":"0x0","parentHash":"0x0000000000000000000000000000000000000000000000000000000000000000","baseFeePerGas":null}'
key: '{"address":"631cb5db2476e55e5886e0a7d1cdc02701c4a581","crypto":{"cipher":"aes-128-ctr","ciphertext":"67b2fe8cf0acb5b8221883a1219e5cc46b9a5345bddad481a1775f4bdc2d37f6","cipherparams":{"iv":"f351d571f83ff30c25e3ded259efaa67"},"kdf":"scrypt","kdfparams":{"dklen":32,"n":262144,"p":1,"r":8,"salt":"c99e8936a62de9719501d9e3f38c08a896820d5dd9c7689195fe84a079b42250"},"mac":"9ccf8654d97369bd9acb7bec5eb72f1e56e84d781aa86501b9407270e3b1eeeb"},"id":"18a30473-c74e-4f6b-a2e2-d11949c70d03","version":3}'
```

### 压测客户端主机配置

| 操作系统     | IP            | CPU  | 内存 | 磁盘(rssd) |
| ------------ | ------------- | ---- | ---- | ---------- |
| ubuntu 20.04 | 172.17.242.80 | 8C   | 16G  | 20G        |

## 安装chainhammer

### 安装前准备

```sh
cd ~/
git clone https://github.com/drandreaskrueger/chainhammer
cd chainhammer
```

编辑 hammer/clienttype.py 91-94 行内容如下

```python
    consensus = "clique"
    chainName = "poa"
    networkId = 65533
    chainId = 65533
```

编辑 hammer/config.py 17-18 行内容如下

```
RPCaddress='http://172.17.57.222:8545'
RPCaddress2='http://172.17.57.222:8545'
```

编辑 624 行，注释 断言 (可能是磁盘原因，目前没有深究，暂时先注释，否则会报错) 

```python
624     #assert check_timestamp_format(df)
```

编辑 script/install.sh , 注释 安装 go、 geth 、initialize 内容

```
# go
#install_chapter scripts/install-go.sh

# geth
#install_chapter scripts/install-geth.sh

# deploy.py andtests on testRPC
install_chapter scripts/install-initialize.sh
```

### 执行安装

```
./script/install.sh
```

### 执行测试

```sh
# 顺序执行10000个交易
export CH_TXS=10000 CH_THREADING=sequential
./run.sh geth-tps
# 并发20线程执行10000个交易
export CH_TXS=10000 CH_THREADING="threaded2 20"
./run.sh geth-tps
```

默认为前台进程，交易数少可以直接执行，日志会输出到 logs 中，send.py.log 为发送的交易日志，tps.py.log 为统计的 tps 日志

```sh
ll logs/
total 212
drwxr-xr-x  2 root root   4096 Apr  5 16:58 ./
drwxr-xr-x 13 root root   4096 Apr  6 19:00 ../
-rw-r--r--  1 root root    843 Apr  6 11:43 deploy.py.log
-rw-r--r--  1 root root    911 Mar 31 16:11 network.log
-rw-r--r--  1 root root 185455 Apr  6 11:45 send.py.log
-rw-r--r--  1 root root  11283 Apr  6 11:45 tps.py.log
```

如果交易数量很大，前台有可能会中断，所以使用screen 工具来执行

```sh
screen
# 顺序执行20000000个交易
export CH_TXS=20000000 CH_THREADING=sequential
./run.sh geth-tps
```

执行的完成后结果保存到 `results/runs/`目录中

```sh
ll results/runs/Geth_20220406*
-rw-r--r-- 1 root root 5889 Apr  6 11:42 results/runs/Geth_20220406-1141_txs10000.html
-rw-r--r-- 1 root root 5724 Apr  6 11:42 results/runs/Geth_20220406-1141_txs10000.md
```

启动一个简单python web 即可访问 html 结果

```sh
python -m SimpleHTTPServer 8080
```

访问 `http://172.17.242.80:8080/results/runs/Geth_20220406-1141_txs10000.html` 即可查看测试结果 

### 影响测试结果的主要参数

影响Geth TPS 的主要有两个参数

period (出块时间)

gasLimit (一个块的 gas 上限)

可以调整这两个参数进行测试对比结果

## 参考：

https://github.com/drandreaskrueger/chainhammer/blob/master/results/geth.md

https://github.com/drandreaskrueger/chainhammer/blob/master/docs/FAQ.md

https://blog.amis.com/geth-not-broadcasting-transactions-1a881c50dafc