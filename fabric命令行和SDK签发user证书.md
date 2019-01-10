# fabric CA节点向user签发证书 #
## 1. 概述 ##
基于fabric cacheForRWConflict项目，CA签发user证书
cacheForRWConflict项目参考http://10.167.35.183:8080/fabric/cacheForRWConflict/tree/master


## 2. 基于命令行方式为user签发证书 ##

### 2.1. CA节点管理员账号为admin，密码为adminpw, enroll CA节点的admin账号
```
export FABRIC_CA_CLIENT_HOME=$HOME/ca
fabric-ca-client enroll -u http://admin:adminpw@localhost:7054

```
### 2.2. 向fabric CA server register user账号 ###

```
fabric-ca-client identity add huasong --json '{"secret": "123456", "type": "user", "affiliation": "org1.department1", "max_enrollments": 1, "attrs": [{"name": "hf.Revoker", "value":"true"},{"name":"role","value":"auditor","ecert":true},{"name":"enterprise","value":"Cosine","ecert":true}]}'
```

### 2.3. 将user账号进行enroll，为user账号签发证书

```
fabric-ca-client enroll -u http://huasong:123456@localhost:7054 -M $FABRIC_CA_CLIENT_HOME/huasong/msp
```

### 2.4. 将签发的msp目录复制到项目中存放org1的user证书目录下面

```
cp -r $FABRIC_CA_CLIENT_HOME/huasong/ /home/zhangshenbin/blockchainProject/cacheForRWConflict/crypto-config/peerOrganizations/org1.example.com/users/
```

### 此时签发证书完成，user msp中的证书可以用于方位fabric网络，访问方式参考cacheForRWConflict项目

## 3. 基于fabric SDK方式为user签发证书 ##

### 3.1. 首先将CA管理员enroll，生成管理员的证书。
CA节点管理员账号为admin，密码为adminpw, enroll CA节点的admin账号
```
export FABRIC_CA_CLIENT_HOME=$HOME/ca
fabric-ca-client enroll -u http://admin:adminpw@localhost:7054
```

### 3.2. 调用issueCert.js中的registerAndEnrollUser(userName, role, enterprise)函数，userName表示用户id，role表示自定义属性角色，enterprise表示自定义属性企业


```
'use strict';
/*
* Copyright IBM Corp All Rights Reserved
*
* SPDX-License-Identifier: Apache-2.0
*/
/*
 * Register and Enroll a user
 */

var Fabric_Client = require('fabric-client');
var Fabric_CA_Client = require('fabric-ca-client');
var path = require('path');
var fs = require('fs');
var sdkUtils = require('fabric-client/lib/utils')
var child_process = require('child_process');
var fabric_client = new Fabric_Client();
var fabric_ca_client = null;
var ca_admin_user = null;
var member_user = null;

var FABRIC_CA_CLIENT_HOME="/home/zhangshenbin/ca";
var store_path = path.join(__dirname, 'hfc-msp-store');

console.log(' Store path:'+store_path);

// 递归创建目录 同步方法
function mkdirsSync(dirname) {
    if (fs.existsSync(dirname)) {
      return true;
    } else {
      if (mkdirsSync(path.dirname(dirname))) {
        fs.mkdirSync(dirname);
        return true;
      }
    }
  }

registerAndEnrollUser("huasong18","auditor","Cosine");


//fabric ca msp身份
var ca_options = {
    user_id:'admin',
    msp_id:'Org1MSP',    //org1.department1  随便设定，不起作用
    privateKeyFolder:'/home/zhangshenbin/ca/msp/keystore', 
    signedCert:'/home/zhangshenbin/ca/msp/signcerts/cert.pem',
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

function registerAndEnrollUser(userName, role, enterprise){
    
    return Promise.resolve().then(
        
        ()=>{
            //指定当前用户的私钥，证书等基本信息 
            console.log("Load privateKey and signedCert"); 
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
            (state_store) => {
                // assign the store to the fabric client
                // state store是证书信息的json文件
                fabric_client.setStateStore(state_store);

                var	tlsOptions = {
                    trustedRoots: [],
                    verify: false
                };
                // be sure to change the http to https when the CA is running TLS enabled
                // fabric_ca_client = new Fabric_CA_Client('http://localhost:7054', tlsOptions , '', crypto_suite)
                fabric_ca_client = new Fabric_CA_Client('http://localhost:7054', tlsOptions , 'ca0');
                
                return fabric_client.createUser(createUserOpt);
            }
        ).then(
            (caAdminUser) => {
                if (caAdminUser && caAdminUser.isEnrolled()) {
                    console.log('Successfully loaded admin from persistence');
                    ca_admin_user = caAdminUser;
                } else {
                    throw new Error('Failed to get admin.... run enrollAdmin.js');
                }
                // at this point we should have the ca admin user
                // first need to register the user with the CA server
                //user证书属性为: id.attrs hf.Revoker=true,role=auditor,enterprise=Cosine
                var attrsArray = [ {name:"role", value:role, ecert:true}, 
                                   {name:"enterprise", value:enterprise, ecert:true} ];
                return fabric_ca_client.register({enrollmentID: userName, 
                enrollmentSecret:"123456", affiliation: 'org1.department1',
                role: 'client', attrs: attrsArray }, ca_admin_user);
            }
      
        ).then(
            () => {
                var crypto_suite = Fabric_Client.newCryptoSuite();
                var crypto_store = Fabric_Client.newCryptoKeyStore({path: store_path});
                crypto_suite.setCryptoKeyStore(crypto_store);
                fabric_client.setCryptoSuite(crypto_suite);

                // next we need to enroll the user with CA server
                console.log('Successfully registered '+ userName + ' - secret:'+ "123456");
                return fabric_ca_client.enroll({enrollmentID: userName, enrollmentSecret: "123456"});
            }
        ).then((enrollment) => {
            console.log('Successfully enrolled member user '+userName);

            console.log(enrollment.key.toBytes());
            console.log(enrollment.certificate);

            var keystorePath =  __dirname + "/hfc-msp-store/" + userName + "/msp/keystore";
            var signcertsPath =  __dirname + "/hfc-msp-store/" + userName + "/msp/signcerts";
            var cacertsPath =  __dirname + "/hfc-msp-store/" + userName + "/msp/cacerts";

            mkdirsSync(keystorePath);
            mkdirsSync(signcertsPath);
            mkdirsSync(cacertsPath);

            var orginalCaCertsPath =  FABRIC_CA_CLIENT_HOME + "/msp/cacerts/" ;
            var destCaCertsPath =  __dirname + "/hfc-msp-store/" + userName + "/msp/";
            child_process.spawn('cp', ['-r', orginalCaCertsPath, destCaCertsPath]);    

            fs.writeFileSync( keystorePath + "/priv_sk" , enrollment.key.toBytes());
            fs.writeFileSync( signcertsPath + "/cert.pem" , enrollment.certificate );

            return fabric_client.createUser(
                {
                    username: userName,
                    mspid: 'Org1MSP',
                    cryptoContent: { privateKeyPEM: enrollment.key.toBytes(), signedCertPEM: enrollment.certificate }
                }
                );
            }
        ).then((user) => {
            member_user = user;
            return fabric_client.setUserContext(member_user);
            }
        ).then(
            ()=>{
                console.log(userName + ' was successfully registered and enrolled and is ready to intreact with the fabric network');
            }
        ).catch(
            (err) => {
                console.error('Failed to register: ' + err);
                if(err.toString().indexOf('Authorization') > -1) {
                    console.error('Authorization failures may be caused by having admin credentials from a previous CA instance.\n' +
                    'Try again after deleting the contents of the store directory '+store_path);
                }
            }
        );
        }
    );
}
```