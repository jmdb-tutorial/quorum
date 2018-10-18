# Quorum Deep Dive

# Quorum

## Tutorials / useful links

https://www.youtube.com/watch?reload=9&v=YlANBFGy49Q

Deep Dives:

https://hackernoon.com/getting-deep-into-evm-how-ethereum-works-backstage-ac7efa1f0015
https://hackernoon.com/getting-deep-into-geth-why-syncing-ethereum-node-is-slow-1edb04f9dc5

Setup

https://github.com/jpmorganchase/quorum/wiki/Getting-Set-Up


How to use truffle to connect to quorum
https://truffleframework.com/tutorials/building-dapps-for-quorum-private-enterprise-blockchains

## Install for development / debugging

Quorum is a "copy" rather than a "fork" of ethereum. Its because they wouldn't have been able to create a repo called "quorum", it would just have been "ehtereum"

There are a couple of problems with this:

1. Merging to and from upstream is not automatic - have to copy diffs in manually
2. All the packages are still called "ethereum"
3. Its going to be impossible to have geth and quorum in the same workspace

So assuming you have a [go environment](https://ahmadawais.com/install-go-lang-on-macos-with-homebrew/) setup you need to clone quorum into a subfolder called "ethereum/go-ethereum". yeah.



### Initial setup

https://github.com/jpmorganchase/quorum/wiki/Getting-Set-Up

```
cd $GOPATH/src/github.com/ethereum/go-ethereum
make all
make test
```

You may face some issues with the os x firewall

Using quorum

https://github.com/jpmorganchase/quorum/wiki/Using-Quorum


Constellation can only be installed on ubuntu which partly explains (apart from the convenience) why everyone installs test setups in docker with ubuntu as base images.



https://github.com/jpmorganchase/tessera






You need to have go 1.8 installed

Then you should be able to `make all` in this folder and it should compile you a geth binary in `build/bin` which if you run it should tell you something like this:

```
$ build/bin/geth version
Geth
Version: 1.7.2-stable
Git Commit: 0d0c507a5945ab63c8441007f022a143e2418f1e
Quorum Version: 2.1.1
Architecture: amd64
Network Id: 1337
Go Version: go1.8.7
Operating System: darwin
GOPATH=/Users/jmdb/Code/golang
GOROOT=/usr/local/opt/go@1.8/libexec
```

You can see its the quorum build.

You can also try `make test`

So now you should be able to run standard geth stuff.

Because its a copy of ethereum, its called "geth" so we can link to it to call it "quorum" to avoid ambiguity:

```
ln -s ${PWD}/build/bin/geth /usr/local/bin/quorum
quorum version
```

The other program we will need is `bootnode` this generates keys for us

```
ln -s ${PWD}/build/bin/bootnode /usr/local/bin/qbootnode
qbootnode --help
```
Now we need to learn how to configure it. Going to call this `qbootnode` to distinguish it from ethereum.

We can do this in a new directory

```
mkdir -p build/test-net/1-node/node-a
```

In here we need to create the right directory structure:

```
mkdir -p keys logs data/geth

```

When we do more than one node we will have `node-a`, `node-b` etc.

So in here we have a directory where the node will store all its data, a place to put the keys we will generate and somewhere for the logs to go.

So now we need to generate a node keypair for this node. We do this with the `qbootnode` application. So far this is identical to standard ethereum.

[TODO] Q: if you use the standard eth tools to do this, does it still work?

```
qbootnode --genkey data/node-a.key
```

We need the value of the public key of the node for our config:

```
ADDR_NODE_A=$(qbootnode -nodekey data/node-a.key -writeaddress)
echo $ADDR_NODE_A
```

All that is in the node-a.key file is packaged version of the private key. [TODO] Add more details here.

Geth and Quorum can be configured to know what nodes to look for initially via the `static-nodes.json` config file. We create this file in the root of our node dir

```
echo "[" > static-nodes.json
echo "enode://$ADDR_NODE_A@localhost:30303?discport=0" >> static-nodes.json
echo "]" >> static-nodes.json
```

We also need an account. These can be defined later but its also possible to set them up in the genesis file.

```
echo "" >> password.txt
qbootnode --genkey data/node-a.key
ACCOUNT_01_ADDR=$(quorum --datadir=data --password password.txt account new ) | cut -c 11-50
echo $ACCOUNT_O1_ADDR
```

This uses the builtin account creation process which will setup a keystore and create the account for you. It returns the account address (which is an encoding of the public key) [todo] More detail on this + reference https://github.com/Himanshu-Pandey/ethereum-workshop/blob/master/quorum/node/eckeypair.py


```
cat > genesis.json <<EOF
{
  "alloc": {
    "${ACCOUNT_01_ADDR}": {
      "balance": "1000000000000000000000000000"
    }
  },
  "coinbase": "0x0000000000000000000000000000000000000000",
  "config": {
    "homesteadBlock": 0
  },
  "difficulty": "0x0",
  "extraData": "0x",
  "gasLimit": "0x2FEFD800",
  "mixhash": "0x00000000000000000000000000000000000000647572616c65787365646c6578",
  "nonce": "0x0",
  "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "timestamp": "0x00"
}
EOF
```
This is the genesis file - [todo] add more references to this and what everything means.

We now need to create a configuration for the transaction manager `tm.conf`

```
cat > tm.conf <<EOF
url = "http://localhost:9000/"
port = 9000
socketPath = "${PWD}/tm.ipc"
otherNodeUrls = []
publicKeyPath = "${PWD}/keys/tm.pub"
privateKeyPath = "${PWD}/keys/tm.key"
archivalPublicKeyPath = "${PWD}/keys/tma.pub"
archivalPrivateKeyPath = "${PWD}/keys/tma.key"
storagePath = "${PWD}/constellation"
EOF
```

The transaction managers will be running on port 9000





```
git clone git@github.com:jpmorganchase/quorum.git ethereum/go-ethereum
```

## Consensus mechanisms

https://github.com/jpmorganchase/quorum/issues/131

https://github.com/ConsenSys/QuorumNetworkManager/wiki

https://github.com/jpmorganchase/quorum-tools

https://github.com/jpmorganchase/quorum/wiki/ZSL


https://github.com/ethereum/EIPs/issues/225

There are three consensus mechanisms available in quorum:

### QuorumChain

This is the original consensus mechanism that was introduced and involves a kind of leader election which has block makers and block voters which nodes are which is specified in the genesis.json file. There is a [utility](https://github.com/davebryson/quorum-genesis) that helps you make this file.

### RAFT

https://github.com/jpmorganchase/quorum/blob/master/raft/doc.md

This is a leader protocol which decides who is minting blocks, based on whoever the RAFT leader is at the time. The blocks are passed via the raft protocol. Raft has its own port which it will send traffic on.

They use the etcd implementation of RAFT from here https://github.com/etcd-io/etcd

### Istanbul PBFT

https://github.com/ethereum/EIPs/issues/650

https://medium.com/getamis/istanbul-bft-ibft-c2758b7fe6ff

## Tools

https://github.com/synechron-finlabs/quorum-maker/


RPC Port : 22000
Whisper Port : 22001
Constellation Port : 22002
RPC Port : 22003
Quorum Maker Node Manager Port : 22004
Raft Port


## Navigating the code

Key entry points:

### node.go:Start

Starts up all the basic services like the database and p2p networking.

### api.go:SendTransaction

Where it all goes on - here is where you can see it sending a tx - also a major branch from ethereum because it adds in the `privateFor` parameter to the transaction headers.

### p2p/peer:run

This is the p2p initialisation code which puts it into a readloop for receiving and handling peer to peer messages

https://github.com/ethereum/wiki/wiki/RLP

Great article about n/w architecture of geth
[https:/
/www.tutorialdocs.com/article/ethereum-network-architecture.html](https://www.tutorialdocs.com/article/ethereum-network-architecture.html)

https://wiki.parity.io/Light-Ethereum-Subprotocol-(LES)
