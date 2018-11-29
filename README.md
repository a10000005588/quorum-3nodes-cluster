# quorum-3nodes-cluster
Setting the quorum cluster with 3 nodes using the docker.

# Quorum-docker-nodes cluster usage


## Environment:

* Ubuntu: Ubuntu 16.04.5 LTS (In docker, I use xenial version from Ubuntu dockerhub)

* Quorum: v2.1.1
* Golang: go1.11.2.linux-amd64
* Tessera: 0.7 release


## Docker image:

### Port Usage 
raftport = 50400
rpcport = 21000
tessera server = 9000


### Step1: Start container for node 1

The quorum and tessera are already in the docker image:

docker image size : 2GB

pull the docker image into your nodes.
```
docker pull a10000005588/quorum-3nodes-cluster
```

run docker and bind the port

```
docker run -it -d -p 9000:9000 -p 50400:50400 -p 21000:21000 a10000005588/quorum-3nodes-cluster:latest
```


### Step2: Start tessera server
cd tessera and edit the tessera-init-sample.json

```
echo "[*] Initialising Tessera configuration"

currentDir=$(pwd)
    DDIR="${currentDir}/tdata"
    mkdir -p ${DDIR}
    mkdir -p ${DDIR}/logs

    cp -r "tessera-key" "${DDIR}"
    rm -f "${DDIR}/tm.ipc"

    #change tls to "strict" to enable it (don't forget to also change http -> https)
    cat <<EOF > ${DDIR}/tessera-config.json
{
    "useWhiteList": false,
    "jdbc": {
        "username": "sa",
        "password": "",
        "url": "jdbc:h2:${DDIR}/db$;MODE=Oracle;TRACE_LEVEL_SYSTEM_OUT=0"
    },
    "server": {
        "port": 9000,
        "hostName": "http://<node public ip>",   // add your node's public ip
        "bindingAddress": "http://0.0.0.0:9000",
        "sslConfig": {
            "tls": "OFF",
            "generateKeyStoreIfNotExisted": true,
            "serverKeyStore": "${DDIR}/server-keystore",
            "serverKeyStorePassword": "quorum",
            "serverTrustStore": "${DDIR}/server-truststore",
            "serverTrustStorePassword": "quorum",
            "serverTrustMode": "TOFU",
            "knownClientsFile": "${DDIR}/knownClients",
            "clientKeyStore": "${DDIR}/client-keystore",
            "clientKeyStorePassword": "quorum",
            "clientTrustStore": "${DDIR}/client-truststore",
            "clientTrustStorePassword": "quorum",
            "clientTrustMode": "TOFU",
            "knownServersFile": "${DDIR}/knownServers"
        }
    },
    "peer": [ // setting your peer's public ip 
        {
            "url": "http://<node1 public ip>:9000"
        },
        {
            "url": "http://<node2 public ip>:9000"
        },
        {
            "url": "http://<node3 public ip>:9000"
        }
    ],
    "keys": {
        "passwords": [],
        "keyData": [
            {   // if the node index is 1, modify <node-index> here with  node1  
                "privateKeyPath": "${DDIR}/tessera-key/<node-index>/tm.key",
                "publicKeyPath": "${DDIR}/tessera-key/<node-index>/tm.pub"
            }
        ]
    },
    "alwaysSendTo": [],
    "unixSocketFile": "${DDIR}/tm.ipc"
}
EOF
```


then, exec `./tessera-init-sample`, it will generate the data for tessera server.


Goto the directory: `tdata`, there should exist the `tessera-config.json`

and execute it with:
```
java -jar /home/test/quorum-3nodes-example/tessera/tessera-app-0.8-SNAPSHOT-app.jar -configfile tessera-config.json >> tessera.log 2>&1 &
```

use `tail -f tessera.log` to check the whether if the tessera server is operating or not.

### Step3: Start quourm node

cd to the directory which contains the `genesis.json` to initialize the quorum node data information with the following script:

```
geth --datadir /home/test/quorum-3nodes-example/node/ init /home/test/quorum-3nodes-example/genesis.json
```

then, cd to `node` directory

modify the `static-nodes-sample.json` with your 3 nodes public ip and rename it with the `static-nodes.json` in the same folder.


cp to the nodekey according to your node index
(ex: if you are in node1 server, copy the nodekey in node1 files to the directory where the static-nodes.json is.)



start the quorum node

```
PRIVATE_CONFIG=/home/test/quorum-3nodes-example/tessera/tdata/tm.ipc nohup geth --datadir node --nodiscover --verbosity 5 --networkid 31337 --raft --raftport 50400 --rpc --rpcaddr 0.0.0.0 --rpcport 22000 --rpcapi admin,db,eth,debug,miner,net,shh,txpool,personal,web3,quorum,raft --emitcheckpoints --port 21000 2>>node.log &
```

in node directory, exec `geth attach geth.ipc`

### Step4: Setting the node2 and node3.

repeat the step 1~3

### Step5: Deploy a private contract 

prepare a private contract which will send a private tx to node2 

(notice: in the 13 line, the privateFor must use the public key of the node2 which is in the tessera folder.)

```javascript=
a = eth.accounts[0]
web3.eth.defaultAccount = a;
web3.personal.unlockAccount(web3.eth.defaultAccount,'')
// abi and bytecode generated from simplestorage.sol:
// > solcjs --bin --abi simplestorage.sol
var abi = [{"constant":true,"inputs":[],"name":"storedData","outputs":[{"name":"","type":"uint256"}],"payable":false,"type":"function"},{"constant":false,"inputs":[{"name":"x","type":"uint256"}],"name":"set","outputs":[],"payable":false,"type":"function"},{"constant":true,"inputs":[],"name":"get","outputs":[{"name":"retVal","type":"uint256"}],"payable":false,"type":"function"},{"inputs":[{"name":"initVal","type":"uint256"}],"payable":false,"type":"constructor"}];

var bytecode = "0x6060604052341561000f57600080fd5b604051602080610149833981016040528080519060200190919050505b806000819055505b505b610104806100456000396000f30060606040526000357c0100000000000000000000000000000000000000000000000000000000900463ffffffff1680632a1afcd914605157806360fe47b11460775780636d4ce63c146097575b600080fd5b3415605b57600080fd5b606160bd565b6040518082815260200191505060405180910390f35b3415608157600080fd5b6095600480803590602001909190505060c3565b005b341560a157600080fd5b60a760ce565b6040518082815260200191505060405180910390f35b60005481565b806000819055505b50565b6000805490505b905600a165627a7a72305820d5851baab720bba574474de3d09dbeaabc674a15f4dd93b974908476542c23f00029";

// in privateFor, fill the node 2 tessera public key

var simpleContract = web3.eth.contract(abi);
var simple = simpleContract.new(42, {from:web3.eth.accounts[0], data: bytecode, gas: 0x47b760, privateFor: ["6BDsbX8Bp4A0E1TkuAxE/JM0Hv1/GgaAJwDOwITxvBA="]}, function(e, contract) {
        if (e) {
                console.log("err creating contract", e);
        } else {
                if (!contract.address) {
                        console.log("Contract transaction send: TransactionHash: " + contract.transactionHash + " waiting to be mined...");
                } else {
                        console.log("Contract mined! Address: " + contract.address);
                        console.log(contract);
                }
        }
});
```


ex send a private tx from node1 to node3

```shell
root@ccb6e9823ae1:/home/test/quorum-3nodes-example# ./runscript.sh private-contract.js

Contract transaction send: TransactionHash: 0x62c3abc504f4b65b2c240bc26ca2b0707382248a2652e234c9cff4a51156a45a waiting to be mined...
true
```

go to the geth console

use `
```
eth.getTransactionReceipt("0x62c3abc504f4b65b2c240bc26ca2b0707382248a2652e234c9cff4a51156a45a")`
```
to get the private contract address

```

> eth.getTransactionReceipt("0x62c3abc504f4b65b2c240bc26ca2b0707382248a2652e234c9cff4a51156a45a")
{
  blockHash: "0x639f7c4e6cba259b9d2af7610f490658d7c8113f731e82f7cd00b6eda8e05658",
  blockNumber: 2,
  contractAddress: "0x5afe80b74c89de63e25a942039f85f6d5a632765",
  cumulativeGasUsed: 0,
  from: "0xe496641297ce910f335b60793d6d2b569b6799ca",
  gasUsed: 0,
  logs: [],
  logsBloom: "0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
  status: "0x1",
  to: null,
  transactionHash: "0x62c3abc504f4b65b2c240bc26ca2b0707382248a2652e234c9cff4a51156a45a",
  transactionIndex: 0
}
>
```

check whether the tx is work or not
```
  contractAddress: "0x5afe80b74c89de63e25a942039f85f6d5a632765",


var address="0x5afe80b74c89de63e25a942039f85f6d5a632765"

var abi = [{"constant":true,"inputs":[],"name":"storedData","outputs":[{"name":"","type":"uint256"}],"payable":false,"type":"function"},{"constant":false,"inputs":[{"name":"x","type":"uint256"}],"name":"set","outputs":[],"payable":false,"type":"function"},{"constant":true,"inputs":[],"name":"get","outputs":[{"name":"retVal","type":"uint256"}],"payable":false,"type":"function"},{"inputs":[{"name":"initVal","type":"uint256"}],"type":"constructor"}];

var private = eth.contract(abi).at(address)
```

in node1, the result will be

```
private.get()
42
```

in node2, the result will be
```
private.get()
42
```

in node3, the result will be
```
private.get()
0
```

## Adding a new node.

...

