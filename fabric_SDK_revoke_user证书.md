# revoke CA节点签发的user证书 #
## 1. 概述 ##
- 基于fabric cacheForRWConflict项目，使用node SDK对CA签发的user证书进行撤销
- cacheForRWConflict项目参考http://10.167.35.183:8080/fabric/cacheForRWConflict/tree/master

## 2. 主要步骤 ##
- (1) 设置fabric client的证书信息
- (2) 设置fabric_ca_client的证书信息为Admin@org1.example.com
- (3) 初始化channel，将peer0.org1.example.com和order.example.com节点加入channel
- (4) 基于fabric_ca_client revoke user证书
- (5) 基于fabric_ca_client生成crl列表
- (6) 获取channel config_block.pb信息，生成original_config_proto
- (7) 将original_config_proto转换成json对象update_config
- (8) 将crl列表信息加入到update_config中，update_config.channel_group.groups.Application.groups.Org1MSP.values.MSP.value.config.revocation_list = [CRL_FILE];
- (9) 将json对象update_config转换成json字符串，再转换成pb格式update_config_proto
- (10) 计算original_config_proto和update_config_proto的差值pb，生成config_proto
- (11) 将差值pb config_proto上链


## 3. 为user签发证书 ##
### 3.1. enroll CA节点的admin账号

```
export FABRIC_CA_CLIENT_HOME=$HOME/ca
fabric-ca-client enroll -u http://admin:adminpw@localhost:7054

```
### 3.2. 向fabric CA server register user账号 ###

```
fabric-ca-client identity add huasong2 --json '{"secret": "123456", "type": "user", "affiliation": "org1.department1", "max_enrollments": 1, "attrs": [{"name": "hf.Revoker", "value":"true"},{"name":"role","value":"auditor"},{"name":"enterprise","value":"Cosine"}]}'
```

### 3.3. 将user enroll进行enroll，为user账号签发证书

```
fabric-ca-client enroll -u http://huasong2:123456@localhost:7054 -M $FABRIC_CA_CLIENT_HOME/huasong/msp
```

### 3.4. 将签发的msp目录复制到项目中存放org1的user证书目录下面

```
cp -r $FABRIC_CA_CLIENT_HOME/huasong2/ /home/zhangshenbin/blockchainProject/cacheForRWConflict/crypto-config/peerOrganizations/org1.example.com/users/
```

### 此时签发证书完成，user msp中的证书可以用于方位fabric网络，访问方式参考cacheForRWConflict项目



## 4. 基于fabric SDK 撤销user证书 ##
### 4.1. 打开一个新的终端，开启configtxlator服务

```
configtxlator start
```

### 4.2. node SDK撤销user证书 ###

- 调用revoke函数撤销user证书

```
revoke("huasong2").then((result) => {
    console.log("revoke successful !!!!\n" + result);
});
```

- revoke函数代码

```
'use strict'
/*
	Revoke the a user in org
*/

var hfc = require('fabric-client');
var sdkUtils = require('fabric-client/lib/utils')

var Fabric_CA_Client = require('fabric-ca-client');
var path = require('path');
var agent = require('superagent-promise')(require('superagent'), Promise);
var requester = require('request');
var fs = require('fs');

var ca_admin_user = null;
var CRL_FILE = null;

//增量pb文件
var config_proto = null;

var original_config_proto = null;


//fabric ca msp身份
var ca_options = {
    user_id:'admin',
    msp_id:'Org1MSP',    //org1.department1  随便设定，不起作用
    privateKeyFolder:'/home/zhangshenbin/ca/msp/keystore', 
    signedCert:'/home/zhangshenbin/ca/msp/signcerts/cert.pem',
};

// Admin@org1.example.com/msp 身份
var options = {
    user_id:'Admin@org1.example.com',
    msp_id:'Org1MSP', 
    channel_id: 'mychannel', 
    //peer节点 chaincode 打包后的名字
    chaincode_id: 'cacheForRWConflict',
    //连接的peer url
    peer_url: 'grpcs://localhost:7051',//因为启用了TLS，所以是grpcs,如果没有启用TLS，那么就是grpc 
    //接收事件url
    event_url: 'grpcs://localhost:7053',//因为启用了TLS，所以是grpcs,如果没有启用TLS，那么就是grpc 
    //共识url
    orderer_url: 'grpcs://localhost:7050',//因为启用了TLS，所以是grpcs,如果没有启用TLS，那么就是grpc 
    //E-cert 私钥
    privateKeyFolder:'/home/zhangshenbin/blockchainProject/cacheForRWConflict/crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/keystore', 
    //E-cert 签名的公钥证书
    signedCert:'/home/zhangshenbin/blockchainProject/cacheForRWConflict/crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/signcerts/Admin@org1.example.com-cert.pem', 
    //通信用的tls ca证书  公钥加密通信信息 服务端用私钥解密
    peer_tls_cacerts:'/home/zhangshenbin/blockchainProject/cacheForRWConflict/crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt', 
        
    orderer_tls_cacerts: '/home/zhangshenbin/blockchainProject/cacheForRWConflict/crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/tls/ca.crt',

    server_hostname: "peer0.org1.example.com" 
};

const getKeyFilesInDir = (dir) => { 
    //该函数用于找到keystore目录下的私钥文件的路径 
    const files = fs.readdirSync(dir) 
    const keyFiles = [] 
    files.forEach((file_name) => { 
        let filePath = path.join(dir, file_name) 
        if (file_name.endsWith('_sk')) { 
            keyFiles.push(filePath) 
        } 
    }) 
    return keyFiles
}


revoke("huasong2").then((result) => {
    console.log("revoke successful !!!!\n" + result);
});


function revoke(userName){

    var clientStore = null;
    var channel = {};
    var orderer = null;

    var client = null;
    var tx_id = null;
    var signatures = [];
    var request = null;
    var fabric_ca_client = null;

    
    return Promise.resolve().then(
        () => {
            //指定当前用户的私钥，证书等基本信息 
            console.log("Load privateKey and signedCert"); 
            client = new hfc(); 
            //设置ca证书的选项
            var createUserOpt = { 
                username: ca_options.user_id, 
                mspid: ca_options.msp_id, 
                cryptoContent: { 
                      privateKey: getKeyFilesInDir(ca_options.privateKeyFolder)[0], 
                      signedCert: ca_options.signedCert
                } 
            }
            return sdkUtils.newKeyValueStore(
                { 
                 path: "/tmp/fabric-client-stateStore/"
                }
            ).then(
                (store) => {
                    clientStore = store;
                    client.setStateStore(store);
                    // first check to see if the admin is already enrolled
                    var	tlsOptions = {
                        trustedRoots: [],
                        verify: false
                    };
                    fabric_ca_client = new Fabric_CA_Client('http://localhost:7054', tlsOptions , 'ca0');
              
                    return client.createUser(createUserOpt);
                }
            ).then(
                (caAdminUser)=>{
                    if (caAdminUser && caAdminUser.isEnrolled()) {
                        console.log('Successfully loaded admin from persistence');
                        ca_admin_user = caAdminUser;
                    } else {
                        throw new Error('Failed to get admin... run enrollAdmin.js');
                    }
                    client._userContext = null;
                    var createUserOpt = { 
                        username: options.user_id, 
                        mspid: options.msp_id, 
                        cryptoContent: { 
                              privateKey: getKeyFilesInDir(options.privateKeyFolder)[0], 
                              signedCert: options.signedCert
                        } 
                    }
                    client.setStateStore(clientStore);
                    return client.createUser(createUserOpt);

                }
            )
            .then(
                //配置channel peer orderer节点信息
                (org1AdminUser) => {
                    if (org1AdminUser && org1AdminUser.isEnrolled()) {
                        console.log('Successfully loaded admin from persistence');
                    } else {
                        throw new Error('Failed to get admin... run enrollAdmin.js');
                    }
                    //设置client连接的peer节点信息
                    //因为启用了TLS，所以指定Peer的TLS的CA证书，client必须有peer的tls ca证书，才能发送消息给peer
                    channel = client.newChannel(options.channel_id); 

                    let data = fs.readFileSync(options.peer_tls_cacerts); 
                    let peer = client.newPeer(options.peer_url, 
                        { 
                            pem: Buffer.from(data).toString(), 
                            'ssl-target-name-override': options.server_hostname 
                        }
                    );
                    channel.addPeer(peer); 
                    //接下来连接Orderer的时候也启用了TLS，也是同样的处理方法
                    let odata = fs.readFileSync(options.orderer_tls_cacerts); 
                    let orderTlsPem = Buffer.from(odata).toString(); 
                    orderer = client.newOrderer(options.orderer_url, { 
                        'pem': orderTlsPem, 
                        'ssl-target-name-override': "orderer.example.com"
                    }); 
                    channel.addOrderer(orderer); 
                    console.log('now revoke the user1');
                    return fabric_ca_client.revoke({enrollmentID: userName}, ca_admin_user);
                }
            ).then(

                (resp) => {
                    console.log('The revoke result is ' + resp.success);
                    var result = resp.success;
                    if (result === true) {
                        console.log('Successfully revoke the user1');
                    } else {
                        console.log('Operation is failed')
                    }
                    console.log("finish");

                    return fabric_ca_client.generateCRL( {},ca_admin_user); //base64编码的
                }

            ).then(

                (CRL) => {
                    if(!CRL) {
                        console.log('Fail to generateCRL');
                    }else {
                        console.log('Return result Successfully generateCRL');
                    }
                    CRL_FILE = CRL;
                    console.log(CRL_FILE)
                    // update crl into channel config block
                    // 1. first get the config Block in the channel

                    return channel.getChannelConfig();
                }

            ).then(

                (config_envelope) => {

                    // 2. translate the envolope to proto bytes
                    console.log('!!!!!!!!!!!config_envelope ' + config_envelope);
                    console.log('!!!!!!!!!!!config_envelope.config ' + config_envelope.config);
                    original_config_proto = config_envelope.config.toBuffer();
                
                    console.log('#######################################' + typeof(original_config_proto));
                    console.log('!!!!!!!!: ' + original_config_proto);
                    // 3. then get the config proto to JSON
                    console.log('3 get the original_config_proto successfully');
                   
                    return agent.post('http://127.0.0.1:7059/protolator/decode/common.Config', original_config_proto).buffer();
                }

            ).then(

                (response) => {

                    // get the original config in JSON  原始的configl block json
                    var original_config_json = response.text.toString();
                
                    // 4. make a copy of original config and make change 
                    var update_config_json = original_config_json;
                    var update_config = JSON.parse(update_config_json);
                    console.log('+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++');
                    console.log(update_config.channel_group.groups.Application.groups.Org1MSP.values.MSP.value.config);
                
                    // 5. now edit the update config 
                    update_config.channel_group.groups.Application.groups.Org1MSP.values.MSP.value.config.revocation_list = [CRL_FILE];
                    console.log('+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++');
                    console.log(update_config.channel_group.groups.Application.groups.Org1MSP.values.MSP.value.config);
                    update_config_json = JSON.stringify(update_config);
                
                    console.log("update_config_json: " + update_config_json);

                    // 6. get the proto byte of the update_config_json
                    return agent.post('http://127.0.0.1:7059/protolator/encode/common.Config', update_config_json.toString()).buffer();
                }

            ).then(

                (response) => {
                    var update_config_proto = response.body;
                                    
                    // 7. compute the diff between original and updated config proto
                    var formData = {
                        channel: 'mychannel',
                        original: {
                            value: original_config_proto,
                            options: {
                                filename: 'original.proto',
                                contentType: 'application/octet-stream'
                            }
                        },
                
                        updated: {
                            value: update_config_proto,
                            options: {
                                filename: 'updated.proto',
                                contentType: 'application/octet-stream'
                            }
                        }
                    };

                    return new Promise((resolve, reject) => {
                        requester.post({
                            url: 'http://127.0.0.1:7059/configtxlator/compute/update-from-configs',
                            //==============================================
                            encoding: null,
                            headers: {
                                accept: '/',
                                expect: '100-continue'
                            },
                            //===============================================
                            formData: formData
                        }, function optionalCallback(err, res, body) {
                            if (err){
                                console.log(err.toSting());
                                reject(err);
                            } else {
                               // console.log(body);
                                var proto = Buffer.from(body, 'binary');
                                resolve(proto);
                            }
                        });
                    });
                }

            ).then(
                (response) => {

                    config_proto = response;
                    var signature = client.signChannelConfig(config_proto);
                    signatures.push(signature);
                    
                    console.log(config_proto.toString());
 
                     tx_id = client.newTransactionID();
                     request = {
                         config: config_proto,
                         signatures : signatures,
                         name : 'mychannel',
                         orderer : orderer,
                         txId  : tx_id
                     };
                     //更新通道
                    console.log('ZZZZZZZZZZZZZZZZZ\n' + signatures)
                     
                    return client.updateChannel(request);
                }

            ).then(
                (result) => {
                    console.log(result);
                    if(result.status && result.status === 'SUCCESS') {
                        console.log('Successfully updated the channel.');
                        // return sleep(5000);
                        return;
                    } else {
                        console.error('Failed to update the channel. ');
                        Promise.reject('Failed to update the channel');
                    }
                }
            ).then(
                (nothing) => {
                    console.log("Successfully waited to make sure new channel was updated")
                }
            )
        }
    )
    ;
}

```