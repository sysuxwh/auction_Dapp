# 拍卖市场_以太坊Dapp

---

[TOC]

## 一、初始化项目环境  

### （一）truffle框架介绍及安装  
Truffle框架就是一个帮助书写编译和发布基于Solidity的智能合约的工具。Truffle具有以下优点：

 - 首先对客户端做了深度集成。开发，测试，部署一行命令都可以搞定。不用再记那么多环境地址，繁重的配置更改，及记住诸多的命令。
 - 它提供了一套类似`maven`或`gradle`这样的项目构建机制，能自动生成相关目录，默认是基于Web的。
 - 提供了合约抽象接口，可以直接通过`var meta = MetaCoin.deployed();`拿到合约对象后，在Javascript中直接操作对应的合约函数。原理是使用了基于`web3.js`封装的`Ether Pudding`工具包。简化开发流程。
 - 提供了控制台，使用框架构建后，可以直接在命令行调用输出结果，可极大方便开发调试。
 - 提供了监控合约，配置变化的自动发布，部署流程。不用每个修改后都重走整个流程。


**NodeJS**  
首先，Truffle需要依赖NodeJS。Windows用户如果没安装的话可以访问[NodeJS官网][1]下载。安装成功后在命令行输入`node -v`和`npm -v`检查是否安装成功。

**Truffle**  
以管理员身份打开powershell，输入`npm install -g truffle`安装truffle（图中为VSCode的命令行）。  
![安装truffle][2]

> 如果你是Windows用户，那就需要注意：Windows 系统有个命名问题，它会让我们在执行 Truffle 命令的时候只打开配置文件"truffle.js"，而不会读取里面的内容。解决方法是。修改Windows的PATHEXT环境变量，去掉.js后缀，避免以后在Truffle目录下运行"truffle"命令可能遇到的麻烦。

### （二）**创建项目目录**  
truffle提供了很多项目模板，即是[truffle box][3]，可以快速搭建一个去中心化应用的代码骨架。简单来说，`truffle box`是将solidity智能合约、相关库、前端框架都集成在一起的集合，方便开发人员在最大程度上简化不必要的环境搭建和技术选型工作。 

创建一个项目目录，然后进行初始化。先利用`truffle box`的`webpack`模板来配置拍卖市场。  
```
$ mkdir auctionDapp
$ cd auctionDapp
$ truffle unbox webpack
```

可是在配置的过程中发生了错误，在Stackoverflow上找到了同样问题以及解决方案：[Error in unboxing truffle-react on Windows][4]  
输入以下两条命令后，耐心等待即可。  
```
npm install --global--production windows-build-tools  
npm install --global node-gyp
```  
![solution][5]  
这次再重建项目目录，就没问题了。  
![webpack配置][6]  
```
|--app          // 前端设计，包括html,css,js
|--build        // 智能合约编译后文件存储路径
|--contracts    // 智能合约文件存储路径
|--migrations   // 存放发布脚本文件
|--node_modules // 相关nodejs库文件
|--test         // 合约测试文件存放路径
|--box-img-lg.png
|--box-img-sm.png
|--LICENSE
|--package-lock.json
|--package.json
|--truffle.js
|--webpack.config.js
```
删掉contracts目录下用于测试的ConvertLib.sol,MetaCoin.sol，避免干扰。

---

## 二、编写智能合约   
### （一）维克里拍卖原理  
拍卖是财产权利转让的最古老方式之一，常见的拍卖或竞标方式有下列四种：英国式拍卖法；荷兰式拍卖法；最高价得标拍；最高价得标、次高价付款拍卖法（维克里拍卖法）。比较传统、常见的方法是英国式拍卖法，即是竞标者出价由下往上喊，喊价最高者得标，竞标者可多次重复提高出价。

这次我选择实现的是维克里拍卖。**维克里拍卖（Vickrey auction）**，即次价密封投标拍卖(Second-price sealed-bid auction)。投标者在不知道其他人标价的情况下递出标单，标价最高的人得标，但只需付次高的标价。

从理论上来讲，第二价格密封拍卖是一种有效的拍卖机制。因为此时，每个投标者的最优战略就是依照自己对标的物的估价据实竞标，这显然是一种符合激励相容原则的交易方式。而且，由于拍卖品最终由支付意愿最高的投标者获得，它也是一种能使买卖双方达到帕累托最优的配置机制。关于更多关于维克里拍卖的信息可出门左转谷歌。  

### （二）维克里拍卖法的智能合约实现  
在contrasts目录下，新建文件`AuctionStore.sol`，在这里编写合约。对于初学者，十分建议通过[CryptoZombies][7]学习。CryptoZombies是个在编游戏的过程中学习Solidity智能协议语言的互动教程，对初学者十分地友好。完成前3-4章的学习后，就能大致地编写一个简单地智能合约了。  

拍卖步骤大致分为：

 - 卖家发布商品；
 - 拍卖期间，用户可以进行投标，需要提供 出价 以及 竞拍密匙(保密)，支付相应的出价金额；
 - 拍卖结束后，各个竞标者提供 竞拍密匙，进行公告；
 - 全部公告后，确认最高竞价者，并返还ta（最高价-次高价），这意味着最高竞价者只需要支付次高价即可，其他竞价者将返回与出价相应的金额。


#### **1. 定义结构体 & 状态变量**  
首先定义最主要的商品信息结构体Product。这个结构体用于存储所有于其相关的信息，包括它的商品id、名字、商品分类、拍卖状态、描述图片、描述文字等，具体可看下面代码。  

Product结构体有几个属性需要注意。一是商品状态，通过枚举型定义，商品状态共有3个，分别是`AVAILABLE` - 商品可竞标，`SOLD` - 商品已结束拍卖&已售出，`UNSOLD` - 商品已结束拍卖。在Solidity中，当enum类型的枚举数不够多时，它默认的类型为uint8，即是说这三个状态其实是uint8类型的0、1、2。  

二是商品图片哈希imageHash和商品描述哈希descriptionHash。这里打算使用哈希值用于存储，之后会将商品图像和商品描述(可能文本会较大)上传至IPFS，并将上传文件的散列哈希存储到区块链中。  

三是竞标者信息的映射bidders。这个映射存储了该商品所有竞标者的信息，以竞标者的地址为key，以一个结构体Bidder为value。这个Bidder存储了 竞标者的地址 & 竞拍密匙 & 出价 & 是否已揭标，这些属性在之后都会使用到。  

除了结构体，还需要用到几个状态变量。状态变量是被永久地保存在合约中。也就是说它们被写入以太币区块链中. 想象成写入一个数据库。

 - productCount， uint类型，统计商品的数量
 - productSaler， 一个映射，从 商品id 映射到 卖家地址（以暂时实现的功能来说，似乎没啥用）
 - store，        一个映射，从 商品id 映射到 商品对象

```
    // 商品信息
    struct Product{
        uint id;                    // 商品ID

        string name;                // 商品名字
        string classification;      // 商品分类
        Status status;              // 商品状态
        string imageHash;           // 商品图片哈希
        string descriptionHash;     // 商品描述哈希

        uint startTime;             // 竞拍开始时间
        uint endTime;               // 竞拍结束时间
        uint initPrice;             // 竞拍起始价格
        address buyer;              // 出价最高的投标者地址
        uint topPrice;              // 出价最高价格
        uint secondPrice;           // 出价次高价格
        uint bidNum;                // 竞标者人数
        uint revealNum;             // 揭标者人数
        mapping(address => Bidder) bidders;    // 竞标者信息
    }
    
    // 商品状态 - 用于描述商品信息
    enum Status {
        AVAILABLE,  // 拍卖开始，可竞标
        SOLD,       // 拍卖结束，已售出
        UNSOLD      // 拍卖结束，未售出
    }

    // 投标者信息
    struct Bidder{
        address addr;       // 竞标人地址
        bytes32 pwd;        // 竞拍密匙
        uint value;         // 出价
        bool revealed;      // 是否已经揭标
    }

    uint productCount;  // 统计商品的数量
    mapping(uint => address) productSaler;              // 商品id -> 卖家地址
    mapping(uint => Product) store;                     // 商品id -> 商品对象

    // 构造函数
    constructor() public {
        productCount = 0;
    }
```

#### **2. 发布商品 addProduct()**  

> **msg.sender：** 在 Solidity 中，有一些全局变量可以被所有函数调用。 其中一个就是 msg.sender，它指的是当前调用者（或智能合约）的address，可以在调用函数时通过from参数传入。
**storage & memory：** 在 Solidity 中，有两个地方可以存储变量 。`Storage` 变量是指永久存储在区块链中的变量，使用storage(存储)是相当昂贵的，“写入”操作尤其贵，随着区块链的增长，拷贝份数更多，存储量也就越大。 `Memory` 变量则是临时的，当外部函数对某合约调用完成时，内存型变量即被移除，可以节省很多gas。

我们可以以不同的msg.sender身份来发布商品。发布商品时，要提供相应的商品信息，并需要确保商品拍卖开始时间小于结束时间。然后更新商品数量productCount，使用productCount作为新商品的id，并更新store和productSaler映射。


```
    // 发布商品
    function addProduct(string _name, string _class, string _imageHash, string _descHash, uint _start, uint _end, uint _initPrice) public {
        require(_start < _end, "竞拍开始时间需要小于结束时间");
        productCount++;
        Product memory product = 
            Product(productCount, _name, _class, Status.AVAILABLE, _imageHash, _descHash, _start, _end, _initPrice, 0, 0, 0, 0, 0);
        store[productCount] = product;
        productSaler[productCount] = msg.sender;
    }
```

#### **3. 投标 bidProduct()**  
当商品发布后，用户可以通过指定商品id和竞拍密匙来投标，投标时需要先支付与出价同样的以太币。竞拍密匙用于投标和揭标，揭标时需要提供相同密匙才可以揭标。投标时，需要进行以下判断：

 - 当前时间 > 竞拍开始时间
 - 当前时间 < 竞拍结束时间
 - 出价 > 商品初始价格
 - 未参与过投标（只允许投标一次） 

以上条件都满足后，判定为投标成功，该商品增加这位投标人信息（记录地址、密匙、出价、揭标状态）。

> **关于竞拍密匙的存储：** Ethereum 内部有一个散列函数keccak256，它用了SHA3版本。一个散列函数基本上就是把一个字符串转换为一个256位的16进制数字。字符串的一个微小变化会引起散列数据极大变化。这里使用该函数存储密匙。
**payable 修饰符：** 它是一种可以接收以太的特殊函数。如果一个函数没标记为payable， 而你尝试利用上面的方法发送以太，函数将拒绝你的事务。
**msg.value：** 合约调用方附带的以太币，在这里就是投标者的出资。
 
```
    // 投标
    function bidProduct(uint _productID, string _password) public payable returns(bool) {
        Product storage product = store[_productID];
        // 对竞拍密匙进行转换
        bytes32 sealedBid = keccak256(bytes(_password));

        require(block.timestamp >= product.startTime, "该商品尚未开始竞拍");
        require(block.timestamp <= product.endTime, "该商品已结束竞拍" );
        require(msg.value >= product.initPrice, "投标的虚拟价格不能低于最低价格");
        require(product.bidders[msg.sender].addr == 0, "您已经参与过竞拍");

        product.bidders[msg.sender] = Bidder(msg.sender, sealedBid, msg.value, false);
        product.bidNum++;
        return true;
    }
```

#### **4. 揭标/公布 revealBid()**  
当拍卖结束后，所有投标者可以揭标，全部人揭标后，即得出最终成交者。投标前，需要进行以下判断：

 - 当前时间 > 竞拍结束时间
 - 该用户参与了该商品的投标
 - 该用户的竞拍密匙输入正确
 - 该用户尚未揭标

揭标时一共分为以下几种情况：

 - 第一个揭标：暂时将该用户标为最高竞价者
 - 不是第一个揭标，且出价高于当前最高价：将原最高价返还给原最高竞价者，该用户成为新的最高竞价者；
 - 不是第一个揭标，且出价介于当前最高价和次高价之间：修改次高价，返还该用户的出资；
 - 不是第一个揭标，且出价低于当前次高价：返还该用户的出资；

```
    // 揭标
    function revealBid(uint _productID, string _password) public {
        require(block.timestamp > product.endTime, "该商品仍在竞标，请等待");
        
        // 对竞拍密匙进行转换
        bytes32 sealedBid = keccak256(bytes(_password));
        Product storage product = store[_productID];
        Bidder memory bidder = product.bidders[msg.sender];
        
        require(bidder.addr > 0, "您未参与该商品的竞标");
        require(bidder.pwd == sealedBid, "密码错误");
        require(bidder.revealed == false, "您已揭标");

        uint refund;
        uint price = bidder.value;

        if (product.buyer == 0) {
            // 第一个揭标的人
            product.buyer = msg.sender;
            product.topPrice = price;
            product.secondPrice = product.initPrice;
            refund = 0;    // 成为最高竞价者，暂不退款
        } else {
            // 此时已有他人揭标，即存在最高价
            if (price > product.topPrice) {                
                product.buyer.transfer(product.topPrice);    // 退钱给原最高竞价者
                product.buyer = msg.sender;                     // 修改最高竞价者
                product.secondPrice = product.topPrice;         // 修改次高价
                product.topPrice = price;                       // 修改最高价
                refund = 0;                                     // 成为最高竞价者，暂不退款
                
            } else if (price > product.secondPrice) {
                product.secondPrice = price;                // 修改次高价
                refund = price;                             // out，退还全款
            } else {
                refund = price;                             // out，退还全款
            }
        }

        if (refund > 0) {
            msg.sender.transfer(refund);
        }
        product.bidders[msg.sender].revealed = true;
        product.revealNum++;
    }
```

#### **5. 完成交易 endAuction()**  
所有人揭标后，可以得出该商品的成交对象，由于ta当前出价最高，但他只需要支付第二高的出价，因此需要返还ta差价。成交后，更改商品状态（已出售/没有出售）。此前需要进行如下判断：

 - 当前时间 > 拍卖结束时间
 - 商品仍出于出售状态
 - 参与该商品竞拍的投标者均已揭标

```
    // 所有人揭标结束后，拍卖结束，完成交易
    function endAuction(uint _productID) public {
        Product storage product = store[_productID];
        require(block.timestamp > product.endTime, "该商品仍在竞标，请等待");
        require(product.status == Status.AVAILABLE, "该商品已结束交易");
        require(product.revealNum == product.bidNum, "公告尚未结束");
        if (product.buyer > 0) {
            product.buyer.transfer(product.topPrice - product.secondPrice); 
            product.status = Status.SOLD;
        } else {
            product.status = Status.UNSOLD;
        }        
    }
```

#### **6. 帮助方法** 

> **“view” 函数不花费 “gas”**：当玩家从外部调用一个view函数，是不需要支付一分 gas 的。这是因为 view 函数不会真正改变区块链上的任何数据 - 它们只是读取。

编写一些帮助方法，用于获取相关信息。
```
    // 获取商品信息
    function getProduct(uint _productID) public view returns(uint, string, string, Status, string, string, uint, uint, uint) {
        Product memory product = store[_productID];
        return (product.id, product.name, product.classification, product.status, 
            product.imageHash, product.descriptionHash, product.startTime, product.endTime, product.initPrice);
    }


    // 获取某件商品的投标人数量
    function getBidNum(uint _productID) public view returns(uint count){
        Product memory product = store[_productID];
        count = product.bidNum;
    }

    // 获取某件商品的最高竞价者
    function getBuyer(uint _productID) public view returns(address buyer, uint askingPrice, uint paidPrice){
        Product memory product = store[_productID];
        require(product.status != Status.AVAILABLE, "该商品尚未结束交易");
        buyer = product.buyer;
        askingPrice = product.topPrice;
        paidPrice = product.secondPrice;
    }

    // 获取当前商品数量
    function getProductNum() public view returns(uint){
        return productCount;
    }
```

---

## 三、测试结果  

智能合约必须要部署到链上进行测试。可以选择部署到一些公共的测试链比如Rinkeby或者Ropsten上，缺点是部署和测试时间比较长，显然对于我们的项目来说是不太现实的。还有一种方式就是部署到私链上，Truffle官方推荐使用以下两种客户端：

 - Ganache 
 - truffle develop

Ganache本质上是一个本地ethereum节点仿真器，分为GUI版本和命令行版本。喜欢GUI的可以安装[GUI_Ganache][8]，CLI版本则可以通过`sudo npm install -g ganache-cli`安装。  

如果对GUI没有要求的话，其实个人更推荐使用truffle develop，可以免去安装步骤。它是truffle内置的客户端，跟命令行版本的Ganache基本类似。唯一要注意的是在truffle develop里执行truffle命令的时候需要省略前面的`truffle`，比如`truffle compile`只需要敲`compile`就可以了。

我们选择的是`truffle develop`。VSCode下Ctrl+\` 打开命令行，输入 `truffle compile`，编译合约。`compile`命令会将我们的 Solidity 代码编译为字节码（以太坊虚拟机（EVM）能够识别并执行的代码）。如果编译出现了warning最好解决一下，因为在以太坊中，智能合约一旦部署之后，就再也无法改变源码，因此最好谨慎地对待代码。编译成功后如下图所示：  
![compile][9]  

接下来是部署合约。输入命令`truffle migrate`，出现报错，如图所示。  
![migrate_error][10]  
`migrate`命令会将代码部署到区块链上。出现上图的错误是因为没有指定网络，如使用命令`truffle migrate --network ourTestNet`指定部署到私链ourTestNet中。现在我们只需要有一个用于测试的网络就好了，也就是上面所提到过的，使用`truffle develop`。输入该命令，启动测试终端。  
![develop][11]
由图可见，`truffle develop`在 https://127.0.0.1:9545 端口启动，启动时会给用户生成测试账号，在默认情况下这些测试账号都有 100 个以太币，并且这些以太币都会处于解锁状态，能让我们自由发送它们的以太币。  

然后这次可以部署合约了，注意在这里执行truffle命令的时候需要省略前面的`truffle`。  
![migrate_develop][12]

部署成功了，接下来开始测试。当然你可以使用Solidity智能合约版本的单元测试，单元测试智能合约存放在test目录下。一般来讲，这种文件的命名规则是Test加待测智能合约的名字拼串组成。但是为了更直观、逐步地看出拍卖过程和信息地变化，这里不使用单元测试，而是直接获取合约实例，逐步地调用各函数。  

执行命令`AuctionStore.deployed().then(instance => {auctionInstance = instance})`。这行命令会将 truffle 部署的合约的实例的一个引用赋给 auctionInstance 变量，方便之后调用函数。

我们假设一个场景：用户0（默认的msg.sender）发布了一个商品——iPhoneX，有用户1-4进行投标。先查看一下4个投标者的初始余额，均为100以太币。 

![getInstance & check the bidders][13]

设置一些变量，包括拍卖起始时间、拍卖结束时间、商品起始价格。这里先获取当前时间，拍卖在3分钟（180s）后开始；拍卖持续5分钟（300s）；商品初始价格为2个以太币。

![set the var][14]

用户0开始发布商品iPhone。这里因为测试的缘故，描述图片和描述文字的哈希就没必要放进去了，留到实现前端的时候完善。由图可见，发布该商品花费了 258812 gas。然后使用`getProductNum()`，可以看到当前有一个商品发布。再使用`getProduct(1)`，可以看到商品信息。  

![add a product][15]  
![get the product][16]

如果在商品开始拍卖前进行投标，操作会被拒绝。在商品发布三分钟后，4个用户要进行投标。先看用户1，以密匙"test1"进行加密，出价1.5倍初始价格，即是3个以太币。投标操作花费 110370 gas。如果用户1还想更改出价/更改竞拍密匙，操作将会被拒绝。  

![bid][17]

同理，用户2、3、4都进行投标，分别出价4个以太币、6个以太币、5个以太币。  


这时调用`getBidNum(1)`可以看到竞拍商品1（iPhoneX）的人数。然后查看当前四个用户的余额，可以看到除了消耗的部分gas外，4人分别花费了3/4/6/5个币。
![getBidNum][18]

等到竞拍的5分钟结束后，可以开始揭标。揭标需要输入之前的竞拍密匙，确保安全。  

第一个揭标的是用户1：由于现在只有用户1揭标，即是说暂时的最高竞价者是ta，而商品第二高价格就是其初始价格。在最终价格未确定前，暂时不会返还最高竞价者差价。

![reveal1][19]

第二个揭标的是用户2：用户2的出价比用户1高，因此ta成了最高竞价者。用户1出局，回水（把ta支付的3个以太币返还给他）。

![reveal2][20]

第二个揭标的是用户3：用户3揭标后，ta的出价又比用户2高，因此ta成了最高竞价者。用户2出局，回水（把ta支付的4个以太币返还给他）。

![reveal3][21]

最后揭标的是用户4：它的出价并没有用户3高，直接出局，回水（把ta支付的5个以太币返还给他）。

![- reveal4][22]

这个时候四个用户都揭标了， 查看一下买家是谁。

![noBuyerYet][23]

可是这时获取买家出现报错，这是因为交易并没有完全结束。最高竞价者用户3目前仍然支付的是ta自己的出价，成交时我们应该返还给他（最高价-次高价）的差价。同时，还应该修改商品的状态，判断是否成交。  

![getBuyer][24]  

最后看一下各用户的余额，用户3使用了5个以太币（次高价）获得了该商品。然后再看一看商品属性，可以看到第四个参数更改为1。这个参数代表的是商品状态枚举型变量，1表示已出售。

![getProduct2][25]


  [1]: https://nodejs.org/en/
  [2]: https://github.com/sysuxwh/MyPictureHost/blob/master/AuctionDapp/configuration/truffle_install.png
  [3]: https://truffleframework.com/boxes
  [4]: https://ethereum.stackexchange.com/questions/47937/error-in-unboxing-truffle-react-on-windows
  [5]: https://github.com/sysuxwh/MyPictureHost/blob/master/AuctionDapp/configuration/webpack_bug.png
  [6]: https://github.com/sysuxwh/MyPictureHost/blob/master/AuctionDapp/configuration/webpack_succeed.png
  [7]: https://cryptozombies.io/en/course
  [8]: https://github.com/trufflesuite/ganache/releases
  [9]: https://github.com/sysuxwh/MyPictureHost/blob/master/AuctionDapp/test/compile.png
  [10]: https://github.com/sysuxwh/MyPictureHost/blob/master/AuctionDapp/test/migrate_error.png
  [11]: https://github.com/sysuxwh/MyPictureHost/blob/master/AuctionDapp/test/develop.png
  [12]: https://github.com/sysuxwh/MyPictureHost/blob/master/AuctionDapp/test/migrate_develop.png
  [13]: https://github.com/sysuxwh/MyPictureHost/blob/master/AuctionDapp/test/test_bidders_n_getInstance.png
  [14]: https://github.com/sysuxwh/MyPictureHost/blob/master/AuctionDapp/test/test_getArg.png
  [15]: https://github.com/sysuxwh/MyPictureHost/blob/master/AuctionDapp/test/test_addProduct.png
  [16]: https://github.com/sysuxwh/MyPictureHost/blob/master/AuctionDapp/test/test_getProduct.png
  [17]: https://github.com/sysuxwh/MyPictureHost/blob/master/AuctionDapp/test/test_bid.png
  [18]: https://github.com/sysuxwh/MyPictureHost/blob/master/AuctionDapp/test/test_getBidNum.png
  [19]: https://github.com/sysuxwh/MyPictureHost/blob/master/AuctionDapp/test/test_bid1.png
  [20]: https://github.com/sysuxwh/MyPictureHost/blob/master/AuctionDapp/test/test_bid2.png
  [21]: https://github.com/sysuxwh/MyPictureHost/blob/master/AuctionDapp/test/test_bid3.png
  [22]: https://github.com/sysuxwh/MyPictureHost/blob/master/AuctionDapp/test/test_bid4.png
  [23]: https://github.com/sysuxwh/MyPictureHost/blob/master/AuctionDapp/test/test_noBuyerYet.png
  [24]: https://github.com/sysuxwh/MyPictureHost/blob/master/AuctionDapp/test/test_endAuction.png
  [25]: https://github.com/sysuxwh/MyPictureHost/blob/master/AuctionDapp/test/test_getProduct2.png
