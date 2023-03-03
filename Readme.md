# 软件版本
Fabric 2.2
fabric-samples 3d2875c180c67d9f3b3a166bb15b9c73fb6c7091
fabric-samples 15275a0d4d46696c22e652a0674e36c570431a33
# 准备操作
``` bash
cd ~
git clone https://github.com/hyperledger/fabric-samples.git
git checkout 3d2875c180c67d9f3b3a166bb15b9c73fb6c7091
```
# 手动搭建hyperledger fabric v2.x 网络（一）
演示证书模板生成
``` bash
cryptogen showtemplate > crypto-config.yaml
```
修改`crypto-config.yaml`文件中3处有关`EnableNodeOUs`的部分为`true`。
``` bash
cryptogen generate --config=crypto-config.yaml
```
这期课程生成组织文件，会生成`crypto-config`目录。

# 手动搭建hyperledger fabric v2.x 网络（二）
创建通道所需的配置，需要选择`Sep 16, 2020`的提交
``` bash
wget https://raw.githubusercontent.com/hyperledger/fabric-samples/3d2875c180c67d9f3b3a166bb15b9c73fb6c7091/test-network/configtx/configtx.yaml
```
更改msp路径为项目具体的路径。
搜索所有包含`../organizations`文本的路径，替换为`crypto-config`当前路径。
修改`Profiles`中的部分，按照[官方文档](https://hyperledger-fabric.readthedocs.io/zh_CN/latest/create_channel/create_channel_config.html#profiles)中的内容来。

使用官方工具，生成通道配置。
``` bash
# 生成创世块文件
configtxgen -profile TwoOrgsOrdererGenesis -outputBlock ./channel-artifacts/genesis.block -channelID fabric-channel
# 生成通道文件
configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID mychannel
# 生成组织1锚节点文件
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/OrglMSPanchors.tx -channelID mychannel -asOrg Org1MSP
# 生成组织2锚节点文件
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org2MSPanchors.tx -channelID mychannel -asOrg Org2MSP
```
在`channel-artifacts`中会生成4个文件。

# 手动搭建hyperledger fabric v2.x 网络（三）
创建peer与排序节点
```
wget https://raw.githubusercontent.com/hyperledger/fabric-samples/15275a0d4d46696c22e652a0674e36c570431a33/test-network/docker/docker-compose-test-net.yaml -O docker-compose.yaml
```
修改`networks.test.name`为：`twonodes_test`。
修改`volumes`中对应的宿主机路径。
修改组织1与组织2的网络名称和挂载路径
修改`peer0.org2.example.com`环境变量中的`CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE`为`twonodes_test`。
把cli拆成cli1和cli2，分别表示两个组织的cli，环境变量部分，参考测试网络中的内容进行配置。
热心网友提供的[`docker-compose.yml`文件](https://www.godzhang.top/archives/dockcer-composeyaml-wen-jian)
<!-- 存疑 -->

# 手动搭建hyperledger fabric v2.x 网络（四）完结
第三期中的docker-compose中还有一处需要修改，有关节点启动方式，详看视频。或者直接拿热心网友的文件过来用。
停止集群并清理文件
```
docker-compose down
docker volume prune
```
启动集群
```
docker-compose up -d
docker ps
```
## 加入通道
进入到终端节点
使用cli1创建一个通道`mychannel.block`,并加入到该通道。
``` bash
docker exec -it cli1 bash
peer channel create -o orderer.example.com:7050 -c mychannel -f ./channel-artifacts/channel.tx --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/msp/tlscacerts/tlsca.example.com-cert.pem
ls mychannel.block
peer channel join -b mychannel.block
exit
```
将创建好的通道复制到本地宿主机，将其传入cli2中，并加入到该通道。
``` bash
docker cp cli1:/opt/gopath/src/github.com/hyperledger/fabric/peer/mychannel.block ./
docker cp ./mychannel.block cli2:/opt/gopath/src/github.com/hyperledger/fabric/peer/mychannel.block
docker exec -it cli2 bash
ls mychannel.block
peer channel join -b mychannel.block
exit
```
更新锚节点
``` bash
docker exec -it cli1 bash
peer channel update -o orderer.example.com:7050 -c mychannel -f ./channel-artifacts/OrglMSPanchors.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
exit
docker exec -it cli2 bash
peer channel update -o orderer.example.com:7050 -c mychannel -f ./channel-artifacts/Org2MSPanchors.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
exit
```
关于链码生命周期的操作
``` bash
# 下载链码
wget https://raw.githubusercontent.com/hyperledger/fabric-samples/3d2875c180c67d9f3b3a166bb15b9c73fb6c7091/chaincode/sacc/sacc.go -O chaincode/go/sacc.go
# 编译链码
docker exec -it cli1 bash
cd /opt/gopath/src/github.com/hyperledger/fabric/examples/chaincode/go
go mod init
go mod tidy
go mod vendor
# 打包链码并分发链码
cd /opt/gopath/src/github.com/hyperledger/fabric/peer
peer lifecycle chaincode package sacc.tar.gz --path /opt/gopath/src/github.com/hyperledger/fabric/examples/chaincode/go/ --label sacc_1
ls sacc.tar.gz
exit
docker cp cli1:/opt/gopath/src/github.com/hyperledger/fabric/peer/sacc.tar.gz ./
docker cp ./sacc.tar.gz cli2:/opt/gopath/src/github.com/hyperledger/fabric/peer/sacc.tar.gz
```
开启两个终端，分别进入cli1和cli2，执行以下命令。
``` bash
# 安装链码
peer lifecycle chaincode install sacc.tar.gz
```
组织批准链码
``` bash
# 查询已安装链码id
peer lifecycle chaincode queryinstalled
# 组织批准链码
peer lifecycle chaincode approveformyorg --channelID mychannel --name sacc --version 1.0 --init-required --package-id sacc_1:5970aafc8e84ea61640bc2d52edc760e691b18d51509a3eb9af12bd8de6de418 --sequence 1 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
# 检查是否已经批准
peer lifecycle chaincode checkcommitreadiness --channelID mychannel --name sacc --version 1.0 --init-required --sequence 1 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --output json
# commit链码
peer lifecycle chaincode commit -o orderer.example.com:7050 --channelID mychannel --name sacc --version 1.0 --sequence 1 --init-required --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:9051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
```
链码的整个生命周期完成
链码调用
``` bash
peer chaincode invoke -o orderer.example.com:7050 --isInit --ordererTLSHostnameOverride orderer.example.com --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n sacc --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:9051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt -c '{"Args":["a","bb"]}'
```
链码查询
``` bash
peer chaincode query -C mychannel -n sacc -c '{"Args":["query","a"]}'
```
链码测试键值对["a","cc"]
``` bash
peer chaincode invoke -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n sacc --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:9051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt -c '{"Args":["set","a","cc"]}'
```