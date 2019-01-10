# hyperledger explorer部署 #
## 1. 概述 ##
基于fabric的区块链数据浏览。
github主页https://github.com/hyperledger/blockchain-explorer





## 2. requirements ##
```
nodejs 8.11.x (Note that v9.x is not yet supported)
PostgreSQL 9.5 or greater
  * sudo apt-get update
  * sudo apt-get install postgresql
Jq [https://stedolan.github.io/jq/]
docker 17.06.2-ce [https://www.docker.com/community-edition]
docker-compose 1.14.0 [https://docs.docker.com/compose/]
```

## 3. explorer部署流程 ##

### 3.1. clone项目源代码
···
git clone https://github.com/hyperledger/blockchain-explorer.git.
cd blockchain-explorer
git checkout release-3.5 切换到3.5版本(适用于fabric1.1)
···

### 3.2. 设置postgreSQL数据库信息

* 设置explorerconfig.json
```
cd blockchain-explorer/app
"postgreSQL": {
  "host": "127.0.0.1",
  "port": "5432",
  "database": "fabricexplorer",
  "username": "hppoc",
  "passwd": "password"
}
```

### 3.3. 启动postgreSQL数据库
```
cd blockchain-explorer/app/platform/fabric
修改config.json文件，将fabric网络的项目根目录改成自己的
{
  "network-config": {
    "org1": {
      "name": "Org1MSP",
      "mspid": "Org1MSP",
      "peer1": {
        "requests": "grpcs://127.0.0.1:7051",
        "events": "grpcs://127.0.0.1:7053",
        "server-hostname": "peer0.org1.example.com",
        "tls_cacerts":
          "/home/zhangshenbin/blockchainProject/cacheForRWConflict/crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt"
      },
      "peer2": {
        "requests": "grpcs://127.0.0.1:8051",
        "events": "grpcs://127.0.0.1:8053",
        "server-hostname": "peer1.org1.example.com",
        "tls_cacerts":
          "/home/zhangshenbin/blockchainProject/cacheForRWConflict/crypto-config/peerOrganizations/org1.example.com/peers/peer1.org1.example.com/tls/ca.crt"
      },
      "admin": {
        "key":
          "/home/zhangshenbin/blockchainProject/cacheForRWConflict/crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/keystore",
        "cert":
          "/home/zhangshenbin/blockchainProject/cacheForRWConflict/crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/signcerts"
      }
    },
    "org2": {
      "name": "Org2MSP",
      "mspid": "Org2MSP",
      "peer1": {
        "requests": "grpcs://127.0.0.1:9051",
        "events": "grpcs://127.0.0.1:9053",
        "server-hostname": "peer0.org2.example.com",
        "tls_cacerts":
          "/home/zhangshenbin/blockchainProject/cacheForRWConflict/crypto-config/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt"
      },
      "peer2": {
        "requests": "grpcs://127.0.0.1:10051",
        "events": "grpcs://127.0.0.1:10053",
        "server-hostname": "peer1.org2.example.com",
        "tls_cacerts":
          "/home/zhangshenbin/blockchainProject/cacheForRWConflict/crypto-config/peerOrganizations/org2.example.com/peers/peer1.org2.example.com/tls/ca.crt"
      },
      "admin": {
        "key":
          "/home/zhangshenbin/blockchainProject/cacheForRWConflict/crypto-config/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp/keystore",
        "cert":
          "/home/zhangshenbin/blockchainProject/cacheForRWConflict/crypto-config/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp/signcerts"
      }
    }
  },
  "channel": "mychannel",
  "orderers": [
    {
      "mspid": "OrdererMSP",
      "server-hostname": "orderer.example.com",
      "requests": "grpcs://127.0.0.1:7050",
      "tls_cacerts":
        "/home/zhangshenbin/blockchainProject/cacheForRWConflict/crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/tls/ca.crt"
    }
  ],
  "keyValueStore": "/tmp/fabric-client-kvs",
  "configtxgenToolPath": "fabric-path/fabric-samples/bin",
  "SYNC_START_DATE_FORMAT": "YYYY/MM/DD",
  "syncStartDate": "2018/01/01",
  "eventWaitTime": "30000",
  "license": "Apache-2.0",
  "version": "1.1"
}

```

### 3.4. 重新打开一个终端
```
cd blockchain-explorer
npm install
cd blockchain-explorer/app/test
npm install
npm run test
cd client/
npm install
npm test -- -u --coverage
npm run build
```

### 3.5. 启动hyperledger explorder

```
cd blockchain-explorer/
./start.sh (it will have the backend up).
Launch the URL http://localhost:8080 on a browser.
./stop.sh (it will stop the node server).
```