# Centos8使用docker部署区块链浏览器blocksount

## 下载 blockscout 代码

```shell
yum install -y git
git clone https://github.com/blockscout/blockscout.git
```

## 修改 单机启动 Makefile

注意：docker Makefile 方式目前为单机测试环境使用，官方文档中不建议生产使用

```
cd blockscout/docker
```

修改 docker 目录下 Makefile文件中 下面相关内容

```
...
# 已经制作好的镜像，脚本本身可以直接编译制作镜像，但因为网络原因可能会制作失败，所以使用了我提前制作好的镜像
DOCKER_IMAGE = quay.io/ss75710541/blockscout
...
PG_CONTAINER_IMAGE = docker.mirrors.ustc.edu.cn/library/postgres:14.0-alpine
...
 start: build postgres
        @echo "==> Starting blockscout"
        @docker run -d --name $(BS_CONTAINER_NAME) \
...
```

## 启动 blockscout

下载依赖镜像, 如果不拉取镜像会触发编译镜像流程

```
docker pull quay.io/ss75710541/blockscout
docker pull docker.mirrors.ustc.edu.cn/library/postgres:14.0-alpine
```

启动 blockscout

```
docker rm -f blockscout
export NETWORK=Ethereum
export SUBNETWORK="Ethereum Classic"
export LOGO=/images/ethereum.png
export LOGO_FOOTER=/images/ethereum.png
export ETHEREUM_JSONRPC_VARIANT=geth
export ETHEREUM_JSONRPC_HTTP_URL=http://172.17.103.114:8545
export COIN=ETH
export DATABASE_URL=postgresql://root:123456@172.17.103.114:5432/blockscout
make start
```

## 查看日志

```
docker logs blockscout
```

## 参考：

https://docs.blockscout.com/for-developers/information-and-settings/env-variables

https://github.com/blockscout/blockscout/tree/master/apps/block_scout_web/assets/static/images