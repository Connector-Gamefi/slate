---
title: API Reference

language_tabs: # must be one of https://git.io/vQNgJ
  - solidity

toc_footers:
  - <a href='#'>Sign Up for a Developer Key</a>
  - <a href='https://github.com/slatedocs/slate'>Documentation Powered by Slate</a>

includes:
  - errors

search: true

code_clipboard: true

meta:
  - name: description
    content: Documentation for the Kittn API
---

# Introduction

这里将会介绍Connexion所有合约的技术细节，其中包括FT、NFT、Market、Treasure和Auction几部分内容。

# 合约实现细节

## ERC20代币合约
 
 除了FT代币的基础功能以外，还需要提供一键创建代币合约的功能。为了实现这个功能，我们编写了3个合约，分别是：`GameERC20Factory.sol`、 `GameERC20Proxy.sol` 和 `GameERC20Token.sol`，下面是他们的关系图：

 ![image ERC20关系图](./images/ERC20.png)

 对 delegatecall 熟悉的人应该知道：合约A通过delegatecall调用合约B，交易会按照合约B的逻辑执行，但是执行的上下文和更改的状态都在合约A中，我们称合约A为代理合约，合约B为逻辑合约。`GameERC20Proxy`就是代理合约，存储token的状态，用户可以通过`GameERC20Factory`创建任意个`GameERC20Proxy`的实例，每个实例就是一个token，所有通过`GameERC20Factory`创建出来的token的逻辑合约都是`GameERC20Token`。

---

### GameERC20Factory

#### Storage variable

##### vaultCount

```solidity
uint256 public vaultCount;
```

 vaultCount表示已经创建的token总数

##### vaults

```solidity
mapping(uint256 => address) public vaults;
```

 vaults存储已经创建的token地址

##### logic

```solidity
address public immutable logic;
```

 logic是所有代币的逻辑合约地址，使用不可变的变量存储逻辑合约地址保证每个工厂创建出来的代币逻辑合约都是一样的。

#### Functions

##### generate

```solidity
function generate(
    string memory _name,
    string memory _symbol,
    uint256 _cap
) external whenNotPaused returns (uint256 _index) {
    bytes memory _initializationCallData =
    abi.encodeWithSignature(
        "initialize(string,string,uint256,address)",
        _name,
        _symbol,
        _cap,
        msg.sender
    );

    address vault = address(
        new GameErc20Proxy(
            logic,
            _initializationCallData
        )
    );

    vaults[vaultCount] = vault;
    vaultCount++;

    return vaultCount - 1;
}
```

generate是用来创建新代币的方法

Parameters:

Name | Type | Description
--------- | ------- | -----------
_name | string | token's name
_symbol | string | token's symbol
_cap | uint256 | 代币的最大容量

Return Values:

Name | Type | Description
--------- | ------- | -----------
_index | uint256 | 当前代币在已创建代币数组中的下标












### GameErc20Proxy

#### Storage variable

##### logic

```solidity
address public immutable logic;
```

 logic是所有代币的逻辑合约地址，使用不可变的变量存储逻辑合约地址保证token的合约逻辑不可以更改，可升级的合约就是通过改变logic合约实现的。

#### Functions

##### fallback

```solidity
fallback() external payable {
    address _impl = logic;
    assembly {
        let ptr := mload(0x40)
        calldatacopy(ptr, 0, calldatasize())
        let result := delegatecall(gas(), _impl, ptr, calldatasize(), 0, 0)
        let size := returndatasize()
        returndatacopy(ptr, 0, size)
        switch result
            case 0 {
                revert(ptr, size)
            }
            default {
                return(ptr, size)
            }
    }
}
```

 fallback是实现代理合约的关键函数，代理合约的概念是在[EIP1167协议](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1167.md)中产生，有如下几个特性：

 1. 调用成功后返回true，无法管理返回的数据；
 2. 当调用的方法在代理合约中不存在时，合约会调用`fallback`函数。可以编写`fallback`函数的逻辑处理这种情况。代理合约使用自定义的`fallback`函数将调用请求重定向到逻辑合同中。
 3. 每当合约A将调用代理到另一个合同B时，它都会在合约A的上下文中执行合约B的代码。这意味着将保留msg.value和msg.sender值，并且每次存储修改都会影响合约A。

 也可以参考Openzipplin的合约库具体实现[Openzeppelin Proxy](https://github.com/OpenZeppelin/openzeppelin-labs/blob/master/upgradeability_using_eternal_storage/contracts/Proxy.sol)

### GameERC20Token

```solidity
function mint(address account, uint256 amount) public onlyOwner {
  if (totalSupply() + amount > cap)
      amount = cap - totalSupply();
  _mint(account, amount);
}
```

 拥有owner权限的地址铸造代币的方法，铸造的总数量将不可以超过预设的总供应量

Parameters:

Name | Type | Description
--------- | ------- | -----------
account | address | 接收铸造代币的地址
amount | uint256 | 铸造的数量


## 多游戏通用的 NFT token contract

 Loot曾经被定义为NFT的新范式。在合约代码层面，Loot的创新在于它将NFT的属性写在去中心化系统上，很好的解决了传统NFT的Metadata可以被开发者随意变更的问题。在形象艺术方面，Loot刻意将图片省略，使得不同的作者可以各自发挥想象力，用不同的画面来描述同一个Loot。

 虽然Loot相较于传统NFT有了很大的进步，但是Loot也有一些不足。Loot的属性种类十分有限，也十分具体。举个例子：第一代Loot中有一个属性是“战斧”，尽管画家们可以任意设计“战斧”的形象，但是它只能是战斧，这样就将画家们的想象力禁锢在这一件事物中。

 我们基于Loot的优缺点，做了进一步的创新，使用数字来代替Loot中的文字属性，并且单个NFT拥有属性的数量将不再受限制。这样，同一个NFT将可以在不同的故事剧本中成为完全不一样的角色，也为成为不同游戏世界的互通的桥梁。

 关于connecxion的NFT，我们创建了一个适用于GameFi的NFT新协议[Non-fungible Token for GameFi](https://github.com/bnb-chain/BEPs/pull/129)。

### Storage variable

> 存储属性的数据结构

```solidity
struct AttributeBaseData {
    uint8 decimal;
    bool exist;
}

struct AttributeData {
    uint128 attrID;
    uint128 attrValue;
}

// attrID => decimal
mapping(uint128 => AttributeBaseData) internal _attrBaseData;
// tokenID => attribute data
mapping(uint256 => AttributeData[]) internal _attrData;
```

`AttributeBaseData`存储的是单个属性的精度，为了解决小数问题。

`AttributeData`存储的是属性ID和属性值

`_attrBaseData`是一个键值对，key为属性ID，值为属性的基础数据（精度、是否被创建），用来查询属性的精度

`_attrData`也是一个键值对，key为tokenID，值为此NFT具有的属性数组

### Functions

#### tokenURI

> 通过SVG展示NFT属性的方法

```solidity
function tokenURI(uint256 tokenID) override public view returns (string memory) {
    AttributeData[] memory attrData = _attrData[tokenID];

    string memory output = '<svg xmlns="http://www.w3.org/2000/svg" preserveAspectRatio="xMinYMin meet" viewBox="0 0 350 350"><style>.base { fill: white; font-family: serif; font-size: 14px; }</style><rect width="100%" height="100%" fill="black" /><text x="10" y="20" class="base">ID</text><text x="80" y="20" class="base">Value</text><text x="185" y="20" class="base">ID</text><text x="255" y="20" class="base">Value';

    string memory p1 = '</text><text x="10" y="';
    string memory p2 = '</text><text x="80" y="';
    string memory p3 = '</text><text x="185" y="';
    string memory p4 = '</text><text x="255" y="';
    string memory p5 = '" class="base">';

    bytes memory tb;
    for (uint256 i; i < _attrData[tokenID].length; i++) {
        uint128 id = attrData[i].attrID;
        uint128 value = attrData[i].attrValue;
        if (i % 2 == 0) {
            string memory y = toString(40 + 20 * i / 2);
            tb = abi.encodePacked(tb, p1, y, p5, toString(id), p2, y, p5, toString(value));
        } else {
            string memory y = toString(40 + 20 * (i - 1) / 2);
            tb = abi.encodePacked(tb, p3, y, p5, toString(id), p4, y, p5, toString(value));
        }
    }
    tb = abi.encodePacked(tb, '</text></svg>');

    string memory json = Base64.encode(bytes(string(abi.encodePacked('{"name": "Bag #', toString(tokenID),
        '", "description": "GameLoot is a general NFT for games. Images, attribute name and other functionality are intentionally omitted for each game to interprets. You can use gameLoot as you like in a variety of games.", "image": "data:image/svg+xml;base64,',
        Base64.encode(abi.encodePacked(output, tb)), '"}'))));
    output = string(abi.encodePacked('data:application/json;base64,', json));

    return output;
}
```

 这里同样采用Loot的方式，使用SVG展示NFT的属性。
 
Parameters:

Name | Type | Description
--------- | ------- | -----------
tokenID | uint256 | NFT 的唯一标识

Return Values:

Name | Type | Description
--------- | ------- | -----------
output | string | SVG数据经过Base64编码后的数据

#### create

> 创建单个属性

```solidity
function create(uint128 attrID_, uint8 decimals_) override public onlyOwner {
    super.create(attrID_, decimals_);
}
```

 只有owner拥有属性的创建权限，已经存在的属性ID不可以被再次创建。
 
Parameters:

Name | Type | Description
--------- | ------- | -----------
attrID_ | uint128 | NFT属性的唯一标识
decimals_ | uint8 | 属性的精度

#### createBatch

> 创建多个属性

```solidity
function createBatch(uint128[] memory attrIDs_, uint8[] memory decimals_) override public onlyOwner {
    super.createBatch(attrIDs_, decimals_);
}
```

 只有owner拥有属性的创建权限，已经存在的属性ID不可以被再次创建。
 
Parameters:

Name | Type | Description
--------- | ------- | -----------
attrIDs_ | uint128[] | NFT属性的唯一标识数组
decimals_ | uint8[] | 属性的精度数组，每个元素的下标与attrIDs_的元素一一对应

#### attach

> 将单个属性添加到NFT属性列表中

```solidity
function attach(uint256 tokenID_, uint128 attrID_, uint128 value_) override public onlyTreasure {
    _attach(tokenID_, attrID_, value_);
}
```

 将某一个属性和对应的值添加到NFT属性列表中，只有[treasure合约](#金库合约)才拥有调用此方法的权限。
 
Parameters:

Name | Type | Description
--------- | ------- | -----------
tokenID_ | uint256 | NFT 的唯一标识
attrID_ | uint128 | NFT属性的唯一标识
value_ | uint128 | NFT属性的值

#### attachBatch

> 将多个属性添加到NFT属性列表中

```solidity
function attachBatch(uint256 tokenID_, uint128[] memory attrIDs_, uint128[] memory values_) override public onlyTreasure {
    _attachBatch(tokenID_, attrIDs_, values_);
}
```

 将某多个属性和对应的值添加到NFT属性列表中，只有[treasure合约](#金库合约)才拥有调用此方法的权限。
 
Parameters:

Name | Type | Description
--------- | ------- | -----------
tokenID_ | uint256 | NFT 的唯一标识
attrIDs_ | uint128[] | NFT属性的唯一标识数组
values_ | uint128[] | NFT属性值数组，每个元素下标与attrIDs_元素的对应

#### remove

> 将单个属性从NFT属性列表中移除

```solidity
function remove(uint256 tokenID_, uint256 attrIndex_) override public onlyTreasure {
    _remove(tokenID_, attrIndex_);
}
```

 通过属性的下标，将对应的属性从NFT属性列表中移除，若下标超出属性数组范围，交易会执行失败，只有[treasure合约](#金库合约)才拥有调用此方法的权限。
 
Parameters:

Name | Type | Description
--------- | ------- | -----------
tokenID_ | uint256 | NFT 的唯一标识
attrIndex_ | uint256 | 对应属性在NFT属性列表中的下标

#### removeBatch

> 将多个属性从NFT属性列表中移除

```solidity
function removeBatch(uint256 tokenID_, uint256[] memory attrIndexes_) override public onlyTreasure {
    _removeBatch(tokenID_, attrIndexes_);
}
```

 通过多个属性的下标，将对应的属性从NFT属性列表中移除，若下标超出属性数组范围，交易会执行失败，只有[treasure合约](#金库合约)才拥有调用此方法的权限。
 
Parameters:

Name | Type | Description
--------- | ------- | -----------
tokenID_ | uint256 | NFT 的唯一标识
attrIndexes_ | uint256[] | 对应属性在NFT属性列表中的下标数组

#### update

> 更新NFT单个属性的值

```solidity
function update(uint256 tokenID_, uint256 attrIndex_, uint128 value_) override public onlyTreasure {
    _update(tokenID_, attrIndex_, value_);
}
```

 通过属性的下标，将对应的属性的值更新，若下标超出属性数组范围，交易会执行失败，只有[treasure合约](#金库合约)才拥有调用此方法的权限。
 
Parameters:

Name | Type | Description
--------- | ------- | -----------
tokenID_ | uint256 | NFT 的唯一标识
attrIndex_ | uint256 | 对应属性在NFT属性列表中的下标
value_ | uint128 | 需要设置的对应属性的新值

#### updateBatch

> 更新NFT多个属性的值

```solidity
function updateBatch(uint256 tokenID_, uint256[] memory attrIndexes_, uint128[] memory values_) override public onlyTreasure {
    _updateBatch(tokenID_, attrIndexes_, values_);
}
```

 通过属性的下标，将多个属性的值更新，若下标超出属性数组范围，交易会执行失败，只有[treasure合约](#金库合约)才拥有调用此方法的权限。
 
Parameters:

Name | Type | Description
--------- | ------- | -----------
tokenID_ | uint256 | NFT 的唯一标识
attrIndexes_ | uint256[] | 对应属性在NFT属性列表中的下标数组
values_ | uint128[] | 需要设置的对应属性的新值集合，每个元素下标与attrIndexes_元素的对应

## 金库合约

> 其中一个使用ecrecover指令恢复签名者的方法

```solidity
function signatureWallet(
    address _wallet,
    address _this,
    address _token,
    uint256 _tokenID,
    uint256 _nonce,
    uint128[] memory _attrIDs,
    uint128[] memory _attrValues,
    uint256[] memory _attrIndexesUpdate,
    uint128[] memory _attrValuesUpdate,
    uint256[] memory _attrIndexesRMs,
    bytes memory _signature
) internal pure returns (address){
    bytes32 hash = keccak256(
        abi.encode(_wallet, _this, _token, _tokenID, _nonce, _attrIDs, _attrValues, _attrIndexesUpdate, _attrValuesUpdate, _attrIndexesRMs)
    );
    return ECDSA.recover(ECDSA.toEthSignedMessageHash(hash), _signature);
}
```

 金库合约是区块链系统与中心化游戏系统交互的关口，所有从区块链上充值到游戏内部的资产(包括FT和NFT)都是通过将资产锁定在treasure合约中来实现的；同样的，所有从游戏内部提现的资产也是通过解锁treasure合约中的资产来实现的。提现流程图如下：

 ![image](./images/withdraw.png)

 从图中可以看出，用户需要签名才可以发出有效提现交易。签名在合约中的应用有两方面：ecrecover指令可以恢复签名数据的签名者地址，这样就能验证签名数据的真伪；另外，提现方法是可以被任何用户调用的，为了保证交易可以符合预期地被确认，签名机会对如下数据进行签名：

 - 接收者地址，为了防止错误的地址接收到从treasure合约提现出的资产
 - 金库合约地址，为了防止签名在另一个金库合约重用
 - 代币合约地址，确定此签名只能够从treasure合约中转移出一种资产
 - 不重复的随机数，确保此签名只能够使用一次
 - FT的数量，用来控制提现出的金额
 - NFT的tokenID，用来确保将正确的NFT提出
 - NFT属性更新后的数据

### ERC20 金库合约

#### Functions

##### upChain

> 

```solidity

```

Parameters:

Name | Type | Description
--------- | ------- | -----------
tokenID_ | uint256 | NFT 的唯一标识
attrIndexes_ | uint256[] | 对应属性在NFT属性列表中的下标数组
values_ | uint128[] | 需要设置的对应属性的新值集合，每个元素下标与attrIndexes_元素的对应

# 控件列表

## 右边代码：

> To authorize, use this code:

```ruby
require 'kittn'

api = Kittn::APIClient.authorize!('meowmeowmeow')
```

```python
import kittn

api = kittn.authorize('meowmeowmeow')
```

```shell
# With shell, you can just pass the correct header with each request
curl "api_endpoint_here" \
  -H "Authorization: meowmeowmeow"
```

```javascript
const kittn = require('kittn');

let api = kittn.authorize('meowmeowmeow');
```

> Make sure to replace `meowmeowmeow` with your API key.

## 注意提醒控件

<aside class="notice">
You must replace <code>meowmeowmeow</code> with your personal API key.
</aside>

<aside class="success">
Remember — a happy kitten is an authenticated kitten!
</aside>

<aside class="warning">Inside HTML code blocks like this one, you can't use Markdown, so use <code>&lt;code&gt;</code> blocks to denote code.</aside>

## 表格

Parameter | Default | Description
--------- | ------- | -----------
include_cats | false | If set to true, the result will also include cats.
available | true | If set to false, the result will include kittens that have already been adopted.

