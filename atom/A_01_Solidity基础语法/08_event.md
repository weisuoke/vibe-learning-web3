# Event 事件

## 1. 【30字核心】

**Event是智能合约的日志机制，用于记录重要操作到区块链日志中，Gas成本远低于Storage，DApp前端可以监听和过滤这些事件实现实时更新。**

---

## 2. 【第一性原理】

### 什么是第一性原理？

**第一性原理**：回到事物最基本的真理，从源头思考问题

### Event的第一性原理 🎯

#### 1. 最基础的定义

**Event = 写入区块链日志的数据**

核心特点：
- **写入日志**：存储在交易收据的logs中，不是合约Storage
- **只写不读**：合约内部无法读取已发出的事件
- **便宜**：Gas成本比Storage低约5-10倍
- **可索引**：最多3个参数可标记为`indexed`，支持高效过滤

仅此而已！没有更基础的了。

#### 2. 为什么需要Event？

**核心问题：如何让前端应用知道合约发生了什么？**

传统方法的问题：
- **轮询合约状态**：频繁调用view函数，效率低
- **存储到Storage**：Gas成本高昂
- **返回值**：只有调用者能收到，其他人看不到

**解决方案**：Event提供了一种便宜、高效的通知机制。

#### 3. Event的三层价值

##### 价值1：前端通知 - 实时更新UI

```solidity
contract TokenContract {
    event Transfer(address indexed from, address indexed to, uint256 value);
    
    function transfer(address to, uint256 amount) external {
        // 转账逻辑...
        
        // 发出事件通知前端
        emit Transfer(msg.sender, to, amount);
    }
}
```

```javascript
// 前端监听事件
contract.on("Transfer", (from, to, value) => {
    console.log(`${from} 转账 ${value} 给 ${to}`);
    updateUI(); // 实时更新界面
});
```

##### 价值2：历史记录 - 便宜的链上日志

```solidity
contract AuditTrail {
    // 存储到Storage：约20,000 Gas
    mapping(uint256 => string) public actions;
    uint256 public actionCount;
    
    function recordToStorage(string memory action) external {
        actions[actionCount++] = action;  // 很贵！
    }
    
    // 发出事件：约375 Gas + 8 Gas/字节
    event ActionRecorded(address indexed user, string action, uint256 timestamp);
    
    function recordToEvent(string memory action) external {
        emit ActionRecorded(msg.sender, action, block.timestamp);  // 便宜！
    }
}
```

##### 价值3：链下索引 - 支持复杂查询

```javascript
// 使用indexed参数高效过滤
const filter = contract.filters.Transfer(userAddress, null); // from = userAddress
const events = await contract.queryFilter(filter, fromBlock, toBlock);

// 获取用户的所有转账记录
events.forEach(event => {
    console.log(`转给 ${event.args.to}，金额 ${event.args.value}`);
});
```

#### 4. 从第一性原理推导Event设计

**推理链：**

```
1. 前提：DApp需要知道合约状态变化
   ↓
2. 推导：轮询效率低，需要推送机制
   ↓
3. 推导：数据需要便宜地记录到链上
   ↓
4. 推导：使用日志而非Storage（便宜5-10倍）
   ↓
5. 推导：需要能过滤特定事件 → indexed参数
   ↓
6. 推导：需要按主题查询 → topics数组
   ↓
7. 推导：合约不需要读取事件 → 只写设计
   ↓
8. 最终：Event = 便宜的只写链上日志 + 可索引查询
```

#### 5. 一句话总结第一性原理

**Event是智能合约的只写日志系统，通过indexed参数支持高效过滤，Gas成本远低于Storage，是DApp前后端通信和历史记录的核心机制。**

---

## 3. 【3个核心概念】

### 核心概念1：Event声明与触发 📢

**一句话定义：** 使用`event`关键字声明事件结构，使用`emit`关键字触发事件。

#### 基本语法：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract EventBasics {
    // ===== 声明事件 =====
    
    // 简单事件
    event SimpleEvent();
    
    // 带参数的事件
    event Transfer(address from, address to, uint256 amount);
    
    // 带indexed的事件（最多3个indexed参数）
    event Approval(
        address indexed owner, 
        address indexed spender, 
        uint256 value
    );
    
    // 复杂事件
    event OrderCreated(
        uint256 indexed orderId,
        address indexed buyer,
        uint256 totalPrice,
        uint256 timestamp,
        string description  // 非indexed的动态类型
    );
    
    // ===== 触发事件 =====
    
    function emitSimple() external {
        emit SimpleEvent();
    }
    
    function emitTransfer(address to, uint256 amount) external {
        emit Transfer(msg.sender, to, amount);
    }
    
    function emitApproval(address spender, uint256 value) external {
        emit Approval(msg.sender, spender, value);
    }
    
    function createOrder(uint256 orderId, string memory desc) external payable {
        emit OrderCreated(
            orderId,
            msg.sender,
            msg.value,
            block.timestamp,
            desc
        );
    }
}
```

**事件存储结构：**

```
每个事件在日志中的结构：
┌─────────────────────────────────────────────────┐
│ address: 发出事件的合约地址                      │
├─────────────────────────────────────────────────┤
│ topics[0]: 事件签名的keccak256哈希              │
│ topics[1]: 第1个indexed参数                     │
│ topics[2]: 第2个indexed参数                     │
│ topics[3]: 第3个indexed参数                     │
├─────────────────────────────────────────────────┤
│ data: 非indexed参数的ABI编码                    │
└─────────────────────────────────────────────────┘

示例：emit Transfer(alice, bob, 100)
- topics[0]: keccak256("Transfer(address,address,uint256)")
- topics[1]: alice的地址（如果indexed）
- topics[2]: bob的地址（如果indexed）
- data: 100的ABI编码
```

---

### 核心概念2：indexed参数与过滤 🔍

**一句话定义：** `indexed`标记的参数会作为topics存储，可以被高效过滤查询，最多3个indexed参数。

#### indexed详解：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract IndexedDemo {
    // indexed参数存储在topics中，可以用于过滤
    event Transfer(
        address indexed from,    // topics[1]
        address indexed to,      // topics[2]
        uint256 value            // data（不能用于过滤）
    );
    
    // 最多3个indexed参数
    event ComplexEvent(
        uint256 indexed id,      // topics[1]
        address indexed user,    // topics[2]
        bytes32 indexed hash,    // topics[3]
        uint256 amount,          // data
        string message           // data
    );
    
    // 动态类型作为indexed会存储其keccak256哈希
    event StringIndexed(
        string indexed name      // topics[1] = keccak256(name)
    );
    
    function emitEvents() external {
        emit Transfer(msg.sender, address(0x123), 100);
        emit ComplexEvent(1, msg.sender, keccak256("test"), 50, "hello");
        emit StringIndexed("alice");
    }
}
```

**前端过滤示例：**

```javascript
const { ethers } = require('ethers');

// 连接合约
const contract = new ethers.Contract(address, abi, provider);

// ===== 过滤特定事件 =====

// 过滤from为特定地址的Transfer
const filterFrom = contract.filters.Transfer(userAddress, null, null);

// 过滤to为特定地址的Transfer
const filterTo = contract.filters.Transfer(null, userAddress, null);

// 同时过滤from和to
const filterBoth = contract.filters.Transfer(userAddress, recipientAddress, null);

// ===== 查询历史事件 =====

// 查询最近1000个区块内的事件
const fromBlock = await provider.getBlockNumber() - 1000;
const events = await contract.queryFilter(filterFrom, fromBlock, 'latest');

events.forEach(event => {
    console.log(`从 ${event.args.from} 转到 ${event.args.to}，金额 ${event.args.value}`);
    console.log(`区块号: ${event.blockNumber}`);
    console.log(`交易哈希: ${event.transactionHash}`);
});
```

**indexed vs 非indexed对比：**

| 特性 | indexed | 非indexed |
|------|---------|-----------|
| 存储位置 | topics | data |
| 可过滤 | ✅ | ❌ |
| 数量限制 | 最多3个 | 无限制 |
| 动态类型 | 存储哈希 | 存储原值 |
| Gas成本 | 稍高 | 稍低 |

---

### 核心概念3：前端事件监听 🎧

**一句话定义：** 前端可以通过WebSocket或轮询监听实时事件，或查询历史事件日志。

#### 完整的前端集成：

```javascript
// ===== 使用ethers.js监听事件 =====

const { ethers } = require('ethers');

// 使用WebSocket provider以支持实时监听
const provider = new ethers.WebSocketProvider('wss://eth-mainnet.ws.alchemyapi.io/v2/YOUR_KEY');
const contract = new ethers.Contract(contractAddress, abi, provider);

// ===== 实时监听 =====

// 监听所有Transfer事件
contract.on("Transfer", (from, to, value, event) => {
    console.log(`转账事件:`);
    console.log(`  从: ${from}`);
    console.log(`  到: ${to}`);
    console.log(`  金额: ${ethers.formatEther(value)} ETH`);
    console.log(`  区块: ${event.blockNumber}`);
    console.log(`  交易哈希: ${event.transactionHash}`);
});

// 监听特定条件的事件
const myAddress = '0x...';
const filter = contract.filters.Transfer(myAddress, null);
contract.on(filter, (from, to, value, event) => {
    console.log(`我转出了 ${ethers.formatEther(value)} ETH 给 ${to}`);
});

// ===== 一次性监听 =====

contract.once("Transfer", (from, to, value) => {
    console.log('收到第一个转账事件');
});

// ===== 查询历史事件 =====

async function getTransferHistory(userAddress, fromBlock = 0) {
    const filter = contract.filters.Transfer(userAddress, null);
    const events = await contract.queryFilter(filter, fromBlock, 'latest');
    
    return events.map(event => ({
        from: event.args.from,
        to: event.args.to,
        value: event.args.value,
        blockNumber: event.blockNumber,
        transactionHash: event.transactionHash
    }));
}

// ===== 停止监听 =====

function stopListening() {
    contract.removeAllListeners("Transfer");
}

// ===== 使用wagmi/viem（React项目推荐）=====

// 在React组件中
import { useContractEvent } from 'wagmi';

function TransferListener() {
    useContractEvent({
        address: contractAddress,
        abi: contractAbi,
        eventName: 'Transfer',
        listener(logs) {
            logs.forEach(log => {
                console.log('Transfer:', log.args);
            });
        },
    });
    
    return <div>正在监听转账事件...</div>;
}
```

---

## 4. 【最小可用】

掌握以下内容，就能正确使用Event：

### 4.1 基本事件声明和触发

```solidity
contract EventBasics {
    // 声明事件
    event Deposited(address indexed user, uint256 amount);
    
    // 触发事件
    function deposit() external payable {
        emit Deposited(msg.sender, msg.value);
    }
}
```

### 4.2 ERC20标准事件

```solidity
contract ERC20Events {
    // ERC20标准定义的事件
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    
    function transfer(address to, uint256 amount) external returns (bool) {
        // 转账逻辑...
        emit Transfer(msg.sender, to, amount);
        return true;
    }
    
    function approve(address spender, uint256 amount) external returns (bool) {
        // 授权逻辑...
        emit Approval(msg.sender, spender, amount);
        return true;
    }
}
```

### 4.3 前端监听模板

```javascript
// ethers.js v6
import { ethers } from 'ethers';

const provider = new ethers.WebSocketProvider('wss://...');
const contract = new ethers.Contract(address, abi, provider);

// 实时监听
contract.on("Transfer", (from, to, value) => {
    console.log(`${from} -> ${to}: ${value}`);
});

// 查询历史
const events = await contract.queryFilter("Transfer", -1000); // 最近1000块
```

### 4.4 indexed使用原则

```solidity
contract IndexedGuidelines {
    // ✅ 需要过滤查询的用indexed
    event Transfer(
        address indexed from,   // 需要按发送者查询
        address indexed to,     // 需要按接收者查询
        uint256 value           // 不需要过滤，放data
    );
    
    // ❌ 不要对所有参数都indexed
    // 每个indexed参数增加约375 Gas
    // 最多只能有3个indexed
    
    // ✅ 动态类型indexed会存哈希
    event NameChanged(
        address indexed user,
        string indexed oldName, // 存储keccak256(oldName)
        string newName          // 存储原始值
    );
}
```

---

**这些知识足以：**
- ✅ 在合约中正确发出事件
- ✅ 前端监听实时事件
- ✅ 查询历史事件
- ✅ 使用indexed进行过滤

---

## 5. 【1个类比】

### 类比1：Event = 通知铃声 🔔

#### 生活场景类比：手机通知系统

想象你的手机通知系统：

**通知场景：**
- **App发送通知（emit）**：微信发送"收到转账"通知
- **通知内容（参数）**：转账人、金额、时间
- **通知分类（indexed）**：可以按App、类型筛选通知
- **通知历史（logs）**：可以查看历史通知记录

**对应关系：**

| 手机通知 | Event | 说明 |
|---------|-------|------|
| 发送通知 | emit | 触发事件 |
| 通知内容 | event参数 | 事件携带的数据 |
| 通知分类 | indexed | 可筛选的标签 |
| 通知中心 | logs | 所有事件存储的地方 |
| 筛选通知 | filter | 按条件查询事件 |

**举例：**

```
收到通知：
┌─────────────────────────────────────────┐
│ 📱 微信支付                              │
│ 收到转账 ¥100.00                        │
│ 来自：张三                               │
│ 时间：12:30                              │
└─────────────────────────────────────────┘

对应的Event：
emit Transfer(张三, 我, 100, 12:30);
```

```solidity
contract NotificationSystem {
    // 通知事件
    event Notification(
        address indexed app,      // 哪个App
        address indexed user,     // 通知谁
        string message,           // 通知内容
        uint256 timestamp         // 时间
    );
    
    // 发送通知
    function notify(address user, string memory message) external {
        emit Notification(msg.sender, user, message, block.timestamp);
    }
}
```

---

#### 前端领域类比：EventEmitter / 自定义事件

如果你是前端工程师，Event就像Node.js的EventEmitter或浏览器的CustomEvent：

**Node.js EventEmitter：**

```javascript
const EventEmitter = require('events');

class MyContract extends EventEmitter {
    transfer(from, to, amount) {
        // 执行转账逻辑...
        
        // 发出事件（类似emit）
        this.emit('Transfer', { from, to, amount });
    }
}

const contract = new MyContract();

// 监听事件
contract.on('Transfer', (data) => {
    console.log(`转账: ${data.from} -> ${data.to}: ${data.amount}`);
});

contract.transfer('Alice', 'Bob', 100);
```

**浏览器CustomEvent：**

```javascript
// 创建自定义事件（类似Solidity的event声明）
const transferEvent = new CustomEvent('Transfer', {
    detail: { from: 'Alice', to: 'Bob', amount: 100 }
});

// 监听事件
document.addEventListener('Transfer', (e) => {
    console.log('转账:', e.detail);
});

// 触发事件（类似emit）
document.dispatchEvent(transferEvent);
```

**对应的Solidity：**

```solidity
// Solidity事件
event Transfer(address indexed from, address indexed to, uint256 value);

function transfer(address to, uint256 amount) external {
    // 执行转账...
    
    // 发出事件
    emit Transfer(msg.sender, to, amount);
}
```

**对比表：**

| 概念 | JavaScript | Solidity |
|-----|-----------|----------|
| 声明事件 | class定义/CustomEvent | event关键字 |
| 触发事件 | emit()/dispatchEvent() | emit |
| 监听事件 | on()/addEventListener() | contract.on() |
| 过滤事件 | 手动过滤 | indexed + filter |
| 历史查询 | 不支持 | queryFilter() |

---

### 类比2：indexed = 邮件标签 📧

#### 生活场景类比：Gmail标签系统

想象Gmail的邮件标签：

**邮件标签：**
- **标签（indexed）**：重要、工作、个人
- **邮件内容（data）**：具体的邮件正文
- **按标签搜索（filter）**：label:工作

```
没有标签（无indexed）：
📧 邮件列表
├── 邮件1（需要逐个查看）
├── 邮件2
└── ...（无法快速筛选）

有标签（有indexed）：
📧 工作 (50)
📧 个人 (30)
📧 重要 (10)
└── 点击即可筛选
```

```solidity
contract EmailSystem {
    // indexed参数就像邮件标签
    event Email(
        address indexed sender,     // 标签：发件人
        address indexed recipient,  // 标签：收件人
        string indexed category,    // 标签：分类（存储哈希）
        string subject,             // 内容：主题
        string body                 // 内容：正文
    );
    
    function sendEmail(
        address to,
        string memory category,
        string memory subject,
        string memory body
    ) external {
        emit Email(msg.sender, to, category, subject, body);
    }
}
```

```javascript
// 按"发件人"筛选
const fromAlice = contract.filters.Email(aliceAddress, null, null);

// 按"收件人"筛选
const toBob = contract.filters.Email(null, bobAddress, null);

// 按"分类"筛选
const workCategory = contract.filters.Email(null, null, ethers.id("work"));
```

---

### 类比总结表

| Event概念 | 生活场景类比 | 前端领域类比 |
|----------|-------------|-------------|
| event声明 | 定义通知模板 | CustomEvent/EventEmitter |
| emit | 发送通知 | dispatchEvent/emit |
| indexed | 通知分类标签 | 不直接对应 |
| logs | 通知历史 | 不直接对应（链上才有） |
| filter | 筛选通知 | 手动过滤 |
| queryFilter | 查询历史通知 | 不直接对应 |

---

## 6. 【反直觉点】

### 误区1：合约可以读取自己发出的事件 ❌

**为什么错？**

Event是**只写**的日志系统，合约内部**无法读取**任何已发出的事件。事件存储在交易收据中，不在合约Storage中。

```solidity
contract EventReadDemo {
    event DataStored(uint256 indexed id, string data);
    
    // ❌ 错误：试图读取事件
    // function getEventData(uint256 id) external view returns (string memory) {
    //     // 无法实现！合约不能读取事件
    // }
    
    // ✅ 正确：需要读取的数据必须存在Storage中
    mapping(uint256 => string) public dataStorage;
    
    function storeData(uint256 id, string memory data) external {
        dataStorage[id] = data;  // 存到Storage供合约读取
        emit DataStored(id, data);  // 发事件供前端监听
    }
}
```

**为什么人们容易这样错？**

事件看起来像是"记录"了某些数据，直觉上会认为可以查询。但EVM设计上事件是单向的——只能写入，不能在合约内读取。

**正确理解：**

| 数据用途 | 使用方式 |
|---------|---------|
| 合约需要读取 | 存到Storage |
| 仅前端需要 | 用Event（更便宜） |
| 两者都需要 | Storage + Event |

---

### 误区2：indexed动态类型可以查到原始值 ❌

**为什么错？**

当动态类型（string、bytes、数组）被标记为indexed时，存储的是**keccak256哈希**，而不是原始值。

```solidity
contract IndexedDynamicType {
    // string作为indexed
    event NameChanged(
        address indexed user,
        string indexed oldName,  // 存储keccak256(oldName)
        string newName           // 存储原始值
    );
    
    function changeName(string memory oldName, string memory newName) external {
        emit NameChanged(msg.sender, oldName, newName);
    }
}
```

```javascript
// 查询indexed string
const filter = contract.filters.NameChanged(
    null,
    ethers.id("Alice")  // 需要提供哈希，不是原始字符串
);

const events = await contract.queryFilter(filter);
events.forEach(event => {
    console.log(event.args.oldName);  // 输出哈希，不是"Alice"
    console.log(event.args.newName);  // 输出"Alice"（非indexed）
});
```

**为什么人们容易这样错？**

直觉上indexed意味着"可查询"，会期望能查到原始值。但出于效率考虑，动态类型indexed只存储哈希。

**正确理解：**

```solidity
event Example(
    uint256 indexed id,      // 可过滤，可获取原值
    address indexed addr,    // 可过滤，可获取原值
    string indexed strHash,  // 可过滤（用哈希），获取的是哈希
    string strValue          // 不可过滤，可获取原值
);
```

**建议**：如果需要动态类型的原始值，不要用indexed，或者同时存储indexed和非indexed版本。

---

### 误区3：Event比Storage便宜很多，应该全用Event ❌

**为什么错？**

Event确实便宜，但有严格的使用限制：
1. 合约内部无法读取事件
2. 事件只能通过链下服务（如The Graph）查询
3. 历史事件查询可能受节点限制

```solidity
contract WhenToUseEvent {
    mapping(address => uint256) public balances;
    
    event BalanceUpdated(address indexed user, uint256 newBalance);
    
    // ❌ 错误：只用Event，合约无法知道余额
    function depositBad() external payable {
        emit BalanceUpdated(msg.sender, msg.value);
        // 下次怎么知道余额是多少？
    }
    
    // ✅ 正确：Storage存状态，Event通知前端
    function depositGood() external payable {
        balances[msg.sender] += msg.value;  // 合约需要读取
        emit BalanceUpdated(msg.sender, balances[msg.sender]);  // 通知前端
    }
}
```

**为什么人们容易这样错？**

看到Event便宜5-10倍，会倾向于用Event替代所有Storage。但忽略了两者的用途完全不同。

**正确理解：**

| 场景 | 使用 | 原因 |
|-----|------|------|
| 合约状态（余额、所有权） | Storage | 合约需要读取 |
| 通知前端 | Event | 便宜、实时 |
| 历史记录（不需要合约读） | Event | 便宜 |
| 需要链上验证的数据 | Storage | Event无法验证 |

---

## 7. 【实战代码】

### 基础实现：完整的代币合约事件

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

/// @title Event完整演示 - ERC20代币事件
contract EventDemo {
    
    // ========== 状态变量 ==========
    
    string public name = "EventDemo Token";
    string public symbol = "EDT";
    uint8 public decimals = 18;
    uint256 public totalSupply;
    
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    
    address public owner;
    bool public paused;
    
    // ========== 事件定义 ==========
    
    // ERC20标准事件
    event Transfer(
        address indexed from,
        address indexed to,
        uint256 value
    );
    
    event Approval(
        address indexed owner,
        address indexed spender,
        uint256 value
    );
    
    // 自定义事件
    event Mint(
        address indexed to,
        uint256 amount,
        uint256 newTotalSupply
    );
    
    event Burn(
        address indexed from,
        uint256 amount,
        uint256 newTotalSupply
    );
    
    event OwnershipTransferred(
        address indexed previousOwner,
        address indexed newOwner
    );
    
    event Paused(address indexed account);
    event Unpaused(address indexed account);
    
    // 带更多信息的事件
    event TransferWithMessage(
        address indexed from,
        address indexed to,
        uint256 value,
        string message,
        uint256 timestamp
    );
    
    // ========== 修饰器 ==========
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    modifier whenNotPaused() {
        require(!paused, "Paused");
        _;
    }
    
    // ========== 构造函数 ==========
    
    constructor(uint256 initialSupply) {
        owner = msg.sender;
        _mint(msg.sender, initialSupply * 10**decimals);
        emit OwnershipTransferred(address(0), msg.sender);
    }
    
    // ========== ERC20功能 ==========
    
    function transfer(address to, uint256 amount) external whenNotPaused returns (bool) {
        _transfer(msg.sender, to, amount);
        return true;
    }
    
    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }
    
    function transferFrom(address from, address to, uint256 amount) external whenNotPaused returns (bool) {
        require(allowance[from][msg.sender] >= amount, "Insufficient allowance");
        allowance[from][msg.sender] -= amount;
        _transfer(from, to, amount);
        return true;
    }
    
    // ========== 扩展功能 ==========
    
    /// @notice 带消息的转账
    function transferWithMessage(
        address to,
        uint256 amount,
        string calldata message
    ) external whenNotPaused returns (bool) {
        _transfer(msg.sender, to, amount);
        emit TransferWithMessage(msg.sender, to, amount, message, block.timestamp);
        return true;
    }
    
    /// @notice 铸造代币
    function mint(address to, uint256 amount) external onlyOwner {
        _mint(to, amount);
    }
    
    /// @notice 销毁代币
    function burn(uint256 amount) external {
        require(balanceOf[msg.sender] >= amount, "Insufficient balance");
        balanceOf[msg.sender] -= amount;
        totalSupply -= amount;
        emit Burn(msg.sender, amount, totalSupply);
        emit Transfer(msg.sender, address(0), amount);
    }
    
    // ========== 管理功能 ==========
    
    function pause() external onlyOwner {
        paused = true;
        emit Paused(msg.sender);
    }
    
    function unpause() external onlyOwner {
        paused = false;
        emit Unpaused(msg.sender);
    }
    
    function transferOwnership(address newOwner) external onlyOwner {
        require(newOwner != address(0), "Invalid address");
        emit OwnershipTransferred(owner, newOwner);
        owner = newOwner;
    }
    
    // ========== 内部函数 ==========
    
    function _transfer(address from, address to, uint256 amount) internal {
        require(from != address(0), "From zero address");
        require(to != address(0), "To zero address");
        require(balanceOf[from] >= amount, "Insufficient balance");
        
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        
        emit Transfer(from, to, amount);
    }
    
    function _mint(address to, uint256 amount) internal {
        require(to != address(0), "Mint to zero address");
        
        totalSupply += amount;
        balanceOf[to] += amount;
        
        emit Mint(to, amount, totalSupply);
        emit Transfer(address(0), to, amount);
    }
}
```

---

### 进阶：前端完整集成

```javascript
// ===== 使用ethers.js v6完整集成 =====

const { ethers } = require('ethers');

// 合约ABI（简化版）
const abi = [
    "event Transfer(address indexed from, address indexed to, uint256 value)",
    "event Approval(address indexed owner, address indexed spender, uint256 value)",
    "event TransferWithMessage(address indexed from, address indexed to, uint256 value, string message, uint256 timestamp)",
    "function transfer(address to, uint256 amount) returns (bool)",
    "function balanceOf(address) view returns (uint256)"
];

class TokenEventListener {
    constructor(contractAddress, providerUrl) {
        this.provider = new ethers.WebSocketProvider(providerUrl);
        this.contract = new ethers.Contract(contractAddress, abi, this.provider);
        this.listeners = [];
    }
    
    // ===== 实时监听 =====
    
    // 监听所有转账
    onTransfer(callback) {
        const listener = (from, to, value, event) => {
            callback({
                from,
                to,
                value: ethers.formatEther(value),
                blockNumber: event.log.blockNumber,
                transactionHash: event.log.transactionHash
            });
        };
        this.contract.on("Transfer", listener);
        this.listeners.push({ event: "Transfer", listener });
    }
    
    // 监听特定地址的转入
    onReceive(address, callback) {
        const filter = this.contract.filters.Transfer(null, address);
        const listener = (from, to, value, event) => {
            callback({
                from,
                value: ethers.formatEther(value),
                transactionHash: event.log.transactionHash
            });
        };
        this.contract.on(filter, listener);
        this.listeners.push({ event: filter, listener });
    }
    
    // 监听特定地址的转出
    onSend(address, callback) {
        const filter = this.contract.filters.Transfer(address, null);
        const listener = (from, to, value, event) => {
            callback({
                to,
                value: ethers.formatEther(value),
                transactionHash: event.log.transactionHash
            });
        };
        this.contract.on(filter, listener);
        this.listeners.push({ event: filter, listener });
    }
    
    // ===== 历史查询 =====
    
    // 获取地址的转账历史
    async getTransferHistory(address, fromBlock = 0) {
        // 转出记录
        const sentFilter = this.contract.filters.Transfer(address, null);
        const sentEvents = await this.contract.queryFilter(sentFilter, fromBlock);
        
        // 转入记录
        const receivedFilter = this.contract.filters.Transfer(null, address);
        const receivedEvents = await this.contract.queryFilter(receivedFilter, fromBlock);
        
        // 合并并排序
        const allEvents = [...sentEvents, ...receivedEvents]
            .sort((a, b) => a.blockNumber - b.blockNumber)
            .map(event => ({
                type: event.args.from === address ? 'SENT' : 'RECEIVED',
                from: event.args.from,
                to: event.args.to,
                value: ethers.formatEther(event.args.value),
                blockNumber: event.blockNumber,
                transactionHash: event.transactionHash
            }));
        
        return allEvents;
    }
    
    // 获取最近的转账事件
    async getRecentTransfers(count = 10) {
        const currentBlock = await this.provider.getBlockNumber();
        const events = await this.contract.queryFilter("Transfer", currentBlock - 10000);
        
        return events
            .slice(-count)
            .reverse()
            .map(event => ({
                from: event.args.from,
                to: event.args.to,
                value: ethers.formatEther(event.args.value),
                blockNumber: event.blockNumber
            }));
    }
    
    // ===== 清理 =====
    
    removeAllListeners() {
        this.listeners.forEach(({ event, listener }) => {
            this.contract.off(event, listener);
        });
        this.listeners = [];
    }
    
    async disconnect() {
        this.removeAllListeners();
        await this.provider.destroy();
    }
}

// ===== 使用示例 =====

async function main() {
    const listener = new TokenEventListener(
        '0x...ContractAddress',
        'wss://eth-mainnet.ws.alchemyapi.io/v2/YOUR_KEY'
    );
    
    // 监听所有转账
    listener.onTransfer((data) => {
        console.log(`转账: ${data.from} -> ${data.to}: ${data.value} ETH`);
    });
    
    // 监听特定地址
    const myAddress = '0x...';
    listener.onReceive(myAddress, (data) => {
        console.log(`收到 ${data.value} ETH 来自 ${data.from}`);
    });
    
    // 查询历史
    const history = await listener.getTransferHistory(myAddress, 15000000);
    console.log('转账历史:', history);
    
    // 程序退出时清理
    process.on('SIGINT', async () => {
        await listener.disconnect();
        process.exit();
    });
}

main();
```

---

## 8. 【面试必问】

### 问题1："为什么使用Event而不是直接存到Storage？Event的优缺点是什么？"

**普通回答（❌ 不出彩）：**

"Event比Storage便宜，可以让前端监听。缺点是合约不能读取。"

**出彩回答（✅ 推荐）：**

> **Event和Storage是两种完全不同的数据存储方式，各有适用场景：**
>
> **Gas成本对比：**
> - Storage写入：约20,000 Gas（新slot）/ 5,000 Gas（修改）
> - Event：约375 Gas + 8 Gas/字节
> - Event便宜约5-10倍
>
> **Event的优势：**
> 1. **成本低**：适合记录大量历史数据
> 2. **实时通知**：前端可以通过WebSocket监听
> 3. **可索引**：indexed参数支持高效过滤查询
> 4. **链下查询**：The Graph等服务可以索引所有事件
>
> **Event的限制：**
> 1. **只写不读**：合约内部无法读取已发出的事件
> 2. **查询依赖节点**：历史事件查询可能受RPC节点限制
> 3. **动态类型indexed存哈希**：无法直接获取原值
>
> **使用场景选择：**
> ```
> 合约需要读取 → Storage
> 仅前端需要 → Event
> 需要链上验证 → Storage
> 历史记录/审计 → Event
> 两者都需要 → Storage + Event
> ```
>
> **实际应用示例：**
> ```solidity
> // Storage：合约需要知道余额
> mapping(address => uint256) public balances;
> 
> // Event：通知前端 + 历史记录
> event Deposit(address indexed user, uint256 amount);
> 
> function deposit() external payable {
>     balances[msg.sender] += msg.value;  // 合约状态
>     emit Deposit(msg.sender, msg.value); // 通知 + 记录
> }
> ```

**为什么这个回答出彩？**
1. ✅ 给出了具体的Gas数据
2. ✅ 列举了优势和限制
3. ✅ 提供了选择指南
4. ✅ 给出了实际代码示例

---

### 问题2："indexed参数的作用是什么？最多能有几个？"

**普通回答（❌ 不出彩）：**

"indexed可以用来过滤事件，最多3个。"

**出彩回答（✅ 推荐）：**

> **indexed参数的作用和技术细节：**
>
> **存储机制：**
> - Event在日志中有`topics`和`data`两部分
> - `topics[0]`：事件签名的keccak256哈希
> - `topics[1-3]`：indexed参数（最多3个）
> - `data`：非indexed参数的ABI编码
>
> **为什么最多3个？**
> - EVM日志设计限制topics数组最多4个元素
> - topics[0]用于事件签名，剩余3个给indexed参数
>
> **indexed的作用：**
> 1. **高效过滤**：可以按indexed参数筛选事件
> 2. **布隆过滤器**：区块头的logsBloom可以快速判断区块是否包含特定事件
> 3. **链下索引**：The Graph等服务可以高效建立索引
>
> **动态类型的特殊处理：**
> ```solidity
> event Example(
>     string indexed name,  // 存储keccak256(name)，不是原值
>     string value          // 存储原值
> );
> ```
> 动态类型（string, bytes, 数组）作为indexed时存储哈希，无法获取原值。
>
> **Gas影响：**
> - 每个indexed参数额外消耗约375 Gas（一个topic）
> - 但带来的查询便利通常值得这个成本
>
> **最佳实践：**
> ```solidity
> event Transfer(
>     address indexed from,    // ✅ 需要按发送者查询
>     address indexed to,      // ✅ 需要按接收者查询
>     uint256 value            // 不需要过滤，放data
> );
> ```

**为什么这个回答出彩？**
1. ✅ 解释了底层存储机制（topics数组）
2. ✅ 说明了3个限制的原因
3. ✅ 提到了动态类型的特殊处理
4. ✅ 给出了最佳实践示例

---

## 9. 【化骨绵掌】

### 卡片1：直觉理解 - Event是什么？ 🎯

**一句话：** Event是智能合约发给外界的"通知"，便宜且可监听。

**举例：**
```solidity
event Transfer(address from, address to, uint256 amount);

function transfer(address to, uint256 amount) external {
    // 转账逻辑...
    emit Transfer(msg.sender, to, amount);  // 发送通知
}
```

**应用：** 前端实时更新、历史记录、审计跟踪。

---

### 卡片2：形式化定义 - Event结构 📐

**一句话：** Event存储在日志中，包含topics（可索引）和data（原始值）两部分。

```
日志结构：
topics[0]: 事件签名哈希
topics[1-3]: indexed参数
data: 非indexed参数
```

**应用：** 理解结构有助于正确设计事件参数。

---

### 卡片3：关键概念 - emit触发 📢

**一句话：** 使用`emit`关键字触发事件，将数据写入交易日志。

**举例：**
```solidity
event Deposited(address user, uint256 amount);

function deposit() external payable {
    emit Deposited(msg.sender, msg.value);
}
```

**应用：** 每个重要状态变化都应该emit事件。

---

### 卡片4：关键概念 - indexed索引 🔍

**一句话：** `indexed`参数存在topics中，可以被过滤查询，最多3个。

**举例：**
```solidity
event Transfer(
    address indexed from,   // 可过滤
    address indexed to,     // 可过滤
    uint256 value           // 不可过滤
);
```

**应用：** 对需要查询的字段使用indexed。

---

### 卡片5：编程实现 - 前端监听 🎧

**一句话：** 使用ethers.js的`on`方法实时监听，`queryFilter`查询历史。

**举例：**
```javascript
// 实时监听
contract.on("Transfer", (from, to, value) => {
    console.log(`${from} -> ${to}: ${value}`);
});

// 历史查询
const events = await contract.queryFilter("Transfer", -1000);
```

**应用：** DApp实时更新、交易历史展示。

---

### 卡片6：对比区分 - Event vs Storage 🆚

**一句话：** Storage用于合约需要读取的状态，Event用于通知和历史记录。

| 特性 | Storage | Event |
|-----|---------|-------|
| 合约可读 | ✅ | ❌ |
| Gas成本 | 高 | 低 |
| 前端可监听 | ❌ | ✅ |
| 可过滤 | ❌ | ✅(indexed) |

**应用：** 两者配合使用，状态存Storage，通知用Event。

---

### 卡片7：进阶理解 - 动态类型indexed ⚠️

**一句话：** string/bytes/数组作为indexed时存储哈希，无法获取原值。

**举例：**
```solidity
event NameChanged(
    string indexed name,  // 存储keccak256(name)
    string value          // 存储原值
);
```

**应用：** 需要原值时不要对动态类型使用indexed。

---

### 卡片8：高级应用 - 过滤查询 🔎

**一句话：** 使用filters创建过滤条件，高效查询特定事件。

**举例：**
```javascript
// 查询from=myAddress的转账
const filter = contract.filters.Transfer(myAddress, null);
const events = await contract.queryFilter(filter, fromBlock);
```

**应用：** 获取用户交易历史、追踪特定地址活动。

---

### 卡片9：最佳实践 - 事件设计 ✅

**一句话：** 每个重要操作发出事件，关键字段用indexed，动态类型谨慎使用indexed。

**举例：**
```solidity
// ✅ 好的设计
event OrderCreated(
    uint256 indexed orderId,    // 按订单ID查询
    address indexed buyer,      // 按买家查询
    uint256 totalPrice,         // 原值
    string description          // 原值
);
```

**应用：** 设计事件时考虑前端查询需求。

---

### 卡片10：总结与延伸 🎓

**一句话：** Event是DApp前后端通信的桥梁，掌握它是构建响应式应用的基础。

**核心要点：**
1. emit触发，on/queryFilter监听
2. indexed可过滤，最多3个
3. 动态类型indexed存哈希
4. 比Storage便宜5-10倍
5. 合约内部无法读取事件

**下一步学习：**
- Storage布局与优化
- The Graph索引服务
- WebSocket实时通信
- 前端状态管理集成

---

## 10. 【一句话总结】

**Event是智能合约的只写日志系统，通过emit触发将数据写入交易日志，indexed参数（最多3个）支持高效过滤查询，Gas成本比Storage低5-10倍，前端通过WebSocket监听实时事件或queryFilter查询历史，是DApp前后端通信和链上审计的核心机制。**

---

## 📚 附录

### 学习检查清单

完成本知识点学习后，你应该能够：

- [ ] 声明和触发事件
- [ ] 正确使用indexed参数
- [ ] 前端监听实时事件
- [ ] 查询历史事件
- [ ] 理解Event和Storage的区别
- [ ] 知道indexed动态类型的限制

### 快速参考卡

**事件声明：**

```solidity
event Transfer(
    address indexed from,
    address indexed to,
    uint256 value
);
```

**触发事件：**

```solidity
emit Transfer(msg.sender, to, amount);
```

**前端监听：**

```javascript
// 实时
contract.on("Transfer", callback);

// 历史
contract.queryFilter("Transfer", fromBlock);

// 过滤
contract.filters.Transfer(from, to);
```

### 下一步学习

推荐按以下顺序继续学习：

1. **Storage优化** - 降低Gas成本
2. **The Graph** - 链下事件索引
3. **合约升级** - 代理模式与事件
4. **安全审计** - 事件日志分析

---

**版本：** v1.0
**创建日期：** 2025-12-07
**作者：** Web3学习助手
**适用人群：** 前端工程师转Web3开发
