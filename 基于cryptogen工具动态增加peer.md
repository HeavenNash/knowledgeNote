# 基于cryptogen工具动态增加peer节点 #
## 1. 概述 ##
- 基于fabric cacheForRWConflict项目，使用cryptogen工具为新增加的peer节点签发证书，然后配置新增peer节点的yaml文件，启动peer容器，最后将新增的peer节点加入channel

- cacheForRWConflict项目参考http://10.167.35.183:8080/fabric/cacheForRWConflict/tree/master


## 2. 动态增加peer节点 ##
### 2.1. 修改crypto-config.yaml文件，将Template中的count改为3，即增加1个org1的peer节点
```
OrdererOrgs:
  - Name: Orderer
    Domain: example.com
    Specs:
      - Hostname: orderer
PeerOrgs:
  - Name: Org1
    Domain: org1.example.com
    # EnableNodeOUs: true
    Template:
      Count: 3
    Users:
      Count: 1
  - Name: Org2
    Domain: org2.example.com
    # EnableNodeOUs: true
    Template:
      Count: 2
    Users:
      Count: 1
```

### 2.2. 为新增的peer节点通过cryptogen工具生成证书,切换到crypto-config.yaml所在的目录，运行如下命令
```
cryptogen extend --config=./crypto-config.yaml
```
运行成功后，会在crypto-config/peerOrganizations/org1.example.com/peers/ 目录下面生成peer2.org1.example.com的相关证书

### 2.3. 在项目的base目录下面新建add-peer2-org1-base.yaml
```
# Copyright IBM Corp. All Rights Reserved.
#
# SPDX-License-Identifier: Apache-2.0
#
version: '2'

services:

  peer2.org1.example.com:
    container_name: peer2.org1.example.com
    extends:
      file: peer-base.yaml
      service: peer-base
    environment:
      - CORE_PEER_ID=peer2.org1.example.com
      - CORE_PEER_ADDRESS=peer2.org1.example.com:7051
      - CORE_PEER_CHAINCODELISTENADDRESS=peer2.org1.example.com:7052
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer2.org1.example.com:7051
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer0.org1.example.com:7051
      - CORE_PEER_LOCALMSPID=Org1MSP
    volumes:
        - /var/run/:/host/var/run/
        - ../crypto-config/peerOrganizations/org1.example.com/peers/peer2.org1.example.com/msp:/etc/hyperledger/fabric/msp
        - ../crypto-config/peerOrganizations/org1.example.com/peers/peer2.org1.example.com/tls:/etc/hyperledger/fabric/tls

    ports:
      - 11051:7051
      - 11052:7052
      - 11053:7053
```

### 2.4. 在项目的根目录下面新建add-peer2-org1.yaml
```
# Copyright IBM Corp. All Rights Reserved.
#
# SPDX-License-Identifier: Apache-2.0
#
version: '2'

services:

  peer2.org2.example.com:
    container_name: peer2.org1.example.com
    extends:
      file:  base/add-peer2-org1-base.yaml
      service: peer2.org1.example.com
```

### 2.5.  启动新增加的peer2.org1.example.com节点容器
```
docker-compose -f add-peer2-org1.yaml up -d
```

### 2.6. 进入cli容器，并且切换到新增的peer2.org1.example.com节点的环境变量
```
docker exec -it cli bash
```

切换环境变量
```
export CORE_PEER_ADDRESS=peer2.org1.example.com:7051

export CORE_PEER_LOCALMSPID="Org1MSP"

export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer2.org1.example.com/tls/ca.crt

export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
```

### 2.7. 将新增的peer2.org1.example.com节点加入channel，并且检查是否加入成功
```
peer channel join -b mychannel.block
peer channel list
```

### 2.8. 为新增的peer2.org1.example.com节点安装chaincode，并且实例化。实例化省略也可以，在sdk调用chaincode的时候会自动实例化。全部完成后，就可以通过sdk查询新增的peer节点账本了。
```
peer chaincode install -n cacheForRWConflict -v 1.0 -p github.com/hyperledger/fabric/examples/chaincode/   

ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem    

peer chaincode instantiate -o orderer.example.com:7050 --tls true --cafile $ORDERER_CA -C mychannel -n cacheForRWConflict -v 1.0 -c '{"Args":[""]}' -P "OR ('Org1MSP.peer','Org2MSP.peer')"
```