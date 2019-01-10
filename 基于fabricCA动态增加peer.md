# 基于fabric CA动态增加peer节点 #
## 1. 概述 ##
- 基于fabric cacheForRWConflict项目，使用fabric CA为新增加的peer节点签发证书，然后配置新增peer节点的yaml文件，启动peer容器，最后将新增的peer节点加入channel。实验中，我们配置了两个CA节点，一个是管理org1 peer身份的CA节点，另一个是管理peer tls通信的CA节点。

- cacheForRWConflict项目参考http://10.167.35.183:8080/fabric/cacheForRWConflict/tree/master


## 2. 动态增加peer节点 ## 
### 2.1. 配置docker-compose-cli.yaml，为org1新增ca0节点和tlsca_org1节点。其中，ca0管理org1的身份证书， tlsca_org1管理org1中的tls通信证书

```
# 加入org1 ca节点

  ca0: 
    image: hyperledger/fabric-ca 
    environment: 
      - FABRIC_CA_HOME=/etc/hyperledger/fabric-ca-server 
      - FABRIC_CA_SERVER_CA_NAME=ca0 
      - FABRIC_CA_SERVER_TLS_ENABLED=false 
    ports: 
      - "7054:7054" 
    # 指定ca节点的公钥和私钥，给用户签发证书的时候需要用到，ca的公钥文件名固定，私钥文件名每次都会变
    command: sh -c 'fabric-ca-server start --ca.certfile /etc/hyperledger/fabric-ca-server-config/ca.org1.example.com-cert.pem --ca.keyfile /etc/hyperledger/fabric-ca-server-config/${PRIVATE_KEY} -b admin:adminpw -d' 
    volumes: 
      - ./crypto-config/peerOrganizations/org1.example.com/ca/:/etc/hyperledger/fabric-ca-server-config 
    container_name: ca0
```

```
# 加入TLS ca节点
  tlsca_org1:
    image: hyperledger/fabric-ca 
    environment: 
      - FABRIC_CA_HOME=/etc/hyperledger/fabric-ca-server 
      - FABRIC_CA_SERVER_CA_NAME=tlsca_org1
      - FABRIC_CA_SERVER_TLS_ENABLED=false
    ports: 
      - "8054:7054" 
    # 指定ca节点的公钥和私钥，给用户签发证书的时候需要用到，ca的公钥文件名固定，私钥文件名每次都会变
    command: sh -c 'fabric-ca-server start --ca.certfile /etc/hyperledger/fabric-ca-server-config/tlsca.org1.example.com-cert.pem --ca.keyfile /etc/hyperledger/fabric-ca-server-config/${PRIVATE_KEY_TLSCA_ORG1} -b admin:adminpw -d'
    volumes: 
      - ./crypto-config/peerOrganizations/org1.example.com/tlsca/:/etc/hyperledger/fabric-ca-server-config 
    container_name: tlsca_org1
```



### 2.2. 修改network_setup.sh脚本中的networkUp函数，增加如下内容
```
function netwrokUp(){
    
    ......

    folder="crypto-config/peerOrganizations/org1.example.com/ca" 
    privName="" 
    for file_a in ${folder}/* 
    do 
    temp_file=`basename $file_a`

    if [ ${temp_file##*.} != "pem" ];then 
    privName=$temp_file 
    fi 
    done 
    echo $privName 

folder="crypto-config/peerOrganizations/org1.example.com/tlsca" 
privName_TLS="" 
for file_a in ${folder}/* 
do 
    temp_file=`basename $file_a`

    if [ ${temp_file##*.} != "pem" ];then 
       privName_TLS=$temp_file 
    fi 
done 
    echo $privName_TLS

    ......

#     CHANNEL_NAME=$CH_NAME TIMEOUT=$CLI_TIMEOUT docker-compose -f $COMPOSE_FILE up -d 2>&1
    
    if [ "${IF_COUCHDB}" == "couchdb" ]; then
      CHANNEL_NAME=$CH_NAME TIMEOUT=$CLI_TIMEOUT PRIVATE_KEY=$privName PRIVATE_KEY_TLSCA_ORG1=$privName_TLS docker-compose -f $COMPOSE_FILE -f $COMPOSE_FILE_COUCH up -d 2>&1
    else
      CHANNEL_NAME=$CH_NAME TIMEOUT=$CLI_TIMEOUT PRIVATE_KEY=$privName PRIVATE_KEY_TLSCA_ORG1=$privName_TLS docker-compose -f $COMPOSE_FILE up -d 2>&1
fi
    ......
```

### 2.3. 修改完成后，重新启动cacheForRWConflict项目的fabric网络
```
./network_setup.sh down
./network_setup.sh up
```

### 2.4. 在项目的base目录下面新建add-peer2-org1-base.yaml
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

### 2.5. 在项目的根目录下面新建add-peer2-org1.yaml

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

### 2.6. 基于管理org1身份证书的ca0节点，为新增的peer2.org1.example.com签发证书。

```
export FABRIC_CA_CLIENT_HOME=$HOME/ca

fabric-ca-client enroll -u http://admin:adminpw@localhost:7054

fabric-ca-client register --id.name peer2.org1.example.com --id.type peer --id.affiliation org1.department1 --id.secret peer2pw

fabric-ca-client enroll -u http://peer2.org1.example.com:peer2pw@localhost:7054 -M $FABRIC_CA_CLIENT_HOME/peer2.org1.example.com/msp
```

### 2.7. 将peer2.org1.example.com文件夹复制到 项目中的如下目录
```
cp -r $FABRIC_CA_CLIENT_HOME/peer2.org1.example.com ${project dir}/crypto-config/peerOrganizations/org1/example.com/peers
```

### 2.8. 为peer2.org1.example.com增加admin证书

- 在${project dir}/crypto-config/peerOrganizations/org1/example.com/peers目录下面新建
admincerts文件夹，  mkdir admincerts ，并且将signcerts文件夹中的cert.pem复制到admincerts文件夹中。

### 2.9. 基于管理org1中tls通信的tlsca_org1节点，为新增的peer2.org1.example.com签发tls通信证书。
```
export FABRIC_CA_CLIENT_HOME=$HOME/tlsca
fabric-ca-client enroll -u http://admin:adminpw@localhost:8054

fabric-ca-client register --id.name peer2.org1.example.com --id.type peer --id.affiliation org1.department1 --id.secret peer2pw

fabric-ca-client enroll -d --enrollment.profile tls -u http://peer2.org1.example.com:peer2pw@localhost:8054 -M $FABRIC_CA_CLIENT_HOME/peer2.org1.example.com/tlsmsp --csr.hosts peer2.org1.example.com

```

### 2.10. 将peer2.org1.example.com文件夹复制到 项目中的如下目录
```
cp $FABRIC_CA_CLIENT_HOME/peer2.org1.example.com/tlsmsp/tlscacerts/* ${project dir}/crypto-config/peerOrganizations/org1/example.com/peers/peer2.org1.example.com/tls/ca.crt

cp $FABRIC_CA_CLIENT_HOME/peer2.org1.example.com/tlsmsp/signcerts/* ${project dir}/crypto-config/peerOrganizations/org1/example.com/peers/peer2.org1.example.com/tls/server.crt

cp $FABRIC_CA_CLIENT_HOME/peer2.org1.example.com/tlsmsp/keystore/* ${project dir}/crypto-config/peerOrganizations/org1/example.com/peers/peer2.org1.example.com/tls/server.key
```

### 2.11. 启动新增加的peer2.org1.example.com节点容器
```
docker-compose -f add-peer2-org1.yaml up -d
```

### 2.12. 进入cli容器，并且切换到新增的peer2.org1.example.com节点的环境变量
```
docker exec -it cli bash
```

- 切换到peer2.org1.example.com的环境变量
```
export CORE_PEER_ADDRESS=peer2.org1.example.com:7051

export CORE_PEER_LOCALMSPID="Org1MSP"

export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer2.org1.example.com/tls/ca.crt

export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
```

### 2.13. 将新增的peer2.org1.example.com节点加入channel，并且检查是否加入成功
```
peer channel join -b mychannel.block
peer channel list
```

### 2.14. 为新增的peer2.org1.example.com节点安装chaincode，并且实例化。实例化省略也可以，在sdk调用chaincode的时候会自动实例化。全部完成后，就可以通过sdk查询新增的peer节点账本了。
```
peer chaincode install -n cacheForRWConflict -v 1.0 -p github.com/hyperledger/fabric/examples/chaincode/   

ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem    

peer chaincode instantiate -o orderer.example.com:7050 --tls true --cafile $ORDERER_CA -C mychannel -n cacheForRWConflict -v 1.0 -c '{"Args":[""]}' -P "OR ('Org1MSP.peer','Org2MSP.peer')"
```