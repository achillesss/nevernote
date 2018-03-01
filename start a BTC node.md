## Start a ``BTC`` node

下面介绍两种BTC节点客户端：
1. ``Bitcoin Core``
2. ``btcd`` & ``btcwallet``

### ``Bitcoin Core``

``Bitcoin Core`` 是使用 ``C++`` 编写的比特币全节点

#### 安装

- MacOS

	```bash
	// 安装 xcode 命令行工具
	xcode-select --install
	
	// 安装 homebrew
	/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
	
	// 安装依赖
	brew install automake berkeley-db4 libtool boost --c++11 miniupnpc openssl pkg-config protobuf python3 qt libevent
	
	// 下载源码，从源码安装
	git clone https://github.com/bitcoin/bitcoin
	cd bitcoin
	./autogen.sh
	./configure
	make
	make check
	```

#### 配置

``Bitcoin Core`` 的配置文件位置默认为：
``$HOME/Library/Application Support/Bitcoin/bitcoin.conf``

主要配置如下：

```bash
# rpc服务用户名及密码
# rpcuser=user
# rpcpassword=password
	
# 使用测试网络
# testnet=1

# 允许RPC服务接收命令行及JSON-RPC命令
# server=1
	
# 将全部的transaction进行索引，允许调用'getrawtransaction'来查询所有transaction
# txindex=1
	
# 可能还需要重建现有链的状态
# reindex-chainstate=1
	
# 可能需要重建所有区块链的状态（从头开始）
# reindex=1
	
```

#### 运行

切换到``bitcoin``项目目录下，通过如下命令运行：
``./src/bitcoind``

如果有已知节点``address:port``需要在运行时添加，运行命令``./srcbitcoind -addnode address:port``

#### 客户端连接范例

```go
connCfg := &rpcclient.ConnConfig{
		Host:         "localhost:8332",
		User:         "yourrpcuser",
		Pass:         "yourrpcpass",
		HTTPPostMode: true, // Bitcoin core only supports HTTP POST mode
		DisableTLS:   true, // Bitcoin core does not provide TLS by default
}

// Notice the notification parameter is nil since notifications are
// not supported in HTTP POST mode.
client, err := rpcclient.New(connCfg, nil)
if err != nil {
	log.Fatal(err)
}
defer client.Shutdown()
```

### ``btcd`` & ``btcwallet``

``btcd`` 是使用 ``go`` 编写的比特币全节点，``btcwallet`` 是使用 ``go`` 编写的比特币钱包应用。运行 ``btcd`` 之后，只有一个全节点，如果需要使用钱包相关功能（创建钱包、发送交易），则需要运行 ``btcwallet``，并将 ``btcwallet`` 与 ``btcd`` 连接起来。

#### 安装

- MacOS

	```bash
	// 安装 glide
	go get -u github.com/Masterminds/glide
	
	// 获取源码
	git clone https://github.com/btcsuite $GOPATH/src/github.com/btcsuite
	
	// 安装 btcd
	cd $GOPATH/src/github.com/btcsuite/btcd
	glide install
	go install . ./cmd/...
	
	// 安装 btcwallet
	cd $GOPATH/src/github.com/btcsuite/btcwallet
	glide install
	go install . ./cmd/...
	
	```

#### 配置

- ``btcd``

	``btcd`` 默认配置为：
	``$HOME/Library/Application Support/Btcd/btcd.conf``
	
	主要配置如下：
	
	```bash
	[Application Option]
	# rpc 用户及密码，此用户名及密码会在钱包中使用
	# rpcuser=user
	# rpcpass=password
	
	# 使用测试网络
	# testnet=1
	
	# rpc 监听端口
	# rpclisten=127.0.0.1:18333
	
	# 索引全部transactions，并能使用'getrawtransaction'来查询任何transaction
	# txindex=1
	```
	

- ``btcwallet``

	``btcwallet`` 默认配置为：
	``$HOME/Library/Application Support/Btcwallet/btcwallet.conf``
	
	主要配置如下：
	
	```bash
	[Application Option]
	# 连接至 'btcd' 的用户名及密码，务必与 'btcd.conf' 中的用户名及密码一致
	# username=user
	# password=password
	
	# rpc监听端口
	# rpclisten=127.0.0.1:18334
	
	# 使用测试网络
	# testnet=1
	
	# 连接到 'btcd' 的地址
	# rpcconnect=localhost:18333
	```
	
#### 运行

- ``btcd``
	使用命令 ``$ btcd`` 运行 ``btcd``

	如果有已知节点``address:port``需要在启动时连接，使用命令 ``$btcd --addpeer address:port``
	
- ``btcwallet``

	首次使用钱包时，必须创建一个钱包：
	
	``$ btcwallet --create``
	
	接下来会提示创建 ``private passphrase for new wallet``
	
	然后提示是否对数据进行包装加密
	
	然后会提示创建 ``public passphrase for new wallet``
	
	然后询问是否有钱包种子，这里填 ``no``
	
	然后会将生成的钱包种子打印到屏幕，注意保存
	
	最后输入 ``OK``，按下回车，结束钱包的创建
	
	创建完钱包之后，就能运行钱包了。使用刚刚第二次创建的 ``public passphrase for new wallet``，运行命令：
	
	``$ btcwallet --walletpass $YOUR_PUBLIC_PASSPHRASE``
	
	即可运行钱包了
	
#### 客户端连接范例

此处需要使用证书使用``TLS``连接钱包节点

```go
// Connect to local btcwallet RPC server using websockets.
certHomeDir := btcutil.AppDataDir("btcwallet", false)
certs, err := ioutil.ReadFile(filepath.Join(certHomeDir, "rpc.cert"))
if err != nil {
	log.Fatal(err)
}

connCfg := &rpcclient.ConnConfig{
	Host:         "localhost:18332",
	Endpoint:     "ws",
	User:         "yourrpcuser",
	Pass:         "yourrpcpass",
	Certificates: certs,
}

client, err := rpcclient.New(connCfg, &ntfnHandlers)
if err != nil {
	log.Fatal(err)
}
```


### 资料

####[Bitcoin Core](https://github.com/bitcoin/bitcoin)

####[Bitcoin Core APIs](https://en.bitcoin.it/wiki/Original_Bitcoin_client/API_calls_list)

####[btcd](https://github.com/btcsuite/btcd)

####[btcwallet](https://github.com/btcsuite/btcwallet)

####[btcd rpcclient](https://github.com/btcsuite/btcd/tree/master/rpcclient)

####[btcd连接范例](https://github.com/btcsuite/btcd/tree/master/rpcclient/examples)
	