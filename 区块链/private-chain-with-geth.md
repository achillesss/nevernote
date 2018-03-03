### ``Geth`` 安装

- MacOS:

	```bash
	brew tap ethereum/ethereum
	brew install ethereum
	```

- Ubuntu:

	```bash
	sudo apt-get install software-properties-common
	sudo add-apt-repository -y ppa:ethereum/ethereum
	sudo apt-get update
	sudo apt-get install ethereum
	```
	
- [Windows](https://geth.ethereum.org/downloads/)

- SourceCode:

	```bash
	git clone https://github.com/ethereum/go-ethereum
	brew install go
	cd go-ethereum
	make geth
	```
	
### 节点初始化

使用创世区块初始化节点：``geth init genesis.json --datadir ~/eth/private``

- 创世区块(``genesis.json``)

	```json
	{
	    "config": {
	        "chainId": 123456,
	        "homesteadBlock": 0,
	        "eip155Block": 0,
	        "eip158Block": 0,
	        "eip160Block": 0
	    },
	    "nonce": "0x0000000000000042",
	    "timestamp": "0x0",
	    "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
	    "extraData": "0x",
	    "gasLimit": "0x8000000",
	    "difficulty": "0x400",
	    "mixhash": "0x0000000000000000000000000000000000000000000000000000000000000000",
	    "coinbase": "0x0000000000000000000000000000000000000000",
	    "alloc": {
	        "0000000000000000000000000000000000000001": {
	            "balance": "1"
	        },
	        "0000000000000000000000000000000000000002": {
	            "balance": "1"
	        },
	        "0000000000000000000000000000000000000003": {
	            "balance": "1"
	        },
	        "0000000000000000000000000000000000000004": {
	            "balance": "1"
	        }
	    }
	}
	```
	说明：
	1. ``chainId`` 链标识，拥有标识相同的链的节点才能相互连接
	2. ``difficulty`` 挖矿初始难度，默认为 ``b1024`` 即 ``0x400``，越小越好挖矿
	3. 其他参数在``geth``创世区块中虽然不太重要，但也不要随意改动

	
### 启动私有链

初始化完毕之后，可以直接启动节点了：``geth --rpcapi "db,net,web3,personal,admin,eth" --datadir ~/eth/private --networkid 123456 console 2>> ~/eth/private/out.log``

一些 ``flag``:

- ``--rpcapi`` 设置哪些API被允许通过RPC打开，默认情况下开启``web3,net,eth``
- ``--rpcport`` HTTP-RPC 监听端口，默认 ``8545``
-  ``--rpcaddr`` HTTP-RPC 监听地址，默认 ``localhost``
-  ``--rpccorsdomain`` 允许执行RPC的地址，``,``分隔，``*``代表所有
-  ``console`` 开启javascript控制台
-  ``networkid`` 网络标识，同一网络标识下的节点才能互联，最好与创世区块中的 ``chainId`` 一致


### 在私有链上运行ETH钱包

运行钱包中的程序，并设置rpc接口所要使用的节点``.ipc``文件

- MacOS: ``/Applications/Ethereum\ Wallet.app/Contents/MacOS/Ethereum\ Wallet --rpc ~/Library/eth1/geth.ipc``
- Linux: ``ethereumwallet --rpc ~/eth/private/geth.ipc``
- Windows: ``"Ethereum Wallet" --rpc \\.\pipe\geth.ipc``


### 控制台命令

- 查看本节点和哪些节点互联：``admin.peers``
- 查看本节点的``enode``: ``admin.nodeInfo.enode``
- 连接其他节点``enode``: ``admin.addPeer(enode)``
- 查看本节点所拥有的账户: ``eth.accounts`` 或者 ``personal.listAccounts``
- 在本节点创建新账户: ``personal.newAccount(password)``
- 查询某账户的 ``eth``: ``eth.getBalance(account)``
- 发起交易: 
	1. 解锁账户: ``personal.unlockAccount(account0)``
	2. 发起交易: ``eth.sendTransaction({from:account0,to:account1,value:web3.toWei(1,"ether")})``
	3. 挖矿确认交易: ``miner.start()``
	4. 确认完毕后停止交易: ``miner.stop()``
	5. 3+4可以写成: ``miner.start();admin.sleep(5);miner.stop()`` 挖5秒钟的矿
