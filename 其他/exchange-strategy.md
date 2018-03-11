# 交易策略

## 被动挂单策略

此策略非常简单，但效率极其低下。

假设有交易对eth_btc：

- ``ve``表示eth数量
- ``vb``表示btc数量
- ``peb``表示eth对btc价格
- ``pbe``表示btc对eth价格
- ``c``表示交易所的手续费费率

其中：

$$pbe \times peb=1$$

并且做如下约定：

- 所有卖单(ask)中的参数为``ve``和``peb``。意为“我需要在``peb``价位卖出``ve``个eth以获取btc”。
- 所有买单(bid)中的参数为``vb``和``pbd``。意为“我需要在``1/pbe``价位以``vb``个btc买入eth”，即“我需要在``pbe``价位卖出``vb``个btc以获取eth”。

可以看出来，实际上卖单即卖单，卖单即买单。


### 从一个卖单开始

假设我们有一卖单：

$$volume_{eth} = ve = v_{0}$$

$$price_{eth} = peb = p_{0}$$

#### 不收手续费的交易所

在该交易所中：

$$vb = amout_{btc} = volume_{eth} \times price_{eth} = v_{0}p_{0}$$

此时如果我们将卖出eth得到的btc在我们出售eth的价位将eth全部买回来，那么我们是不亏不赚的，如果我们想赚的更多的eth，则需要在刚刚卖出eth的价位之下买入eth。所以：

我们至少有这样一个买单：

我们拿出所有的btc：

$$volume_{btc} = vb = v_{0}p_{0}$$

至少在这个价格：

$$price_{btc} = pbe = {1 \over p_{0}}$$

能够买入eth的数量为：

$$ve = amount_{btc} = volume_{btc} \times price_{btc} = v_{0}$$

现在我们有了两笔交易：

| ordertype | volume       | $price_{eth}$ | amount       |
| --------- | -----------: | ------------: | -----------: |
| ask       | $v_{0}$      | $p_{0}$       | $v_{0}p_{0}$ |
| bid       | $v_{0}p_{0}$ | $p_{0}$       | $v_{0}$      |

#### 收手续费的交易所

实际上，现在交易所都是收手续费的，于是，我们在出售订单之后，实际拿到的btc数量为：

$$vb = amout_{btc} = volume_{eth} \times price_{eth} \times (1-c)=  v_{0}p_{0}(1-c)$$

于是：

$$volume_{btc} = vb = v_{0}p_{0}(1-c)$$

同时我们期望后面我们至少能买到的eth为：

$$volume_{eth} = ve = v_{0}$$

所以实际上我们在下买单的时候，价格可能为：

$$price_{eth} = {volume_{btc} \over volume_{eth}} = p_{0}(1-c) $$

但实际上，我们的买单中依然会被交易所收取一部分的交易费！如果接下来我们还要继续下卖单，那么可以在第二个卖单中将第一个买单的手续费支出计算在内；但是如果接下来我们并不想下单了呢？所以，我们在下第一个买单的时候，为了不不因其他意外导致亏损，我们的策略应该是：在本次买单中将本次交易的手续费也代入计算收益：

$$volume_{eth} = {ve \over (1-c)} = {v_{0} \over (1-c)}$$

$$price_{eth} = {volume_{btc} \over volume_{eth}} = p_{0}(1-c)^{2} $$

这样并没有结束，因为我们发现：买卖单中的公式并不对称。从本质上来说，不论是买单还是卖单，实际上是一个换汇过程，并没有明显的买卖区别。所以，买卖单的计算公式应该是严格对称的。于是，我们在一开始挂卖单的时候，将价格稍作调整，让本单将交易费直接代入计算：

出售时：

$$volume_{eth} = ve = v_{0}$$

$$price_{ask\_eth} = {peb \over (1-c)} = {p_{0} \over (1-c)}$$

$$volume_{btc} = volume_{eth} \times price_{ask\_eth} \times (1-c)= v_{0}p_{0}$$

购买时：

$$volume_{btc} = volume_{eth} \times price_{ask\_eth} \times (1-c)= v_{0}p_{0}$$

$$price_{bid\_eth} = peb \times (1-c) = p_{0}(1-c)$$

$$volume_{eth} =  {volume_{btc} \over price_{bid\_eth}} \times (1-c) = v_{0}$$

这样，我们就有了如下数据：

| ordertype | volume       | $price_{eth\_expect}$ | $price_{eth\_actual}$ | charge | amount       |
| --------- | ------------ | :-------------------: | :-------------------: | :----: | :----------: |
| ask       | $v_{0}$      | $p_{0}$               | ${p_{0} \over (1-c)}$ | c      | $v_{0}p_{0}$ |
| bid       | $v_{0}p_{0}$ | $p_{0}$               | $p_{0}(1-c)$          | c      | $v_{0}$      |

再将数据简化一下，便有了：

| ordertype | $volume_{a}$ | $price_{a\_expect}$ | $price_{b\_expect}$ | $price_{a\_actual}$ | $price_{b\_actual}$ | charge | $volume_{b}$  |
| --------- | ------------ | :-----------------: | :-----------------: | :-----------------: | :-----------------: | :----: | :-----------: |
| $ask_{a}$ | $v$          | $p$                 | $1 \over p$         | ${p \over (1-c)}$   | ${(1-c) \over p}$   | c      | $vp$          |
| $bid_{a}$ | $v$          | $p$                 | $1 \over p$         | $p(1-c)$            | ${1 \over p(1-c)}$  | c      | ${v \over p}$ |

即：在货币A和货币B的交易对中：

令``price_{xy}``表示每个货币y价值多少x个货币，即y对x的价格

出售A公式：

$$price_{B\_A\_actual} = {price_{B\_A} \over (1-c)}$$

$$volume_{B} = volume_{A} \times price_{B\_A\_actual} \times (1-c) = volume_{A} \times ({price_{B\_A} \over (1-c)}) \times (1-c)$$

购买A公式即出售B公式：
$$price_{A\_B\_actual} = {price_{A\_B} \over (1-c)}$$

$$price_{B\_A\_actual} = {1 \over price_{A\_B\_actual}} = {(1-c) \times price_{B\_A}}$$

$$volume_{A} = volume_{B} \times price_{A\_B\_actual} \times (1-c) =  {volume_{B} \times (1-c) \over \{price_{B\_A} \times (1-c)\}}$$

所以，买卖数据表变成了：

| ordertype | $volume_{out}$ | $price_{expect}$ | $price_{actual}$  | charge | $volume_{b}$  |
| --------- | -------------- | :--------------: | :---------------: | :----: | :-----------: |
| $ask_{a}$ | $v$            | $p$              | ${p \over (1-c)}$ | c      | $vp$          |
| $bid_{a}$ | $v$            | $p$              | $p(1-c)$          | c      | ${v \over p}$ |

#### 获取收益

投资实际上还是为了收益，如果向刚刚以上的挂单方式交易，我们还没有拥有正收益。

假设我们每一单的期望**收益率**是：

- `r`

将期望收益率添加进表：

| ordertype | $volume_{out}$ | $price_{expect}$ | $price_{actual}$  | charge | profit rate | $volume_{in}$ |
| --------- | -------------- | :--------------: | :---------------: | :----: | ----------: | :-----------: |
| $ask_{a}$ | $v$            | $p$              | ${p \over (1-c)}$ | c      | 0.0         | $vp$          |
| $bid_{a}$ | $v$            | $p$              | $p(1-c)$          | c      | 0.0         | ${v \over p}$ |

期望收益为零时，卖单公式为：

$$volume_{in} = volume_{out} \times price_{actual} \times (1-c)$$

期望为 ``r`` 时，我们需要调整我们的价格到一个合适的价格：

$$price_{profit} = price_{actual} \times (1+r)$$

于是，卖单公式变成了：

$$volume_{in} = volume_{out} \times price_{profit} \times (1-c) = volume_{out} \times price_{actual} \times (1+r) \times (1-c) = v({(1+r) \over (1-c)}p)(1-c)$$

如果是买单，则：

$$price_{profit} = {price_{actual} \over (1+r)}$$

买单公式变成了：

$$volume_{in} = {volume_{out} \over price_{profit}} \times (1-c) = {v \over ({p(1-c) \over (1+r)})}(1-c)$$

所以，买卖单数据表变成了：

| ordertype | $volume_{out}$ | $price_{expect}$ | charge | profit rate | $price_{profit}$     | $volume_{in}$      |
| --------- | -------------- | :--------------: | :----: | :---------: | :------------------: | :----------------: |
| $ask_{a}$ | $v$            | $p$              | c      | r           | ${(1+r)\over(1-c)}p$ | $vp(1+r)$          |
| $bid_{a}$ | $v$            | $p$              | c      | r           | $p(1-c)\over(1+r)$   | ${v \over p}(1+r)$ |

至此，挂单策略就很明显了，不再赘述。

## 其他策略

待添加