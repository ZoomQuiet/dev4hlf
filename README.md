# dev4hlf
> Hyperledger-Fabric dev. support

## background

Fabric chaincode 开发,太折腾,
每次都得走一系列过程, 才能部署上链再运行, 难以快速调试.

## goal

- 可以快速构建
- 象 Python 开发循环:
  - 修改
  - 运行
  - 观察
- 完成的 golang 合约可以正常部署到标准 Fabric 网络中

## tracking

- 211111 发现资源
- 211207 完成检验, 并反复实验


## usage


整体思路:

- 自行编译最新 fabric
- 自行组建最小 fabric 网络
- 通过 docker 启动网络
- 手工对链码进行 检验/提交/发布
- 然后, 通过调试模式, 略过正式上链流程, 反复快速编译绑定对应端口, 直接发起访问/调试

参考:

奥地利/Klagenfurt roland.bole@samlinux.at 分享姿势:

- [htsc/meetup\-290121 at master · samlinux/htsc](https://github.com/samlinux/htsc/tree/master/meetup-290121)


准备:

完成 HLF 基本环境配置, [Prerequisites — hyperledger\-fabricdocs master documentation](https://hyperledger-fabric.readthedocs.io/en/release-2.3/prereqs.html)

$ go env

    GO111MODULE="on"
    GOARCH="amd64"
    GOBIN=""
    GOCACHE="/home/zoomq/.cache/go-build"
    GOENV="/home/zoomq/.config/go/env"
    GOEXE=""
    GOEXPERIMENT=""
    GOFLAGS=""
    GOHOSTARCH="amd64"
    GOHOSTOS="linux"
    GOINSECURE=""
    GOMODCACHE="/home/zoomq/go/pkg/mod"
    GONOPROXY=""
    GONOSUMDB=""
    GOOS="linux"
    GOPATH="/home/zoomq/go"
    GOPRIVATE=""
    GOPROXY="https://proxy.golang.org,direct"
    GOROOT="/usr/local/go"
    GOSUMDB="sum.golang.org"
    GOTMPDIR=""
    GOTOOLDIR="/usr/local/go/pkg/tool/linux_amd64"
    GOVCS=""
    GOVERSION="go1.17.3"
    GCCGO="gccgo"
    ..


$ docker --version

    Docker version 20.10.11, build dea9396

$ docker-compose --version

    docker-compose version 1.25.0, build unknown

(base) zoomq @ ubuntu-s-1vcpu-2gb-sgp1-01 ~/go/src/github.com
$ mkdir dev4fab & cd dev4fab/



> 编译最新版本 fabric

(base) zoomq @ ubuntu-s-1vcpu-2gb-sgp1-01 ~/go/src/github.com/dev4fab

$ git clone https://github.com/hyperledger/fabric.git

$ cd fabric/


$ cd fabric/

(base) zoomq @ ubuntu-s-1vcpu-2gb-sgp1-01 ~/go/src/github.com/dev4fab/fabric

$ make orderer peer configtxgen

    ...


(base) zoomq @ ubuntu-s-1vcpu-2gb-sgp1-01 ~/go/src/github.com/dev4fab/fabric

$ tree build/ -L 2

    build/
    └── bin
        ├── configtxgen
        ├── orderer
        └── peer

    1 directory, 3 files



$ cd ../

(base) zoomq @ ubuntu-s-1vcpu-2gb-sgp1-01 ~/go/src/github.com/dev4fab


> 将编译好的 fabric 工具追加到环境配置中


$ export PATH=$(pwd)/fabric/build/bin:$PATH


> 检验版本


$ configtxgen -version

    configtxgen:
     Version: 2.4.0
     Commit SHA: 516bc8bab
     Go version: go1.17.3
     OS/Arch: linux/amd64


$ peer version

    peer:
     Version: 2.4.0
     Commit SHA: 516bc8bab
     Go version: go1.17.3
     OS/Arch: linux/amd64
     Chaincode:
      Base Docker Label: org.hyperledger.fabric
      Docker Namespace: hyperledger



(base) zoomq @ ubuntu-s-1vcpu-2gb-sgp1-01 ~/go/src/github.com/dev4fab

$ git clone https://github.com/hyperledger/fabric-samples.git

$ git clone https://github.com/ZoomQuiet/dev4hlf.git

(base) zoomq @ ubuntu-s-1vcpu-2gb-sgp1-01 ~/go/src/github.com/dev4fab


> 当前如下目录

$ tree -L 1 ./

    ./
    ├── dev4hlf
    ├── dev-network
    ├── fabric
    └── fabric-samples

    6 directories, 0 files




(base) zoomq @ ubuntu-s-1vcpu-2gb-sgp1-01 ~/go/src/github.com/dev4fab


> 从官方复制样本配置

$ cp -R fabric/sampleconfig dev-network/

> 从预配样本仓库, 复制入关键配置

$ cp dev4hlf/.env dev-network/

$ cp dev4hlf/docker-compose.yaml dev-network/

$ cp dev4hlf/sampleconfig/configtx.yaml dev-network/sampleconfig/



$ cd dev-network

(base) zoomq @ ubuntu-s-1vcpu-2gb-sgp1-01 ~/go/src/github.com/dev4fab/dev-network


> 创建关键目录


$ mkdir chaincode ledgerData artifacts

> 从官方复制 chaincode 示例

$ cp -R ../fabric-samples/chaincode/sacc chaincode/

(base) zoomq @ ubuntu-s-1vcpu-2gb-sgp1-01 ~/go/src/github.com/dev4fab/dev-network


> 当前目录结构应该如下

$ tree ./ -L 2

    ./
    ├── artifacts
    ├── chaincode
    │   └── sacc
    ├── docker-compose.yaml
    ├── ledgerData
    └── sampleconfig
        ├── configtx.yaml
        ├── core.yaml
        ├── msp
        └── orderer.yaml

    6 directories, 4 files




> 使用 tmux 组织日常调试界面


创建三个窗口:


    +--------------
    |
    |   terminal 0
    |
    +--------------
    |
    |   terminal 1
    |
    +--------------
    |
    |   terminal 2
    |
    +--------------
    

- terminal 0 ~ docker 管理 fabric 网络
- terminal 1 ~ 进行 chaincode 编译部署
- terminal 2 ~ 进行日常请求观察


>> terminal 0

$ export FABRIC_CFG_PATH=$(pwd)/sampleconfig

(base) zoomq @ ubuntu-s-1vcpu-2gb-sgp1-01 ~/go/src/github.com/dev4fab/dev-network


> 生成创世区块
>  generate the genesis block for the ordering service

$ configtxgen -profile SampleDevModeSolo -channelID syschannel -outputBlock genesisblock -configPath $FABRIC_CFG_PATH -outputBlock $(pwd)/artifacts/genesis.block

    2021-12-07 10:32:37.270 UTC 0001 INFO [common.tools.configtxgen] main -> Loading configuration
    2021-12-07 10:32:37.321 UTC 0002 INFO [common.tools.configtxgen.localconfig] completeInitialization -> orderer type: solo
    2021-12-07 10:32:37.321 UTC 0003 INFO [common.tools.configtxgen.localconfig] Load -> Loaded configuration: /home/zoomq/go/src/github.com/dev4fab/dev-network/sampleconfig/configtx.yaml
    2021-12-07 10:32:37.324 UTC 0004 INFO [common.tools.configtxgen] doOutputBlock -> Generating genesis block
    2021-12-07 10:32:37.324 UTC 0005 INFO [common.tools.configtxgen] doOutputBlock -> Creating system channel genesis block
    2021-12-07 10:32:37.324 UTC 0006 INFO [common.tools.configtxgen] doOutputBlock -> Writing genesis block



> 创建通道, 以便交易 create channel creattion transaction


$ configtxgen -channelID ch1 -outputCreateChannelTx $(pwd)/artifacts/ch1.tx -profile SampleSingleMSPChannel -configPath $FABRIC_CFG_PATH


    2021-12-07 10:35:26.729 UTC 0001 INFO [common.tools.configtxgen] main -> Loading configuration
    2021-12-07 10:35:26.760 UTC 0002 INFO [common.tools.configtxgen.localconfig] Load -> Loaded configuration: /home/zoomq/go/src/github.com/dev4fab/dev-network/sampleconfig/configtx.yaml
    2021-12-07 10:35:26.761 UTC 0003 INFO [common.tools.configtxgen] doOutputChannelCreateTx -> Generating new channel configtx
    2021-12-07 10:35:26.762 UTC 0004 INFO [common.tools.configtxgen] doOutputChannelCreateTx -> Writing new channel tx


> 检验成果

$ tree artifacts/

    artifacts/
    ├── ch1.tx
    └── genesis.block

    0 directories, 2 files






>> terminal 1


> 加载配置

$ export FABRIC_CFG_PATH=$(pwd)/sampleconfig

> 加入通道 create the channel ch1

$ peer channel create -o 127.0.0.1:7050 --outputBlock $(pwd)/artifacts/ch1.block -c ch1 -f $(pwd)/artifacts/ch1.tx

    2021-12-07 10:38:18.362 UTC 0001 INFO [channelCmd] InitCmdFactory -> Endorser and orderer connections initialized
    2021-12-07 10:38:18.392 UTC 0002 INFO [cli.common] readBlock -> Received block: 0


> terminal 0 有对应记录:

    orderer.example.com       | 2021-12-07 10:38:18.389 UTC [orderer.commmon.multichannel] newChain -> INFO 013 Created and started new channel ch1
    orderer.example.com       | 2021-12-07 10:38:18.395 UTC [comm.grpc.server] 1 -> INFO 014 streaming call completed grpc.service=orderer.AtomicBroadcast grpc.method=Deliver grpc.peer_address=172.19.0.1:53922 grpc.code=OK grpc.call_duration=31.935078ms


> 检验是否成功创建 # we can fetch the newest block as well

$ peer channel fetch newest $(pwd)/artifacts/ch1.block -c ch1 -o 127.0.0.1:7050

    2021-12-07 10:39:40.373 UTC 0001 INFO [channelCmd] InitCmdFactory -> Endorser and orderer connections initialized
    2021-12-07 10:39:40.375 UTC 0002 INFO [cli.common] readBlock -> Received block: 0

> 加入通道 # join the peer to the channel ch1

$ peer channel join -b $(pwd)/artifacts/ch1.block

    2021-12-07 10:40:25.165 UTC 0001 INFO [channelCmd] InitCmdFactory -> Endorser and orderer connections initialized
    2021-12-07 10:40:25.193 UTC 0002 INFO [channelCmd] executeJoin -> Successfully submitted proposal to join channel


> 编译合约, Build and start the Chaincode


$ cd chaincode/sacc

> 注意, 应该打开 `GO111MODULE=on ` 在 ~/.bashrc


$ go mod vendor 


(base) zoomq @ ubuntu-s-1vcpu-2gb-sgp1-01 ~/go/src/github.com/dev4fab/dev-network/chaincode/sacc

$ tree -L 2 ./

    ./
    ├── go.mod
    ├── go.sum
    ├── sacc.go
    ├── sacc_test.go
    └── vendor
        ├── github.com
        ├── golang.org
        ├── google.golang.org
        └── modules.txt

    4 directories, 5 files

> 编译合约

$ go build -o mycc ./


$ tree -L 2 ./

    ./
    ├── go.mod
    ├── go.sum
    ├── mycc
    ├── sacc.go
    ├── sacc_test.go
    └── vendor
        ├── github.com
        ├── golang.org
        ├── google.golang.org
        └── modules.txt

    4 directories, 6 files

> 直接运行合约 , Start the chaincode

$ CORE_CHAINCODE_LOGLEVEL=debug CORE_PEER_TLS_ENABLED=false CORE_CHAINCODE_ID_NAME=mycc:1.0 ./mycc -peer.address 127.0.0.1:7052


>> terminal 2


> 检验合约, Approve the Chaincode

`approve->check->commit` 是 Fabric 2.X 追加的过程,
在此只需要执行一次, 日常开发就不用反复进行了,
当然, 对应合约版本也不用每次升级;


> 加载配置

$ export FABRIC_CFG_PATH=$(pwd)/sampleconfig



$ peer lifecycle chaincode approveformyorg  -o 127.0.0.1:7050 --channelID ch1 --name mycc --version 1.0 --sequence 1 --init-required --signature-policy "OR ('SampleOrg.member')" --package-id mycc:1.0

    
    2021-12-07 10:47:32.851 UTC 0001 INFO [chaincodeCmd] ClientWait -> txid [348717ac6d26ee9a400a1ea30c200e03550a07f0890fe6043facbb4436cd3ba5] committed with status (VALID) at 0.0.0.0:7051

$ peer lifecycle chaincode checkcommitreadiness -o 127.0.0.1:7050 --channelID ch1 --name mycc --version 1.0 --sequence 1 --init-required --signature-policy "OR ('SampleOrg.member')"
    
    Chaincode definition for chaincode 'mycc', version '1.0', sequence '1' on channel 'ch1' approval status by org:
    SampleOrg: true

$ peer lifecycle chaincode commit -o 127.0.0.1:7050 --channelID ch1 --name mycc --version 1.0 --sequence 1 --init-required --signature-policy "OR ('SampleOrg.member')" --peerAddresses 127.0.0.1:7051


    2021-12-07 10:48:10.671 UTC 0001 INFO [chaincodeCmd] ClientWait -> txid [840d9e43a8b68fa60dc512cf993b8cb0bee6bc054e5e06a86def094647078722] committed with status (VALID) at 127.0.0.1:7051

> 日常测试使用合约 Test the Chaincode

> 初始化, 嘦用一次

$ CORE_PEER_ADDRESS=127.0.0.1:7051 peer chaincode invoke -o 127.0.0.1:7050 -C ch1 -n mycc -c '{"Args":["k1","roland"]}' --isInit
    
    2021-12-07 10:49:30.426 UTC 0001 INFO [chaincodeCmd] chaincodeInvokeOrQuery -> Chaincode invoke successful. result: status:200

对应 docker 日志类似:

    orderer.example.com       | 2021-12-07 10:49:30.428 UTC [comm.grpc.server] 1 -> INFO 01c streaming call completed grpc.service=orderer.AtomicBroadcast grpc.method=Broadcast grpc.peer_address=172.19.0.1:53994 error="rpc error: code = Canceled desc = context canceled" grpc.code=Canceled grpc.call_duration=8.738234ms
    peer0.org1.example.com    | 2021-12-07 10:49:32.434 UTC [gossip.privdata] StoreBlock -> INFO 043 Received block [3] from buffer channel=ch1
    peer0.org1.example.com    | 2021-12-07 10:49:32.435 UTC [committer.txvalidator] Validate -> INFO 044 [ch1] Validated block [3] in 1ms
    peer0.org1.example.com    | 2021-12-07 10:49:32.444 UTC [kvledger] commit -> INFO 045 [ch1] Committed block [3] with 1 transaction(s) in 8ms (state_validation=0ms block_and_pvtdata_commit=2ms state_commit=1ms) commitHash=[dc82ed01f690db15fc7a01ccf8dfd60680065434d94f0f83872fe2152ea62b96]

> 查询

$ CORE_PEER_ADDRESS=127.0.0.1:7051 peer chaincode query -o 127.0.0.1:7050 -C ch1 -n mycc -c '{"Args":["","k1"]}'

    roland

> 交易, Invoke

$ CORE_PEER_ADDRESS=127.0.0.1:7051 peer chaincode invoke -o 127.0.0.1:7050 -C ch1 -n mycc -c '{"Args":["set","k1","Roland2"]}'

    2021-12-07 10:50:51.731 UTC 0001 INFO [chaincodeCmd] chaincodeInvokeOrQuery -> Chaincode invoke successful. result: status:200 payload:"Roland2"

> 查询

$ CORE_PEER_ADDRESS=127.0.0.1:7051 peer chaincode query -o 127.0.0.1:7050 -C ch1 -n mycc -c '{"Args":["","k1"]}'

    Roland2

> 创建

$ CORE_PEER_ADDRESS=127.0.0.1:7051 peer chaincode invoke -o 127.0.0.1:7050 -C ch1 -n mycc -c '{"Args":["set","k2","Snorre"]}'

    2021-12-07 10:51:49.213 UTC 0001 INFO [chaincodeCmd] chaincodeInvokeOrQuery -> Chaincode invoke successful. result: status:200 payload:"Snorre"

> 查询

$ CORE_PEER_ADDRESS=127.0.0.1:7051 peer chaincode query -o 127.0.0.1:7050 -C ch1 -n mycc -c '{"Args":["","k2"]}'

    Snorre



> 帐本数据已经开始生成

(base) zoomq @ ubuntu-s-1vcpu-2gb-sgp1-01 ~/go/src/github.com/dev4fab/dev-network

$ tree -L 1 ledgerData/

    ledgerData/
    ├── chaincodes
    ├── externalbuilder
    ├── ledgersData
    ├── lifecycle
    ├── orderer
    ├── snapshots
    └── transientstore

    7 directories, 0 files



> 追加调试代码


chaincode/sacc/sacc.go 中

追加 os 模块

    ...
    import (
        "fmt"
        "os"
    ...


在合适的地方追加:


    if os.Getenv("DEVMODE_ENABLED") != "" {
        fmt.Printf("Asset key>>> %s\n", args[0])
    }



>> terminal 1

> 重新编译:

$ go build -o mycc ./


> 追加调试环境标记, 再运行:


$ DEVMODE_ENABLED=test  CORE_CHAINCODE_LOGLEVEL=debug CORE_PEER_TLS_ENABLED=false CORE_CHAINCODE_ID_NAME=mycc:1.0 ./mycc -peer.address 127.0.0.1:7052



>> terminal 2


> 查询

$ CORE_PEER_ADDRESS=127.0.0.1:7051 peer chaincode query -o 127.0.0.1:7050 -C ch1 -n mycc -c '{"Args":["","k1"]}'

    Roland2

> 交易, Invoke

$ CORE_PEER_ADDRESS=127.0.0.1:7051 peer chaincode invoke -o 127.0.0.1:7050 -C ch1 -n mycc -c '{"Args":["set","k1","Sayeahooo..."]}'

    2021-12-07 11:03:11.803 UTC 0001 INFO [chaincodeCmd] chaincodeInvokeOrQuery -> Chaincode invoke successful. result: status:200 payload:"Sayeahooo..."


> 查询

$ CORE_PEER_ADDRESS=127.0.0.1:7051 peer chaincode query -o 127.0.0.1:7050 -C ch1 -n mycc -c '{"Args":["","k1"]}'

    Sayeahooo...



>> 对应 terminal 1, 将出现调试输出

$ DEVMODE_ENABLED=test  CORE_CHAINCODE_LOGLEVEL=debug CORE_PEER_TLS_ENABLED=false CORE_CHAINCODE_ID_NAME=mycc:1.0 ./mycc -peer.address 127.0.0.1:7052
Asset key>>> k1
Asset key>>> k1




以上;





## refer.

奥地利/Klagenfurt roland.bole@samlinux.at 分享姿势:

- [htsc/meetup\-290121 at master · samlinux/htsc](https://github.com/samlinux/htsc/tree/master/meetup-290121)


## logging

- 211207 ZQ init.
