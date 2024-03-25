+++
title = 'NFT'
date = 2024-03-25T09:22:21+08:00
+++
## NFT 的技术逻辑
首先，我们知道比特币——“比特币”是一种定义在比特币这条区块链上的“货币”，每个“比特币”从被挖出开始到整个流通的过程都是安全的，因为它受到所有比特币矿工的验证。
矿工都验证了什么呢：
+ 如果我没有1个比特币，那么我给不了你1个比特币，这点，实际上就是加密货币的核心功能——防止双重支付；
+ 我有1个比特币，那么我就能给你一个比特币——矿工们并不区分这个比特币是从哪来的，是在哪个区块挖出来的。比如说，其实中本聪曾经交易过他的比特币，而交易的人可能后来也做过别的交易……

于是，我们现在其实已经分不清谁手里的哪个币是中本聪挖出来的了，因为这些币和其他的币并没有区别，早就和其他的币混在一起了——也就是说，比特币是**同质化**（fungible）的。

接着就有了以太坊——以太坊上也有“以太币”，“以太币”拥有和“比特币”类似的属性——也防双重支付，也是同质化的。

以上这两种，我们称为一条区块链的原生货币。

然后，以太坊上不仅仅有“以太币”，根据某个叫做ERC-20的协议，任何人都可以发行某个货币，或者叫做通证（token），然后，ERC-20规定了这种通证可以在以太坊上和“以太币”一样——防双重支付，同质化，可以自由交换和流通，我们管ERC-20叫同质化通证，比如那些做ICO的项目，最早都是根据ERC-20在以太坊上发同质化通证的。当然那个时候，我们叫做发币。

再之后，又有人搞出了两个协议ERC-721和ERC-1155，这两个代币的协议和之前ERC-20的协议区别在于，这两种协议认为**每个通证都是不一样**的，也就是**非同质**的。这些通证就被成为**非同质通证**，即NFT。每个NFT有自己的类别、创建时间、特殊信息等等……每一个NFT都是独一无二，不可分拆的（现在也出现了可分拆的NFT，我们这里先不说），多个同类型的ERC-721通证或者ERC-1155通证在账户里的时候不会被简单地当作“10个xx通证”，而是会分别存为“xx通证A”，“xx通证B”……同时，矿工在NFT进行交易的时候，依旧遵循货币的规则，判断是否存在双重支付。

## ERC20
ERC20 Token是建立在以太坊区块链上的 Token。ERC20 Token是通过以太坊智能合约创建的，这意味着 Token的供应量、转移规则和其他属性都可以在智能合约中定义和控制。

ERC20 Token由于ERC20 Token建立在以太坊区块链上，因此可以与以太坊智能合约进行交互。
### ERC-20 工作原理
ERC-20 标准为以太坊区块链上的加密代币功能制定了一个全面的框架：
```solidity
function totalSupply() public view returns (uint256);
```
返回总代币供应量。
```solidity
function balanceOf(address _owner) public view returns (uint256 balance);
```
返回地址 _owner 的账户余额。
```solidity
function approve(address _spender, uint256 _value) public returns (bool success);
```
允许 _spender 从您的帐户多次提取，最高金额为 _value（总额）。如果再次调用此功能，则会用 _value 覆盖当前的授权。
```solidity
function allowance(address _owner, address _spender) public view returns (uint256 remaining);
```
返回 _spender 仍然允许从 _owner 提取的金额。
```solidity
function transferFrom(address _from, address _to, uint256 _value) public returns (bool success);
```
从地址 _from 向地址 _to 转移 _value 数量的代币，并且必须触发 Transfer 事件。
```solidity
event Transfer(address indexed _from, address indexed _to, uint256 _value);
```
当代币转移时，包括零值转移时必须触发。

一个创建新代币的代币合约应该在创建代币时触发一个Transfer事件，并将 _from 地址设置为0x0。
```solidity
event Approval(address indexed _owner, address indexed _spender, uint256 _value);
```
必须在调用 approve(address _spender, uint256 _value) 成功时触发。


### 创建 ERC-20 代币
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

interface IERC20 {
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);

    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
}
```

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import "./IERC20.sol";

contract ERC20 is IERC20{
    // 总量
    uint256 public override totalSupply;
    // 名称
    string public name;
    // 代号
    string public symbol;
    // 小数位数，同以太坊保持一致
    uint public decimals = 18;
    // 余额
    mapping(address => uint256) public override balanceOf;

    mapping(address => mapping(address => uint256)) public override allowance;

    constructor(string memory _name, string memory _symbol, uint256 _totalSupply){
        name = _name;
        symbol = _symbol;
        totalSupply = _totalSupply;
    }

    function transfer(address to, uint256 amount) external override returns (bool){
        require(balanceOf[msg.sender] >= amount, "ERROR:balance not enough");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        emit Transfer(msg.sender, to, amount);
        return true;
    }

    function approve(address spender, uint256 amount) external override returns (bool){
        require(msg.sender!=spender,"ERROR:Cant Approve Self.");
        require(balanceOf[msg.sender] >= amount, "ERROR:balance not enough");
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external override returns (bool){
        // 存在 gas fee, 不能等于
        require(balanceOf[from] > amount, "ERROR:balance not enough");
        require(allowance[from][msg.sender] > amount, "ERROR:allowance not enough");
        allowance[from][msg.sender] -= amount;
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        emit Transfer(from, to, amount);
        return true;
    }

    function mint(uint256 amount) external {
        require(msg.sender == owner,"ERROR:Only Owner Can Mint.");
        balanceOf[msg.sender] += amount;
        totalSupply += amount;
        emit Transfer(address(0), msg.sender, amount);
    }

    function burn(uint256 amount) external {
        require(msg.sender == owner,"ERROR:Only Owner Can Burn.");
        balanceOf[msg.sender] -= amount;
        totalSupply -= amount;
        emit Transfer(msg.sender, address(0), amount);
    }
}
```

## ERC165
ERC165是以太坊上的一个标准接口，用于判断合约是否实现了某些特定的接口。

ERC165接口定义了一个名为supportsInterface的函数，该函数接收一个bytes4类型的参数，表示要查询的接口ID。如果合约实现了指定的接口，则返回一个非零值，否则返回0。

例如，对于ERC721接口，其接口ID为0x80ac58cd。因此，要检查合约是否实现了ERC721接口，只需要调用supportsInterface(0x80ac58cd)函数。如果返回值非零，则表示合约实现了ERC721接口。

```solidity
interface ERC165 {
    function supportsInterface(bytes4 interfaceId) external view returns (bool);
}
```
合约实现该接口时，需要在supportsInterface函数中检查传入的interfaceId是否为合约实现的某个接口ID，如果是，则返回true，否则返回false。

## ERC721
ERC-721是以太坊上的一个标准协议，用于创建和交易非同质化Token（NFT）。NFT是一种数字资产，与其他Token不同，每一个NFT都是独特的，具有不可替代性。

与ERC-20协议不同，ERC-721协议的每一个Token都是唯一的，没有可替代性。这意味着每一个NFT都有其独特的标识符，称为tokenId。因此，ERC-721协议的实现需要支持这种不可替代性，确保每一个tokenId只能被拥有者所拥有。

另外，ERC-721还支持多样化的交易方式。与ERC-20协议只支持简单的Token交易不同，ERC-721可以支持更多复杂的交易，比如拍卖和租赁等。
```solidity
interface ERC721 /* is ERC165 */ {
    event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);
    event Approval(address indexed owner, address indexed approved, uint256 indexed tokenId);
    event ApprovalForAll(address indexed owner, address indexed operator, bool approved);

    //该函数用于查询指定地址拥有的NFT数量。
    function balanceOf(address owner) external view returns (uint256 balance);
    //该函数用于查询指定tokenId的拥有者地址。
    function ownerOf(uint256 tokenId) external view returns (address owner);

    //该函数用于将一个NFT从一个地址转移到另一个地址。如果接收地址是一个合约地址，并且合约实现了onERC721Received函数，那么该函数会调用onERC721Received函数。
    function safeTransferFrom(address from, address to, uint256 tokenId, bytes calldata data) external;
    function safeTransferFrom(address from, address to, uint256 tokenId) external;
    
    //该函数用于将一个NFT从一个地址转移到另一个地址。
    function transferFrom(address from, address to, uint256 tokenId) external;

    //该函数用于将NFT的控制权授予另一个地址。
    function approve(address to, uint256 tokenId) external;
    
    //该函数用于查询被授予NFT控制权的地址。
    function getApproved(uint256 tokenId) external view returns (address operator);

    //该函数用于将NFT的控制权授予一个操作者地址，并指定该操作者是否有权代表NFT所有者执行操作。
    function setApprovalForAll(address operator, bool approved) external;
    
    //该函数用于查询一个操作者地址是否被授权代表某个NFT所有者执行操作。
    function isApprovedForAll(address owner, address operator) external view returns (bool);
}
```

## ERC1155
ERC1155相比于ERC721简而言之最大的区别就是它可以一个合约承载多类FT与NFT，可以将其理解为是ERC20和ERC721的融合加强版，想发行同质化和非同质化的代币1155全部搞定，而不用用多个合约承载再进行交互。

ERC721是一个合约承载1类NFT，1类NFT承载多个NFT，如无聊猿，它的合约有且仅能发行无聊猿这一套NFT，每个具体的NFT编号均不相同为递增，但是ERC1155一个合约可以发行多类NFT，它最常用的场景在游戏，比如一个游戏中，可能会有很多类装备如“武器”、“坐骑”、“药品”等，这些装备有的是非同质化的，比如屠龙宝刀只有1个，有的是同质化的比如药品都是一样的喝一瓶补10滴血，而传统的721只能发行一类实体，但是1155却可以发行多类，说起来还有点抽象是不是，直接上代码。

我们来演示一个最简单的1155协议合约，自上而下，我先创建了3种代币类型分别为武器wq、坐骑zj和宝石bs，他们的编号分别为0、1、2。

然后我定义这三类代币的发售最大数量分别为1、10和9999。

在mint函数中，传入三个参数分别为地址、代币编号和数量，依次校验当前用户要mint的代币类型数量是否超过了最大发售数，若未超过则执行mint操作，这里大家注意，相比721的mint这里的mint多传入了一个id，这个id即1155协议中定义的代币类型，同样的在校验的过程中用到了totalSupply相比如721多传入了id，也是因为有多个代币类型，所以需要用id来检索到底要获取的是哪一个代币类型的数量。

```solidity
contract MyToken is ERC1155, Ownable, Pausable, ERC1155Burnable, ERC1155Supply{
    constructor() ERC1155(""){}
    uint256 public constant wq = 0;
    uint256 public constant zj = 1;
    uint256 public constant bs = 2;
    uint256 wq_max =1;
    uint256 zj_max=10;
    uint256 bs_max=9999;

    function mint(address account,uint256 id,uint256 amount) public onlyOwner {
        if(id == 0) {
            require(totalSupply(0) + amount <= wq_max,"Purchase: Max supply reached");
        }
        if(id == 1) {
            require(totalSupply(1) + amount <= zj_max, "Purchase: Max supply reached");
        } 
        if(id == 2) {
            require(totalSupply(2) + amount <= bs_max, "Purchase: Max supply reached");
        }
        _mint(account,id,amount,"");
    }
}
```


## ERC404
ERC-404 是一种介于 ERC-20 和ERC-721 之间的试验性的代币标准。

简单来讲，ERC-404 是一个可以让 NFT 像数字代币一样进行拆分交易，实现“图币”互换的协议，即 1 枚 ERC-404 代币会对应 1 枚 Replicant NFT。大白话来说就是，该协议可以把图片（NFT）转化为代币、把代币转化为图片，两者具有“绑定”的关系。

而且，ERC-404 代币可以直接在 Uniswap 等去中心化平台进行交易，当你购买一个 ERC-404 代币时，你的钱包将自动获得一个 Replicant NFT。而当你卖出该代币时，对应的 NFT 将被自动销毁。

当然，这里面还会涉及一个整数的问题，因为你只有购买（凑齐）一整枚 ERC-404 代币才可持有一个对应的 NFT。比如，你持有 0.5 个代币，那么就相当于拥有 0 个NFT。你持有 1 个代币，那么就相当于拥有 1 个NFT。你持有 1.5 个代币，那么就相当于拥有 1 个NFT。你持有 2 个代币，那么就相当于拥有 2 个NFT。以此类推。这个本质上是将 NFT 图片内容碎片化，因为 NFT 会随着手中代币数量的变化而变化，以此来造成独特的随机属性。

