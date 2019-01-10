# revoke CA节点签发的user证书 #
## 1. 概述 ##
基于fabric cacheForRWConflict项目，对CA签发的user证书进行撤销
cacheForRWConflict项目参考http://10.167.35.183:8080/fabric/cacheForRWConflict/tree/master


## 2. 为user签发证书 ##
### 2.1. enroll CA节点的admin账号

```
export FABRIC_CA_CLIENT_HOME=$HOME/ca
fabric-ca-client enroll -u http://admin:adminpw@localhost:7054

```
### 2.2. 向fabric CA server register user账号 ###

```
fabric-ca-client identity add huasong --json '{"secret": "123456", "type": "user", "affiliation": "org1.department1", "max_enrollments": 1, "attrs": [{"name": "hf.Revoker", "value":"true"},{"name":"role","value":"auditor"},{"name":"enterprise","value":"Cosine"}]}'
```

### 2.3. 将user enroll进行enroll，为user账号签发证书

```
fabric-ca-client enroll -u http://huasong:123456@localhost:7054 -M $FABRIC_CA_CLIENT_HOME/huasong/msp
```

### 2.4. 将签发的msp目录复制到项目中存放org1的user证书目录下面

```
cp -r $FABRIC_CA_CLIENT_HOME/huasong/ /home/zhangshenbin/blockchainProject/cacheForRWConflict/crypto-config/peerOrganizations/org1.example.com/users/
```

### 此时签发证书完成，user msp中的证书可以用于方位fabric网络，访问方式参考cacheForRWConflict项目



## 2. revoke user证书 ##


- 在项目中新增revokeCert.sh

```
# revoke user name
USER_NAME=$1
: ${USER_NAME:=huasong2}

ORG_ADMIN_HOME=/home/zhangshenbin/blockchainProject/cacheForRWConflict/crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com

ORG_CA_ADMIN_HOME=/home/zhangshenbin/ca

# USER_NAME=huasong2

# org1的ca节点的admin帐号
ADMIN_NAME=admin
CA_HOST=localhost:7054
ORG=org1

# order节点
ORDERER_HOST=order.example.com

# Revokes the fabric user
function revokeFabricUserAndGenerateCRL {
   switchToAdminIdentity
   export  FABRIC_CA_CLIENT_HOME=$ORG_CA_ADMIN_HOME
   echo "Revoking the user '$USER_NAME' of the organization '$ORG' with Fabric CA Client home directory set to $FABRIC_CA_CLIENT_HOME and generating CRL ..."
  #  fabric-ca-client revoke -d --revoke.name $USER_NAME --gencrl
   fabric-ca-client revoke -d --revoke.name $USER_NAME
   fabric-ca-client gencrl -M $ORG_CA_ADMIN_HOME/msp

   cp -r $ORG_CA_ADMIN_HOME/msp/crls/ $ORG_ADMIN_HOME/msp/
}

# Switch to the current org's admin identity.  Enroll if not previously enrolled.
function switchToAdminIdentity {
   if [ ! -d $ORG_CA_ADMIN_HOME ]; then
      log "Enrolling admin '$ADMIN_NAME' with $CA_HOST ..."
      export FABRIC_CA_CLIENT_HOME=$ORG_CA_ADMIN_HOME
      fabric-ca-client enroll -d -u http://admin:adminpw@localhost:7054
   fi
   export FABRIC_CA_CLIENT_HOME=$ORG_CA_ADMIN_HOME
}

# export FABRIC_CA_CLIENT_HOME=$HOME/fabric-ca/clients/admin
# fabric-ca-client revoke -e peer1

# Revoke the user and generate CRL using admin's credentials
  revokeFabricUserAndGenerateCRL
```

- 运行revokeCert.sh  ./revokeCert.sh
 该脚本基于被revoke的user账号，生成CRL文件夹，并将其复制到org1证书目录下的Admin@org1.example.com/msp里面


- 为peer0.org1.example.com容器安装jq工具

```
docker exec -it cli bash
export http_proxy=http://10.167.32.133:8080
export https_proxy=http://10.167.32.133:8080
apt update && apt install -y jq
```

- 在项目中新增updateConfig.sh

```
echo 'update config'



CHANNEL_NAME=mychannel

# ORG=org1  这个地方错了  ORG=OrdererOrg
# 以org1的peer身份提交
ORG=Org1MSP

CONFIG_BLOCK_FILE=/tmp/config_block.pb
# Update config block payload file path
CONFIG_UPDATE_ENVELOPE_FILE=/tmp/config_update_as_envelope.pb

# order节点
ORDERER_TLS_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
ORDERER_HOST=orderer.example.com

# initPeerVars <ORG> <NUM>
# 默认是以peer0.org1.example.com的身份提交update config block
function initPeerVars {
   MYHOME=/opt/gopath/src/github.com/hyperledger/fabric/peer
   TLSDIR=$MYHOME/tls
   PEER_HOST=peer0.org1.example.com
   ORG_MSP_ID="Org1MSP"
   export FABRIC_CA_CLIENT=$MYHOME
   export CORE_PEER_ID=$PEER_HOST
   export CORE_PEER_ADDRESS=$PEER_HOST:7051
   export CORE_PEER_LOCALMSPID=$ORG_MSP_ID
   export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
   export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
}

function fetchConfigBlock {
   echo "Fetching the configuration block of the channel '$CHANNEL_NAME'"
   peer channel fetch config $CONFIG_BLOCK_FILE -c $CHANNEL_NAME -o $ORDERER_HOST:7050 --tls --cafile $ORDERER_TLS_CA
}

function createConfigUpdatePayloadWithCRL {
   echo "Creating config update payload with the generated CRL for the organization '$ORG'"
   # Start the configtxlator
   configtxlator start &
   configtxlator_pid=$!
   echo "configtxlator_pid:$configtxlator_pid"
   echo "Sleeping 5 seconds for configtxlator to start..."
   sleep 5

   pushd /tmp

   CTLURL=http://127.0.0.1:7059
   # Convert the config block protobuf to JSON  转换config_block.pb 为json格式
   curl -X POST --data-binary @$CONFIG_BLOCK_FILE $CTLURL/protolator/decode/common.Block > config_block.json
   
   # install jq
   if hash jq 2>/dev/null; then
       echo "jq tool exist"
   else
       echo "jq tool not exist, install it"
       export http_proxy=http://10.167.32.133:8080
       export https_proxy=http://10.167.32.133:8080
       apt update && apt install -y jq
   fi
   
   # Extract the config from the config block  从config_block.json中抽取config.json
   jq .data.data[0].payload.data.config config_block.json > config.json


   # Update crl in the config json  在config.json 加入crl信息，然后转换为updated_config.json
   crl=$(cat $CORE_PEER_MSPCONFIGPATH/crls/crl*.pem | base64 | tr -d '\n')
#    echo '.channel_group.groups.Application.groups.'"${ORG}"'.values.MSP.value.config.revocation_list = ["'"${crl}"'"]'

   cat config.json | jq '.channel_group.groups.Application.groups.'"${ORG}"'.values.MSP.value.config.revocation_list = ["'"${crl}"'"]' > updated_config.json

   # Create the config diff protobuf
   # 将config.json和加入crl的updated_config.json分别转换为config.pb和updated_config.pb
   # 计算config.pb和updated_config.pb之间的差值， 然后将差值转换为 config_update.pb
   curl -X POST --data-binary @config.json $CTLURL/protolator/encode/common.Config > config.pb
   curl -X POST --data-binary @updated_config.json $CTLURL/protolator/encode/common.Config > updated_config.pb
   curl -X POST -F original=@config.pb -F updated=@updated_config.pb $CTLURL/configtxlator/compute/update-from-configs -F channel=$CHANNEL_NAME > config_update.pb

   # Convert the config diff protobuf to JSON 将差值config_update.pb 转换为 config_update.json
   curl -X POST --data-binary @config_update.pb $CTLURL/protolator/decode/common.ConfigUpdate > config_update.json
   
   # Create envelope protobuf container config diff to be used in the "peer channel update" command to update the channel configuration block
   # 在config_update.json文件内容封装成为config_update_as_envelope.json
   # 并将其转换为/tmp/config_update_as_envelope.pb文件
   echo '{"payload":{"header":{"channel_header":{"channel_id":"'"${CHANNEL_NAME}"'", "type":2}},"data":{"config_update":'$(cat config_update.json)'}}}' > config_update_as_envelope.json
   curl -X POST --data-binary @config_update_as_envelope.json $CTLURL/protolator/encode/common.Envelope > $CONFIG_UPDATE_ENVELOPE_FILE

   # Stop configtxlator
   kill $configtxlator_pid
#    echo "fineshed"
   popd
}

function updateConfigBlock {
   echo "Updating the configuration block of the channel '$CHANNEL_NAME'"
   peer channel update -f $CONFIG_UPDATE_ENVELOPE_FILE -c $CHANNEL_NAME -o $ORDERER_HOST:7050 --tls --cafile $ORDERER_TLS_CA
}

initPeerVars
fetchConfigBlock

# Create config update envelope with CRL and update the config block of the channel
createConfigUpdatePayloadWithCRL

updateConfigBlock

```
- 在peer0.org1.example.com容器内运行updateConfig.sh脚本


```
docker exec -it cli bash -c './scripts/updateConfig.sh'
```
