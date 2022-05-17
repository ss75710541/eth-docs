# Geth备份与恢复

## 导出

### 进入需要备份的Geth容器

```
kubectl exec -it uat-node2-geth-0 -n geth
```

### 执行导出备份

```
geth attach ~/.ethereum/geth.ipc

> block=eth.blockNumber
> admin.exportChain("/root/.ethereum/node-"+block,0,block)

cd ~/.ethereum/
tar czvf node-728218.tar.gz node-728218
```

## 导入

### 先设置geth config 静态 nodes 为空

```
  [Node.P2P]
  StaticNodes = []
```

### 正常启动 node 



### 修改 node 初始化命令

```sh
kubectl edit sts uat-export-test-node-geth -n geth
```

```yaml
      ...
      initContainers:
      - args:
        - "1000"
        command:
        - sleep
      ...
```

### 复制geth 备份文件到需要导入的 geth 初始化容器中

略

### 进入 geth 初始化容器，执行导入

```sh
kubectl exec -it uat-export-test-node-geth -n geth -c geth-init
geth import node-728218
```

最终出现类似下面内容

```sh

INFO [05-17|07:10:15.171] Imported new chain segment               blocks=2500  txs=0   mgas=0.000   elapsed=318.026ms mgasps=0.000   number=757,500 hash=661707..adc023 age=5h45s     dirty=2.86MiB
INFO [05-17|07:10:15.320] Imported new chain segment               blocks=1087  txs=3   mgas=0.341   elapsed=143.188ms mgasps=2.382   number=758,587 hash=f45263..d34761 age=4h24m31s  dirty=2.86MiB
INFO [05-17|07:10:15.323] Writing cached state to disk             block=758,587 hash=f45263..d34761 root=78cbd6..5851ce
INFO [05-17|07:10:15.348] Persisted trie from memory database      nodes=14450 size=1.54MiB time=25.198205ms gcnodes=39889 gcsize=9.71MiB gctime=205.017876ms livenodes=1 livesize=-5986.00B
INFO [05-17|07:10:15.348] Writing cached state to disk             block=758,586 hash=fee116..bd2d68 root=78cbd6..5851ce
INFO [05-17|07:10:15.348] Persisted trie from memory database      nodes=0     size=0.00B   time="1.563µs"   gcnodes=0     gcsize=0.00B   gctime=0s           livenodes=1 livesize=-5986.00B
INFO [05-17|07:10:15.348] Writing cached state to disk             block=758,460 hash=3cc9b8..6a154e root=78cbd6..5851ce
INFO [05-17|07:10:15.348] Persisted trie from memory database      nodes=0     size=0.00B   time=851ns       gcnodes=0     gcsize=0.00B   gctime=0s           livenodes=1 livesize=-5986.00B
INFO [05-17|07:10:15.348] Writing snapshot state to disk           root=f955ac..2aacdb
INFO [05-17|07:10:15.348] Persisted trie from memory database      nodes=0     size=0.00B   time="1.012µs"   gcnodes=0     gcsize=0.00B   gctime=0s           livenodes=1 livesize=-5986.00B
INFO [05-17|07:10:15.349] Blockchain stopped
Import done in 1m56.839947474s.

Compactions
 Level |   Tables   |    Size(MB)   |    Time(sec)  |    Read(MB)   |   Write(MB)
-------+------------+---------------+---------------+---------------+---------------
   0   |          0 |       0.00000 |       6.38418 |       0.00000 |     363.69713
   1   |         47 |      98.74317 |       5.21311 |     447.94436 |     185.06102
   2   |          1 |       2.07274 |       0.00000 |       0.00000 |       0.00000
-------+------------+---------------+---------------+---------------+---------------
 Total |         48 |     100.81592 |      11.59729 |     447.94436 |     548.75815

Read(MB):523.19339 Write(MB):1389.43178
Object memory: 694.981 MB current, 1118.980 MB peak
System memory: 1454.668 MB current, 1454.668 MB peak
Allocations:   391.868 million
GC pause:      13.671928ms

Compacting entire database...
Compaction done in 4.04202811s.

Compactions
 Level |   Tables   |    Size(MB)   |    Time(sec)  |    Read(MB)   |   Write(MB)
-------+------------+---------------+---------------+---------------+---------------
   0   |          0 |       0.00000 |       6.94131 |       0.00000 |     404.78069
   1   |          0 |       0.00000 |       6.92839 |     587.77110 |     314.58942
   2   |         63 |     131.59918 |       1.74205 |     131.60114 |     131.59918
-------+------------+---------------+---------------+---------------+---------------
 Total |         63 |     131.59918 |      15.61176 |     719.37224 |     850.96930

Read(MB):787.80635 Write(MB):1691.65939
```

### 修改新 geth 节点静态 node 配置

```yaml
  [Node.P2P]
  StaticNodes = ["enode://d39248f3c5e6b1b4a804b428fd89ab941f7bfb2351828cd38da1c84d8e1a449dd489d493a8e6cbf95a9c9a5cd688bf019024afab8836262cb7815e9f334a656d@x.x.x.x:30303?discport=0","enode://ab367d339ae1b8c7ecfec1bfed951bb0f90589d829694e133f33604f72257c6ac28e36f61145d78e12ae8841a3b91d69693d0daf51b126735ce1ed32543e8f19@x.x.x.x:30303?discport=0","enode://1a6a003ec8abc962093e98e27749340c4fe80685842fd3e38d578ab9230c58e786cc4b532c5fc823faa4e8122c33a37f8dd9fb1596d823e56b88734cff0d1ca4@x.x.x.x:30303?discport=0"]
```

### 更新 geth node

```
sh install-uat-export-test-node-geth.sh
```

 删除 sleep 的 geth node(因为init 容器 sleep 了，等自动替换 pod 需要很久)

```
kubectl delete pod uat-export-test-node-geth-0 -n geth
```

### 查看 geth node 日志

```
kubectl logs uat-export-test-node-geth-0 -n geth
```

出现下面类似日志为正常

```
INFO [05-17|07:19:36.924] Starting Geth on Ethereum mainnet...
INFO [05-17|07:19:36.924] Bumping default cache on mainnet         provided=1024 updated=4096
INFO [05-17|07:19:36.924] Enabling metrics collection
INFO [05-17|07:19:36.924] Enabling metrics export to InfluxDB
INFO [05-17|07:19:36.924] Enabling stand-alone metrics HTTP endpoint address=0.0.0.0:6060
INFO [05-17|07:19:36.924] Starting metrics server                  addr=http://0.0.0.0:6060/debug/metrics
INFO [05-17|07:19:36.927] Maximum peer count                       ETH=50 LES=0 total=50
INFO [05-17|07:19:36.927] Smartcard socket not found, disabling    err="stat /run/pcscd/pcscd.comm: no such file or directory"
WARN [05-17|07:19:36.928] Disable transaction unindexing for archive node
WARN [05-17|07:19:36.928] Sanitizing cache to Go's GC limits       provided=4096 updated=2498
INFO [05-17|07:19:36.928] Enabling recording of key preimages since archive mode is used
INFO [05-17|07:19:36.928] Set global gas cap                       cap=50,000,000
INFO [05-17|07:19:36.928] Allocated trie memory caches             clean=748.00MiB dirty=0.00B
INFO [05-17|07:19:36.928] Allocated cache and file handles         database=/root/.ethereum/geth/chaindata cache=1.22GiB handles=524,288
INFO [05-17|07:19:36.945] Opened ancient database                  database=/root/.ethereum/geth/chaindata/ancient readonly=false
INFO [05-17|07:19:36.946] Initialised chain configuration          config="{ChainID: 65535 Homestead: 0 DAO: <nil> DAOSupport: false EIP150: 0 EIP155: 0 EIP158: 0 Byzantium: 0 Constantinople: 0 Petersburg: 0 Istanbul: 0, Muir Glacier: <nil>, Berlin: <nil>, London: <nil>, Arrow Glacier: <nil>, MergeFork: <nil>, Terminal TD: <nil>, Engine: clique}"
INFO [05-17|07:19:36.946] Initialising Ethereum protocol           network=65535 dbversion=8
INFO [05-17|07:19:37.349] Loaded most recent local header          number=758,587 hash=f45263..d34761 td=1,516,737 age=4h33m53s
INFO [05-17|07:19:37.349] Loaded most recent local full block      number=758,587 hash=f45263..d34761 td=1,516,737 age=4h33m53s
INFO [05-17|07:19:37.349] Loaded most recent local fast block      number=758,587 hash=f45263..d34761 td=1,516,737 age=4h33m53s
INFO [05-17|07:19:37.431] Upgrading chain index                    type=bloombits percentage=0
INFO [05-17|07:19:37.431] Loaded local transaction journal         transactions=0 dropped=0
INFO [05-17|07:19:37.432] Regenerated local transaction journal    transactions=0 accounts=0
INFO [05-17|07:19:37.432] Gasprice oracle is ignoring threshold set threshold=2
INFO [05-17|07:19:37.436] Starting peer-to-peer node               instance=Geth/v1.10.17-stable-25c9b49f/linux-amd64/go1.18
INFO [05-17|07:19:37.443] New local node record                    seq=1,652,770,555,925 id=9a71a5e55d6bdff5 ip=127.0.0.1 udp=0 tcp=30303
INFO [05-17|07:19:37.443] Started P2P networking                   self="enode://7b2d68b9b38bd15dd20e9ad8f7cc3070ec3eca8fc329985569748a5b38ce34e5b73960fb63f3d732e1d2bd6fb27292fd250c64be6e4456ae33eef5d55a1e2e3b@127.0.0.1:30303?discport=0"
INFO [05-17|07:19:37.444] IPC endpoint opened                      url=/root/.ethereum/geth.ipc
INFO [05-17|07:19:37.445] HTTP server started                      endpoint=[::]:8545 auth=false prefix= cors=* vhosts=*
INFO [05-17|07:19:37.445] WebSocket enabled                        url=ws://[::]:8546
INFO [05-17|07:19:37.711] Deep froze chain segment                 blocks=30001 elapsed=765.078ms number=458,870 hash=65c394..a4b02e
INFO [05-17|07:19:38.536] Deep froze chain segment                 blocks=30001 elapsed=825.025ms number=488,871 hash=ea197a..9413c8
INFO [05-17|07:19:39.367] Deep froze chain segment                 blocks=30001 elapsed=831.329ms number=518,872 hash=71652e..155203
INFO [05-17|07:19:40.200] Deep froze chain segment                 blocks=30001 elapsed=832.905ms number=548,873 hash=2a2409..f49424
INFO [05-17|07:19:41.028] Deep froze chain segment                 blocks=30001 elapsed=827.629ms number=578,874 hash=faa50f..0c09eb
INFO [05-17|07:19:41.823] Deep froze chain segment                 blocks=30001 elapsed=794.479ms number=608,875 hash=480370..4fcff2
INFO [05-17|07:19:42.665] Deep froze chain segment                 blocks=30001 elapsed=842.073ms number=638,876 hash=6c9cc4..b639de
INFO [05-17|07:19:43.538] Deep froze chain segment                 blocks=29711 elapsed=872.657ms number=668,587 hash=bcbc78..0794ee
INFO [05-17|07:19:45.494] Upgrading chain index                    type=bloombits percentage=17
INFO [05-17|07:19:47.446] Block synchronisation started
INFO [05-17|07:19:48.373] Downloader queue stats                   receiptTasks=0 blockTasks=168 itemSize=593.30B throttle=8192
INFO [05-17|07:19:48.377] Imported new chain segment               blocks=23    txs=0 mgas=0.000 elapsed=3.382ms   mgasps=0.000 number=758,610 hash=af7f3b..56f0e6 age=4h33m18s dirty=0.00B
INFO [05-17|07:19:48.414] Imported new chain segment               blocks=10    txs=1 mgas=0.207 elapsed=2.904ms   mgasps=71.356 number=758,620 hash=6fae76..6d81fd age=4h32m58s dirty=0.00B
INFO [05-17|07:19:48.442] Imported new chain segment               blocks=184   txs=1 mgas=0.056 elapsed=24.118ms  mgasps=2.316  number=758,804 hash=4e0231..07047b age=4h26m50s dirty=0.00B
INFO [05-17|07:19:48.447] Imported new chain segment               blocks=10    txs=1 mgas=0.207 elapsed=3.117ms   mgasps=66.497 number=758,814 hash=ff70e7..453bda age=4h26m30s dirty=0.00B
INFO [05-17|07:19:48.572] Imported new chain segment               blocks=733   txs=1 mgas=0.056 elapsed=121.975ms mgasps=0.458  number=759,547 hash=d7c397..c9a957 age=4h2m4s   dirty=0.00B
INFO [05-17|07:19:48.831] Imported new chain segment               blocks=1728  txs=1 mgas=0.021 elapsed=253.802ms mgasps=0.083  number=761,275 hash=a1326e..c6daee age=3h4m28s  dirty=0.00B
INFO [05-17|07:19:49.144] Imported new chain segment               blocks=2048  txs=0 mgas=0.000 elapsed=305.898ms mgasps=0.000  number=763,323 hash=b83968..55c6c3 age=1h56m13s dirty=0.00B
INFO [05-17|07:19:49.490] Imported new chain segment               blocks=2048  txs=0 mgas=0.000 elapsed=338.930ms mgasps=0.000  number=765,371 hash=96c2a8..a9d7cc age=47m57s   dirty=0.00B
INFO [05-17|07:19:49.720] Imported new chain segment               blocks=1436  txs=1 mgas=0.223 elapsed=221.790ms mgasps=1.007  number=766,807 hash=06d044..6ce8da dirty=0.00B
INFO [05-17|07:19:52.182] Imported new chain segment               blocks=4     txs=0 mgas=0.000 elapsed="915.403µs" mgasps=0.000  number=766,811 hash=00ee7d..cc13ba dirty=0.00B
INFO [05-17|07:19:53.506] Upgrading chain index                    type=bloombits percentage=69
INFO [05-17|07:19:54.559] Imported new chain segment               blocks=1     txs=0 mgas=0.000 elapsed=3.388ms     mgasps=0.000  number=766,812 hash=f1849c..8798ef dirty=0.00B
INFO [05-17|07:19:56.577] Imported new chain segment               blocks=1     txs=0 mgas=0.000 elapsed="265.778µs" mgasps=0.000  number=766,813 hash=e24a66..019772 dirty=0.00B
INFO [05-17|07:19:56.732] Finished upgrading chain index           type=bloombits
INFO [05-17|07:19:58.035] Imported new chain segment               blocks=1     txs=0 mgas=0.000 elapsed="216.515µs" mgasps=0.000  number=766,814 hash=5301f0..7749ac dirty=0.00B
```

