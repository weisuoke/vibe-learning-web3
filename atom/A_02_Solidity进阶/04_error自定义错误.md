# Solidity 进阶 - error（自定义错误）

## 1. 【30字核心】

**Error是Solidity 0.8.4+引入的自定义错误机制，通过`error`关键字定义、`revert`触发，比require字符串更省Gas，是现代智能合约的错误处理最佳实践。**

---

## 2. 【第一性原理】

### 什么是第一性原理？

**第一性原理**：回到事物最基本的真理，从源头思考问题

### Error的第一性原理 🎯

#### 1. 最基础的定义

**Error = 带有名称和参数的错误类型**

仅此而已！Error就是给错误一个"名字"和"数据"。

#### 2. 为什么需要自定义Error？

**核心问题：如何在保证错误信息清晰的同时，最小化Gas消耗？**

在Solidity 0.8.4之前，错误处理主要用`require`：
```solidity
require(balance >= amount, "Insufficient balance"); // 字符串存储在链上，消耗Gas
```

问题：
- 错误字符串存储在合约字节码中，增加部署成本
- 每次revert都要ABI编码字符串，增加执行成本
- 字符串不是结构化数据，难以在前端解析

**自定义Error解决了这些问题：**
```solidity
error InsufficientBalance(uint256 available, uint256 required);
// 只存储4字节选择器 + 参数，大幅省Gas
```

#### 3. Error的三层价值

##### 价值1：Gas优化（节省约50%）

**问题**：require字符串在大型合约中累积，显著增加Gas成本。

**解决方案**：Error只存储4字节选择器，参数在运行时编码。

```solidity
// ❌ 旧方式：每个字符消耗Gas
require(balance >= amount, "InsufficientBalance: you have X but need Y");

// ✅ 新方式：只存储4字节选择器
error InsufficientBalance(uint256 available, uint256 required);
if (balance < amount) revert InsufficientBalance(balance, amount);

// Gas对比（近似值）：
// require + 50字符串：~3000 gas
// 自定义error + 2参数：~1500 gas
```

##### 价值2：结构化错误数据

**问题**：字符串错误难以被前端程序化处理。

**解决方案**：Error是结构化类型，前端可以精确解析。

```solidity
// 合约定义
error TransferFailed(address from, address to, uint256 amount, string reason);

// 前端可以精确捕获
try {
    await contract.transfer(to, amount);
} catch (error) {
    if (error.errorName === "TransferFailed") {
        const { from, to, amount, reason } = error.errorArgs;
        console.log(`Transfer from ${from} to ${to} failed: ${reason}`);
    }
}
```

##### 价值3：类型安全与IDE支持

**问题**：字符串错误容易拼写错误，无法获得编译器检查。

**解决方案**：Error是类型，编译器会检查参数匹配。

```solidity
error InvalidAmount(uint256 amount);

// ✅ 编译器检查参数类型
revert InvalidAmount(100);

// ❌ 编译错误：参数类型不匹配
// revert InvalidAmount("100");

// ✅ IDE自动补全和类型提示
```

#### 4. 从第一性原理推导Error设计

**推理链：**

```
1. 前提：智能合约需要向用户反馈错误信息
   ↓
2. 推导：字符串错误消耗大量Gas → 需要更高效的方式
   ↓
3. 推导：错误需要携带上下文数据 → 需要参数化
   ↓
4. 推导：前端需要解析错误 → 需要结构化
   ↓
5. 推导：开发者需要编译器检查 → 需要类型系统
   ↓
6. 推导：如何唯一标识错误 → 4字节选择器（类似函数选择器）
   ↓
7. 推导：如何触发错误 → revert + error
   ↓
8. 最终实现：Solidity custom error
   - error关键字定义
   - revert触发
   - 4字节选择器标识
   - ABI编码参数
```

#### 5. 一句话总结第一性原理

**Error是给错误一个"名字"和"数据结构"，用4字节选择器替代字符串，在保持错误信息丰富的同时大幅降低Gas成本。**

---

## 3. 【3个核心概念】

### 核心概念1：error定义与语法 📋

**一句话定义：** 使用`error`关键字定义自定义错误类型，可以包含参数，参数会在revert时被ABI编码。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

// ===== 基础语法 =====

// 无参数错误
error Unauthorized();

// 带参数错误
error InsufficientBalance(uint256 available, uint256 required);

// 多参数错误
error TransferFailed(
    address from,
    address to,
    uint256 amount,
    string reason
);

// 复杂类型参数
error InvalidOrder(
    uint256 orderId,
    address buyer,
    uint256[] itemIds
);

contract ErrorExample {
    mapping(address => uint256) public balances;
    address public owner;
    
    constructor() {
        owner = msg.sender;
    }
    
    // ===== 使用error =====
    
    function withdraw(uint256 amount) external {
        // 方式1：if + revert
        if (msg.sender != owner) {
            revert Unauthorized();
        }
        
        // 方式2：带参数的revert
        if (balances[msg.sender] < amount) {
            revert InsufficientBalance(balances[msg.sender], amount);
        }
        
        balances[msg.sender] -= amount;
        payable(msg.sender).transfer(amount);
    }
    
    // ===== 与require对比 =====
    
    function withdrawOldStyle(uint256 amount) external {
        // 旧方式：require + 字符串
        require(msg.sender == owner, "Unauthorized");
        require(balances[msg.sender] >= amount, "Insufficient balance");
        
        balances[msg.sender] -= amount;
        payable(msg.sender).transfer(amount);
    }
}

// ===== 全局定义 vs 合约内定义 =====

// 全局定义：可以被多个合约使用
error GlobalError(string message);

contract ContractA {
    // 合约内定义：只在这个合约可见
    error LocalError(uint256 code);
    
    function foo() external pure {
        revert GlobalError("from A");
    }
}

contract ContractB {
    function bar() external pure {
        revert GlobalError("from B"); // ✅ 可以使用全局error
        // revert LocalError(1);      // ❌ 不能使用ContractA的局部error
    }
}
```

**详细解释：**

**Error选择器的计算：**

```solidity
// Error选择器 = keccak256(error签名)的前4字节
// 与函数选择器计算方式相同

error InsufficientBalance(uint256 available, uint256 required);

// 选择器计算：
// keccak256("InsufficientBalance(uint256,uint256)") 的前4字节
// → 0xcf479181

// 当revert时，实际发送的数据：
// 0xcf479181 + abi.encode(available, required)
```

**在智能合约开发中的应用：**

```solidity
// OpenZeppelin风格的错误定义
contract ERC20WithErrors {
    error ERC20InsufficientBalance(address sender, uint256 balance, uint256 needed);
    error ERC20InvalidSender(address sender);
    error ERC20InvalidReceiver(address receiver);
    error ERC20InsufficientAllowance(address spender, uint256 allowance, uint256 needed);
    
    function transfer(address to, uint256 amount) public returns (bool) {
        if (to == address(0)) {
            revert ERC20InvalidReceiver(address(0));
        }
        
        uint256 fromBalance = _balances[msg.sender];
        if (fromBalance < amount) {
            revert ERC20InsufficientBalance(msg.sender, fromBalance, amount);
        }
        
        // ...
    }
}
```

---

### 核心概念2：revert语句与错误触发 🚨

**一句话定义：** `revert`语句用于触发自定义错误，回滚所有状态变更，并返回错误数据给调用者。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

error InsufficientFunds(uint256 available, uint256 required);
error InvalidAddress(address addr);
error AccessDenied(address caller, bytes32 requiredRole);

contract RevertExample {
    mapping(address => uint256) public balances;
    mapping(bytes32 => mapping(address => bool)) public roles;
    
    bytes32 public constant ADMIN_ROLE = keccak256("ADMIN");
    
    // ===== 基本revert用法 =====
    
    function withdraw(uint256 amount) external {
        uint256 balance = balances[msg.sender];
        
        // 标准模式：if + revert
        if (balance < amount) {
            revert InsufficientFunds(balance, amount);
        }
        
        balances[msg.sender] = balance - amount;
        payable(msg.sender).transfer(amount);
    }
    
    // ===== 条件复杂时的revert =====
    
    function transfer(address to, uint256 amount) external {
        // 多个条件检查
        if (to == address(0)) {
            revert InvalidAddress(to);
        }
        
        if (balances[msg.sender] < amount) {
            revert InsufficientFunds(balances[msg.sender], amount);
        }
        
        balances[msg.sender] -= amount;
        balances[to] += amount;
    }
    
    // ===== 在修饰符中使用revert =====
    
    modifier onlyRole(bytes32 role) {
        if (!roles[role][msg.sender]) {
            revert AccessDenied(msg.sender, role);
        }
        _;
    }
    
    function adminFunction() external onlyRole(ADMIN_ROLE) {
        // 只有ADMIN可以调用
    }
    
    // ===== 在内部函数中使用revert =====
    
    function _checkBalance(address account, uint256 amount) internal view {
        if (balances[account] < amount) {
            revert InsufficientFunds(balances[account], amount);
        }
    }
    
    function batchWithdraw(uint256[] calldata amounts) external {
        uint256 total = 0;
        for (uint256 i = 0; i < amounts.length; i++) {
            total += amounts[i];
        }
        
        _checkBalance(msg.sender, total); // 复用检查逻辑
        
        balances[msg.sender] -= total;
        payable(msg.sender).transfer(total);
    }
}
```

**详细解释：**

**revert vs require vs assert：**

```solidity
contract ComparisonExample {
    error CustomError(uint256 value);
    
    // revert：显式触发错误，返回剩余Gas
    function useRevert(uint256 x) external pure {
        if (x == 0) {
            revert CustomError(x);
        }
    }
    
    // require：条件检查失败时revert，返回剩余Gas
    function useRequire(uint256 x) external pure {
        require(x != 0, "x cannot be zero");
    }
    
    // assert：用于检查不变量，失败时消耗所有Gas（Solidity 0.8+会返回剩余Gas）
    function useAssert(uint256 x) external pure {
        assert(x != 0); // 应该用于"永远不应该发生"的情况
    }
}

// Gas对比（近似值）
// revert CustomError()：~200 gas + 参数编码
// require("message")：~200 gas + 字符串存储/编码
// assert()：0.8+返回剩余Gas，之前消耗所有
```

**在智能合约开发中的应用：**

```solidity
// 实际项目中的错误处理模式
contract VaultWithErrors {
    error VaultLocked();
    error InsufficientShares(uint256 available, uint256 requested);
    error SlippageExceeded(uint256 expected, uint256 actual);
    error DeadlineExpired(uint256 deadline, uint256 currentTime);
    
    bool public locked;
    mapping(address => uint256) public shares;
    
    function withdraw(
        uint256 shareAmount,
        uint256 minAssets,
        uint256 deadline
    ) external returns (uint256 assets) {
        // 检查1：时间限制
        if (block.timestamp > deadline) {
            revert DeadlineExpired(deadline, block.timestamp);
        }
        
        // 检查2：锁定状态
        if (locked) {
            revert VaultLocked();
        }
        
        // 检查3：份额余额
        if (shares[msg.sender] < shareAmount) {
            revert InsufficientShares(shares[msg.sender], shareAmount);
        }
        
        // 执行提款
        shares[msg.sender] -= shareAmount;
        assets = _convertToAssets(shareAmount);
        
        // 检查4：滑点保护
        if (assets < minAssets) {
            revert SlippageExceeded(minAssets, assets);
        }
        
        // 转账资产
        payable(msg.sender).transfer(assets);
    }
    
    function _convertToAssets(uint256 shareAmount) internal pure returns (uint256) {
        return shareAmount; // 简化示例
    }
}
```

---

### 核心概念3：前端错误捕获与解析 🖥️

**一句话定义：** 前端可以通过ethers.js/viem解析自定义错误的选择器和参数，实现精确的错误处理。

```javascript
// ===== 使用ethers.js v6捕获自定义错误 =====

import { ethers } from 'ethers';

// 合约ABI（包含error定义）
const abi = [
    "error InsufficientBalance(uint256 available, uint256 required)",
    "error Unauthorized()",
    "error TransferFailed(address from, address to, uint256 amount)",
    "function withdraw(uint256 amount)"
];

const contract = new ethers.Contract(address, abi, signer);

async function safeWithdraw(amount) {
    try {
        const tx = await contract.withdraw(amount);
        await tx.wait();
        console.log("Withdrawal successful");
    } catch (error) {
        // ethers.js v6 会自动解析自定义错误
        if (error.code === 'CALL_EXCEPTION') {
            const errorData = error.data;
            
            // 尝试解析错误
            try {
                const iface = new ethers.Interface(abi);
                const decodedError = iface.parseError(errorData);
                
                if (decodedError) {
                    console.log("Error name:", decodedError.name);
                    console.log("Error args:", decodedError.args);
                    
                    // 根据错误类型显示友好消息
                    switch (decodedError.name) {
                        case "InsufficientBalance":
                            const [available, required] = decodedError.args;
                            alert(`余额不足！当前: ${available}, 需要: ${required}`);
                            break;
                        case "Unauthorized":
                            alert("您没有权限执行此操作");
                            break;
                        case "TransferFailed":
                            const [from, to, amt] = decodedError.args;
                            alert(`转账失败: ${from} -> ${to}, 金额: ${amt}`);
                            break;
                        default:
                            alert(`操作失败: ${decodedError.name}`);
                    }
                }
            } catch (parseError) {
                // 无法解析的错误
                console.error("Unknown error:", errorData);
            }
        } else {
            // 其他类型的错误
            console.error("Error:", error);
        }
    }
}
```

```typescript
// ===== 使用viem捕获自定义错误 =====

import { createPublicClient, http, decodeErrorResult } from 'viem';
import { mainnet } from 'viem/chains';

// 定义error ABI
const errorAbi = [
    {
        type: 'error',
        name: 'InsufficientBalance',
        inputs: [
            { name: 'available', type: 'uint256' },
            { name: 'required', type: 'uint256' }
        ]
    },
    {
        type: 'error',
        name: 'Unauthorized',
        inputs: []
    }
] as const;

async function handleContractError(errorData: `0x${string}`) {
    try {
        const decoded = decodeErrorResult({
            abi: errorAbi,
            data: errorData
        });
        
        console.log('Error name:', decoded.errorName);
        console.log('Error args:', decoded.args);
        
        // 类型安全的错误处理
        if (decoded.errorName === 'InsufficientBalance') {
            const { available, required } = decoded.args as { 
                available: bigint; 
                required: bigint 
            };
            return `Insufficient balance: have ${available}, need ${required}`;
        }
        
        if (decoded.errorName === 'Unauthorized') {
            return 'You are not authorized';
        }
        
        return `Unknown error: ${decoded.errorName}`;
    } catch {
        return 'Failed to decode error';
    }
}
```

**详细解释：**

**错误数据的结构：**

```
revert InsufficientBalance(1000, 5000);

错误数据（十六进制）：
0xcf479181                               // 4字节选择器
0000000000000000000000000000000000000000000000000000000000003e8  // available = 1000
0000000000000000000000000000000000000000000000000000000000001388 // required = 5000

选择器计算：
keccak256("InsufficientBalance(uint256,uint256)")[0:4] = 0xcf479181
```

**在智能合约开发中的应用：**

```solidity
// 合约端：定义丰富的错误类型
contract DEXWithErrors {
    error SwapFailed(
        address tokenIn,
        address tokenOut,
        uint256 amountIn,
        uint256 amountOutMin,
        uint256 actualAmountOut,
        string reason
    );
    
    error LiquidityInsufficient(
        address token,
        uint256 requested,
        uint256 available
    );
    
    error PriceImpactTooHigh(
        uint256 impactBps,  // 基点，1 bps = 0.01%
        uint256 maxAllowedBps
    );
    
    function swap(
        address tokenIn,
        address tokenOut,
        uint256 amountIn,
        uint256 amountOutMin
    ) external returns (uint256 amountOut) {
        // 检查流动性
        uint256 liquidity = _getLiquidity(tokenOut);
        if (liquidity < amountOutMin) {
            revert LiquidityInsufficient(tokenOut, amountOutMin, liquidity);
        }
        
        // 计算输出
        amountOut = _calculateOutput(tokenIn, tokenOut, amountIn);
        
        // 检查滑点
        if (amountOut < amountOutMin) {
            revert SwapFailed(
                tokenIn,
                tokenOut,
                amountIn,
                amountOutMin,
                amountOut,
                "Slippage exceeded"
            );
        }
        
        // 检查价格影响
        uint256 priceImpact = _calculatePriceImpact(amountIn, amountOut);
        if (priceImpact > 500) { // 5%
            revert PriceImpactTooHigh(priceImpact, 500);
        }
        
        // 执行交换
        _executeSwap(tokenIn, tokenOut, amountIn, amountOut);
        
        return amountOut;
    }
    
    function _getLiquidity(address) internal pure returns (uint256) { return 10000; }
    function _calculateOutput(address, address, uint256 amountIn) internal pure returns (uint256) { return amountIn; }
    function _calculatePriceImpact(uint256, uint256) internal pure returns (uint256) { return 100; }
    function _executeSwap(address, address, uint256, uint256) internal {}
}
```

---

## 4. 【最小可用】

掌握以下内容，就能在智能合约开发中正确使用自定义错误：

### 4.1 定义自定义错误

```solidity
// 无参数错误
error Unauthorized();

// 带参数错误
error InsufficientBalance(uint256 available, uint256 required);

// 多参数错误
error TransferFailed(address from, address to, uint256 amount, string reason);
```

---

### 4.2 触发错误

```solidity
function withdraw(uint256 amount) external {
    // 模式：if + revert
    if (msg.sender != owner) {
        revert Unauthorized();
    }
    
    if (balances[msg.sender] < amount) {
        revert InsufficientBalance(balances[msg.sender], amount);
    }
    
    // 执行逻辑...
}
```

---

### 4.3 在修饰符中使用

```solidity
error NotOwner(address caller, address owner);

modifier onlyOwner() {
    if (msg.sender != owner) {
        revert NotOwner(msg.sender, owner);
    }
    _;
}
```

---

### 4.4 前端捕获错误（ethers.js）

```javascript
try {
    await contract.withdraw(amount);
} catch (error) {
    if (error.code === 'CALL_EXCEPTION') {
        const iface = new ethers.Interface(abi);
        const decoded = iface.parseError(error.data);
        console.log(`Error: ${decoded.name}`, decoded.args);
    }
}
```

---

### 4.5 require与error的选择

```solidity
// 简单检查：使用require（代码更简洁）
require(amount > 0, "Amount must be positive");

// 复杂错误/需要参数：使用自定义error（更省Gas，更易解析）
if (balance < amount) {
    revert InsufficientBalance(balance, amount);
}
```

---

**这些知识足以：**
- ✅ 定义和使用自定义错误
- ✅ 替代require字符串，优化Gas
- ✅ 在前端正确捕获和解析错误
- ✅ 设计清晰的错误处理架构
- ✅ 遵循现代Solidity最佳实践

---

## 5. 【1个类比】

### 类比1：Error = HTTP状态码 + 响应体 🌐

#### 生活场景类比：Error = 医院诊断报告

想象你去医院看病：

**旧方式（require字符串）：**
- 医生只说"你生病了"
- 没有具体诊断，没有检查数据
- 你不知道具体什么问题

**新方式（自定义Error）：**
- 医生给你一份诊断报告
- 包含：疾病名称（Error名）、检查数据（参数）、建议（reason）
- 你能精确了解问题

```
旧方式诊断：
"检查不通过" ← 就这么一句话

新方式诊断报告（Error）：
{
  诊断名称: "血压异常",
  参数: {
    收缩压: 160,    // available
    正常上限: 120   // required
  },
  建议: "需要降压治疗"
}
```

**举例：**
```solidity
// 旧方式：只有一句话
require(bloodPressure <= 120, "Blood pressure check failed");

// 新方式：结构化诊断报告
error HighBloodPressure(uint256 actual, uint256 maxNormal);

if (bloodPressure > 120) {
    revert HighBloodPressure(bloodPressure, 120);
    // 返回：诊断名 + 实际值 + 正常值
}
```

---

#### 前端领域类比：Error = HTTP状态码 + JSON响应

如果你熟悉REST API，自定义Error就像HTTP错误响应：

```javascript
// HTTP错误响应
{
    "status": 400,              // 状态码 → Error选择器
    "error": "INSUFFICIENT_BALANCE",  // 错误类型 → Error名称
    "data": {                   // 错误数据 → Error参数
        "available": 1000,
        "required": 5000
    },
    "message": "Balance too low"
}

// 对应的Solidity Error
error InsufficientBalance(uint256 available, uint256 required);
revert InsufficientBalance(1000, 5000);
```

**代码对比：**

```javascript
// 前端API错误处理
async function apiCall() {
    const response = await fetch('/api/withdraw', { ... });
    
    if (!response.ok) {
        const error = await response.json();
        
        switch (error.error) {
            case 'INSUFFICIENT_BALANCE':
                alert(`余额不足：有 ${error.data.available}，需要 ${error.data.required}`);
                break;
            case 'UNAUTHORIZED':
                alert('没有权限');
                break;
        }
    }
}
```

```javascript
// 智能合约错误处理
async function contractCall() {
    try {
        await contract.withdraw(amount);
    } catch (error) {
        const decoded = iface.parseError(error.data);
        
        switch (decoded.name) {
            case 'InsufficientBalance':
                alert(`余额不足：有 ${decoded.args.available}，需要 ${decoded.args.required}`);
                break;
            case 'Unauthorized':
                alert('没有权限');
                break;
        }
    }
}
```

**对比表：**

| 概念 | HTTP API | Solidity Error |
|-----|----------|----------------|
| 错误标识 | 状态码 (400, 401...) | 选择器 (4字节) |
| 错误类型 | error字段 | Error名称 |
| 错误数据 | JSON body | ABI编码参数 |
| 解析方式 | JSON.parse | ABI.decode |

---

### 类比2：require vs error = console.log vs 结构化日志 📊

#### 生活场景类比：错误报告方式

想象你是一个工厂的质检员：

**旧方式（require）：**
```
检查失败：产品不合格
```
- 哪个产品？什么问题？都不知道

**新方式（error）：**
```
检查报告 #QC2024001
产品ID: P12345
问题类型: 尺寸超标
实际值: 105mm
标准值: 100mm ± 2mm
```
- 所有信息一目了然

---

#### 前端领域类比：console.log vs 结构化日志

```javascript
// 旧方式：字符串日志（难以分析）
console.log("Error: user " + userId + " failed to withdraw " + amount);

// 新方式：结构化日志（易于分析）
logger.error({
    event: 'WITHDRAWAL_FAILED',
    userId: userId,
    amount: amount,
    reason: 'INSUFFICIENT_BALANCE',
    available: balance
});
```

```solidity
// 旧方式：字符串错误
require(balance >= amount, "Insufficient balance");

// 新方式：结构化错误
error WithdrawalFailed(
    address user,
    uint256 amount,
    string reason,
    uint256 available
);

if (balance < amount) {
    revert WithdrawalFailed(msg.sender, amount, "INSUFFICIENT_BALANCE", balance);
}
```

---

### 类比3：Gas优化 = 压缩算法 🗜️

#### 生活场景类比：快递单号 vs 完整地址

**旧方式（require字符串）：**
```
收件地址：北京市朝阳区建国路88号SOHO现代城B座2105室
```
- 每个字都要存储，占用空间大

**新方式（error选择器）：**
```
快递单号：SF1234567890
```
- 4字节编号对应完整信息
- 需要时再查询完整地址

```solidity
// 字符串：每个字符都存储在字节码中
require(false, "InsufficientBalance: available=1000, required=5000");
// 存储：~60字节

// Error：只存4字节选择器
error InsufficientBalance(uint256 available, uint256 required);
revert InsufficientBalance(1000, 5000);
// 存储：4字节选择器 + 运行时参数编码
```

---

### 类比总结表

| Solidity概念 | 生活场景类比 | 前端领域类比 | 核心相似性 |
|-------------|-------------|-------------|-----------|
| 自定义Error | 医院诊断报告 | HTTP错误响应 | 结构化错误信息 |
| Error参数 | 检查数据 | JSON数据字段 | 携带上下文 |
| Error选择器 | 诊断代码 | HTTP状态码 | 唯一标识 |
| require字符串 | "检查不通过" | console.log | 非结构化 |
| Gas优化 | 快递单号vs完整地址 | 压缩算法 | 用ID代替全文 |
| 前端解析 | 读取诊断报告 | 解析JSON | 提取结构化数据 |

---

## 6. 【反直觉点】

### 误区1：自定义Error总是比require更省Gas ❌

**为什么错？**

很多人认为：只要使用自定义Error，就一定比require更省Gas。

**实际情况：**

- **部署Gas**：Error确实更省（不存储字符串）
- **执行Gas**：取决于参数数量和类型

```solidity
// 场景1：简单检查 - Gas差异不大
require(x > 0, ""); // ~200 gas
if (x == 0) revert ZeroValue(); // ~200 gas

// 场景2：短字符串 vs 多参数Error - 可能Error更贵
require(x > 0, "!x"); // 短字符串，Gas较低
error DetailedError(uint256 a, uint256 b, uint256 c, address d);
revert DetailedError(1, 2, 3, addr); // 参数编码消耗Gas

// 场景3：长字符串 - Error明显更省
require(false, "This is a very long error message that explains everything");
// 字符串存储在字节码中，部署Gas高

error SimpleError();
revert SimpleError();
// 只有4字节选择器，部署Gas低
```

**为什么人们容易这样错？**

因为文档通常说"Error更省Gas"，但没有说明具体场景和条件。

**正确理解：**

```solidity
// 选择策略：
// 1. 简单无参数检查：require短字符串或Error都可以
// 2. 需要携带数据：使用Error（结构化，易解析）
// 3. 长错误消息：使用Error（避免存储字符串）
// 4. 高频调用的检查：使用Error（运行时Gas更优）

// 实际项目中：
// - 优先使用Error（现代最佳实践）
// - 只在极简单场景用require
```

---

### 误区2：Error只能在revert时使用 ❌

**为什么错？**

很多人认为：Error只能通过`revert`触发，不能在其他地方使用。

**实际情况：**

Error是一种类型，可以在Interface中声明、在继承中使用：

```solidity
// Interface中声明Error
interface IVault {
    error InsufficientBalance(uint256 available, uint256 required);
    error VaultLocked();
    
    function withdraw(uint256 amount) external;
}

// 实现接口的合约可以使用这些Error
contract Vault is IVault {
    function withdraw(uint256 amount) external override {
        if (locked) revert VaultLocked();
        if (balances[msg.sender] < amount) {
            revert InsufficientBalance(balances[msg.sender], amount);
        }
        // ...
    }
}

// Error可以被继承
contract BaseContract {
    error BaseError(uint256 code);
}

contract DerivedContract is BaseContract {
    function foo() external pure {
        revert BaseError(1); // 使用继承的Error
    }
}
```

**为什么人们容易这样错？**

因为Error最常见的用法就是`revert`，容易忽略它作为类型的其他特性。

**正确理解：**

```solidity
// Error是类型，可以：
// 1. 在Interface中声明
// 2. 被继承
// 3. 全局定义
// 4. 在库中定义

// 全局Error
error GlobalError(string message);

// 库中的Error
library SafeMath {
    error MathOverflow(uint256 a, uint256 b);
    
    function add(uint256 a, uint256 b) internal pure returns (uint256) {
        unchecked {
            uint256 c = a + b;
            if (c < a) revert MathOverflow(a, b);
            return c;
        }
    }
}
```

---

### 误区3：所有require都应该改成Error ❌

**为什么错？**

很多人认为：既然Error更好，就应该把所有require都改成Error。

**实际情况：**

有些场景require更合适：

```solidity
contract Example {
    // ✅ 适合用Error的场景：
    // - 需要携带参数数据
    // - 前端需要精确解析
    // - 复杂的错误类型
    
    error InsufficientBalance(uint256 available, uint256 required);
    
    function withdraw(uint256 amount) external {
        if (balances[msg.sender] < amount) {
            revert InsufficientBalance(balances[msg.sender], amount);
        }
    }
    
    // ✅ 适合用require的场景：
    // - 简单的输入验证
    // - 不需要携带数据
    // - 开发/调试阶段
    
    function setPrice(uint256 price) external {
        require(price > 0, "Price must be positive"); // 简单明了
        require(price < 1e30, "Price too high");      // 快速理解
    }
    
    // ✅ 混合使用
    function complexFunction(uint256 a, uint256 b) external {
        // 简单检查用require
        require(a > 0 && b > 0, "Invalid input");
        
        // 复杂业务错误用Error
        if (a > maxAllowed) {
            revert ExceedsLimit(a, maxAllowed);
        }
    }
}
```

**为什么人们容易这样错？**

因为追求"最佳实践"而忽略了代码可读性和开发效率。

**正确理解：**

```solidity
// 选择原则：
// 1. 简单输入验证 → require（代码简洁）
// 2. 需要参数数据 → Error（结构化）
// 3. 前端需解析 → Error（易处理）
// 4. 开发调试 → require（快速定位）
// 5. 生产优化 → Error（省Gas）

// 团队约定示例：
// - 外部函数的业务错误 → Error
// - 内部函数的断言检查 → require或assert
// - 参数验证 → 视情况而定
```

---

## 7. 【实战代码】

### 基础实现：完整的错误处理示例

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

// ===== 1. 全局Error定义 =====

error ZeroAddress();
error ZeroAmount();
error DeadlineExpired(uint256 deadline, uint256 currentTime);

// ===== 2. 带完整错误处理的代币合约 =====

contract TokenWithErrors {
    // ===== Error定义 =====
    error InsufficientBalance(address account, uint256 available, uint256 required);
    error InsufficientAllowance(address spender, uint256 available, uint256 required);
    error TransferToZeroAddress();
    error ApproveToZeroAddress();
    error MintToZeroAddress();
    error BurnExceedsBalance(address account, uint256 balance, uint256 amount);
    error Unauthorized(address caller);
    error Paused();
    
    // ===== 状态变量 =====
    string public name;
    string public symbol;
    uint8 public decimals = 18;
    uint256 public totalSupply;
    address public owner;
    bool public paused;
    
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    
    // ===== 事件 =====
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    
    // ===== 构造函数 =====
    constructor(string memory _name, string memory _symbol) {
        name = _name;
        symbol = _symbol;
        owner = msg.sender;
    }
    
    // ===== 修饰符 =====
    modifier onlyOwner() {
        if (msg.sender != owner) {
            revert Unauthorized(msg.sender);
        }
        _;
    }
    
    modifier whenNotPaused() {
        if (paused) {
            revert Paused();
        }
        _;
    }
    
    // ===== 核心函数 =====
    
    function transfer(address to, uint256 amount) external whenNotPaused returns (bool) {
        if (to == address(0)) {
            revert TransferToZeroAddress();
        }
        
        uint256 senderBalance = balanceOf[msg.sender];
        if (senderBalance < amount) {
            revert InsufficientBalance(msg.sender, senderBalance, amount);
        }
        
        unchecked {
            balanceOf[msg.sender] = senderBalance - amount;
            balanceOf[to] += amount;
        }
        
        emit Transfer(msg.sender, to, amount);
        return true;
    }
    
    function approve(address spender, uint256 amount) external returns (bool) {
        if (spender == address(0)) {
            revert ApproveToZeroAddress();
        }
        
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }
    
    function transferFrom(
        address from,
        address to,
        uint256 amount
    ) external whenNotPaused returns (bool) {
        if (to == address(0)) {
            revert TransferToZeroAddress();
        }
        
        uint256 currentAllowance = allowance[from][msg.sender];
        if (currentAllowance < amount) {
            revert InsufficientAllowance(msg.sender, currentAllowance, amount);
        }
        
        uint256 fromBalance = balanceOf[from];
        if (fromBalance < amount) {
            revert InsufficientBalance(from, fromBalance, amount);
        }
        
        unchecked {
            allowance[from][msg.sender] = currentAllowance - amount;
            balanceOf[from] = fromBalance - amount;
            balanceOf[to] += amount;
        }
        
        emit Transfer(from, to, amount);
        return true;
    }
    
    // ===== 管理函数 =====
    
    function mint(address to, uint256 amount) external onlyOwner {
        if (to == address(0)) {
            revert MintToZeroAddress();
        }
        if (amount == 0) {
            revert ZeroAmount();
        }
        
        totalSupply += amount;
        balanceOf[to] += amount;
        emit Transfer(address(0), to, amount);
    }
    
    function burn(uint256 amount) external {
        uint256 accountBalance = balanceOf[msg.sender];
        if (accountBalance < amount) {
            revert BurnExceedsBalance(msg.sender, accountBalance, amount);
        }
        
        unchecked {
            balanceOf[msg.sender] = accountBalance - amount;
            totalSupply -= amount;
        }
        
        emit Transfer(msg.sender, address(0), amount);
    }
    
    function pause() external onlyOwner {
        paused = true;
    }
    
    function unpause() external onlyOwner {
        paused = false;
    }
}

// ===== 3. 测试合约 =====

contract TokenTest {
    TokenWithErrors public token;
    
    constructor() {
        token = new TokenWithErrors("Test Token", "TEST");
        token.mint(address(this), 1000 ether);
    }
    
    // 测试InsufficientBalance错误
    function testInsufficientBalance() external {
        // 尝试转账超过余额的金额
        token.transfer(address(1), 2000 ether);
        // 预期触发：InsufficientBalance(address(this), 1000 ether, 2000 ether)
    }
    
    // 测试TransferToZeroAddress错误
    function testTransferToZero() external {
        token.transfer(address(0), 100 ether);
        // 预期触发：TransferToZeroAddress()
    }
    
    // 测试Unauthorized错误
    function testUnauthorized() external {
        token.mint(address(1), 100 ether);
        // 预期触发：Unauthorized(address(this))
    }
}
```

---

### 进阶：DeFi场景的错误处理

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title SwapRouter with comprehensive error handling
 * @dev 展示DeFi场景中的错误处理最佳实践
 */
contract SwapRouter {
    // ===== Error定义（按功能分组）=====
    
    // 通用错误
    error ZeroAddress();
    error ZeroAmount();
    error InvalidPath();
    error DeadlineExpired(uint256 deadline, uint256 currentTime);
    
    // 余额/授权错误
    error InsufficientInputAmount(uint256 available, uint256 required);
    error InsufficientOutputAmount(uint256 actual, uint256 minimum);
    error InsufficientLiquidity(address token, uint256 required, uint256 available);
    
    // 交易错误
    error SwapFailed(address tokenIn, address tokenOut, string reason);
    error PriceImpactTooHigh(uint256 impactBps, uint256 maxBps);
    error SlippageExceeded(uint256 expected, uint256 actual);
    
    // 权限错误
    error Unauthorized(address caller);
    error PoolNotWhitelisted(address pool);
    
    // ===== 状态变量 =====
    address public owner;
    mapping(address => bool) public whitelistedPools;
    uint256 public maxPriceImpactBps = 500; // 5%
    
    // ===== 事件 =====
    event Swapped(
        address indexed user,
        address indexed tokenIn,
        address indexed tokenOut,
        uint256 amountIn,
        uint256 amountOut
    );
    
    constructor() {
        owner = msg.sender;
    }
    
    // ===== 核心交换函数 =====
    
    function swapExactTokensForTokens(
        uint256 amountIn,
        uint256 amountOutMin,
        address[] calldata path,
        address to,
        uint256 deadline
    ) external returns (uint256[] memory amounts) {
        // 检查1：截止时间
        if (block.timestamp > deadline) {
            revert DeadlineExpired(deadline, block.timestamp);
        }
        
        // 检查2：路径有效性
        if (path.length < 2) {
            revert InvalidPath();
        }
        
        // 检查3：接收地址
        if (to == address(0)) {
            revert ZeroAddress();
        }
        
        // 检查4：输入金额
        if (amountIn == 0) {
            revert ZeroAmount();
        }
        
        // 计算输出金额
        amounts = _getAmountsOut(amountIn, path);
        
        // 检查5：输出金额满足最小要求
        uint256 finalAmount = amounts[amounts.length - 1];
        if (finalAmount < amountOutMin) {
            revert InsufficientOutputAmount(finalAmount, amountOutMin);
        }
        
        // 检查6：价格影响
        uint256 priceImpact = _calculatePriceImpact(amountIn, finalAmount, path);
        if (priceImpact > maxPriceImpactBps) {
            revert PriceImpactTooHigh(priceImpact, maxPriceImpactBps);
        }
        
        // 执行交换
        _executeSwap(path, amounts, to);
        
        emit Swapped(msg.sender, path[0], path[path.length - 1], amountIn, finalAmount);
        
        return amounts;
    }
    
    function swapTokensForExactTokens(
        uint256 amountOut,
        uint256 amountInMax,
        address[] calldata path,
        address to,
        uint256 deadline
    ) external returns (uint256[] memory amounts) {
        if (block.timestamp > deadline) {
            revert DeadlineExpired(deadline, block.timestamp);
        }
        
        if (path.length < 2) {
            revert InvalidPath();
        }
        
        amounts = _getAmountsIn(amountOut, path);
        uint256 requiredInput = amounts[0];
        
        if (requiredInput > amountInMax) {
            revert InsufficientInputAmount(amountInMax, requiredInput);
        }
        
        _executeSwap(path, amounts, to);
        
        return amounts;
    }
    
    // ===== 内部函数 =====
    
    function _getAmountsOut(
        uint256 amountIn,
        address[] memory path
    ) internal view returns (uint256[] memory amounts) {
        amounts = new uint256[](path.length);
        amounts[0] = amountIn;
        
        for (uint256 i = 0; i < path.length - 1; i++) {
            (uint256 reserveIn, uint256 reserveOut) = _getReserves(path[i], path[i + 1]);
            
            if (reserveOut == 0) {
                revert InsufficientLiquidity(path[i + 1], amounts[i], reserveOut);
            }
            
            amounts[i + 1] = _getAmountOut(amounts[i], reserveIn, reserveOut);
        }
    }
    
    function _getAmountsIn(
        uint256 amountOut,
        address[] memory path
    ) internal view returns (uint256[] memory amounts) {
        amounts = new uint256[](path.length);
        amounts[path.length - 1] = amountOut;
        
        for (uint256 i = path.length - 1; i > 0; i--) {
            (uint256 reserveIn, uint256 reserveOut) = _getReserves(path[i - 1], path[i]);
            amounts[i - 1] = _getAmountIn(amounts[i], reserveIn, reserveOut);
        }
    }
    
    function _getAmountOut(
        uint256 amountIn,
        uint256 reserveIn,
        uint256 reserveOut
    ) internal pure returns (uint256) {
        uint256 amountInWithFee = amountIn * 997;
        uint256 numerator = amountInWithFee * reserveOut;
        uint256 denominator = reserveIn * 1000 + amountInWithFee;
        return numerator / denominator;
    }
    
    function _getAmountIn(
        uint256 amountOut,
        uint256 reserveIn,
        uint256 reserveOut
    ) internal pure returns (uint256) {
        uint256 numerator = reserveIn * amountOut * 1000;
        uint256 denominator = (reserveOut - amountOut) * 997;
        return numerator / denominator + 1;
    }
    
    function _getReserves(
        address tokenA,
        address tokenB
    ) internal pure returns (uint256 reserveA, uint256 reserveB) {
        // 简化实现
        return (1000000 ether, 1000000 ether);
    }
    
    function _calculatePriceImpact(
        uint256 amountIn,
        uint256 amountOut,
        address[] memory path
    ) internal pure returns (uint256) {
        // 简化实现：返回固定值
        return 100; // 1%
    }
    
    function _executeSwap(
        address[] memory path,
        uint256[] memory amounts,
        address to
    ) internal {
        // 简化实现
    }
}
```

---

### 前端错误处理完整示例

```typescript
// ===== TypeScript/ethers.js v6 完整错误处理 =====

import { ethers } from 'ethers';

// Error ABI定义
const ERROR_ABI = [
    "error ZeroAddress()",
    "error ZeroAmount()",
    "error DeadlineExpired(uint256 deadline, uint256 currentTime)",
    "error InsufficientInputAmount(uint256 available, uint256 required)",
    "error InsufficientOutputAmount(uint256 actual, uint256 minimum)",
    "error PriceImpactTooHigh(uint256 impactBps, uint256 maxBps)",
    "error SwapFailed(address tokenIn, address tokenOut, string reason)"
];

// 错误消息映射
const ERROR_MESSAGES: Record<string, (args: any) => string> = {
    'ZeroAddress': () => '地址不能为零',
    'ZeroAmount': () => '金额不能为零',
    'DeadlineExpired': (args) => 
        `交易已过期：截止时间 ${new Date(Number(args.deadline) * 1000).toLocaleString()}`,
    'InsufficientInputAmount': (args) => 
        `输入金额不足：有 ${ethers.formatEther(args.available)}，需要 ${ethers.formatEther(args.required)}`,
    'InsufficientOutputAmount': (args) => 
        `输出金额不足：实际 ${ethers.formatEther(args.actual)}，最小要求 ${ethers.formatEther(args.minimum)}`,
    'PriceImpactTooHigh': (args) => 
        `价格影响过高：${Number(args.impactBps) / 100}%（最大允许 ${Number(args.maxBps) / 100}%）`,
    'SwapFailed': (args) => 
        `交换失败：${args.reason}`
};

// 错误解析函数
function parseContractError(error: any): { name: string; message: string; args: any } | null {
    try {
        // 获取错误数据
        let errorData: string | undefined;
        
        if (error.data) {
            errorData = error.data;
        } else if (error.error?.data) {
            errorData = error.error.data;
        } else if (error.transaction) {
            // 尝试从交易获取
            errorData = error.transaction.data;
        }
        
        if (!errorData || errorData === '0x') {
            return null;
        }
        
        // 创建接口并解析
        const iface = new ethers.Interface(ERROR_ABI);
        const decoded = iface.parseError(errorData);
        
        if (!decoded) {
            return null;
        }
        
        // 构建参数对象
        const args: Record<string, any> = {};
        decoded.fragment.inputs.forEach((input, index) => {
            args[input.name] = decoded.args[index];
        });
        
        // 生成友好消息
        const messageGenerator = ERROR_MESSAGES[decoded.name];
        const message = messageGenerator ? messageGenerator(args) : `错误：${decoded.name}`;
        
        return {
            name: decoded.name,
            message,
            args
        };
    } catch {
        return null;
    }
}

// 使用示例
async function executeSwap(
    contract: ethers.Contract,
    amountIn: bigint,
    amountOutMin: bigint,
    path: string[],
    to: string,
    deadline: number
) {
    try {
        const tx = await contract.swapExactTokensForTokens(
            amountIn,
            amountOutMin,
            path,
            to,
            deadline
        );
        
        const receipt = await tx.wait();
        console.log('交换成功！', receipt.hash);
        return receipt;
        
    } catch (error: any) {
        // 尝试解析自定义错误
        const parsed = parseContractError(error);
        
        if (parsed) {
            console.error(`合约错误 [${parsed.name}]:`, parsed.message);
            
            // 根据错误类型执行不同操作
            switch (parsed.name) {
                case 'DeadlineExpired':
                    // 可以自动重试
                    console.log('尝试延长截止时间重试...');
                    break;
                    
                case 'InsufficientOutputAmount':
                    // 提示用户调整滑点
                    console.log('建议增加滑点容忍度');
                    break;
                    
                case 'PriceImpactTooHigh':
                    // 建议减少交易金额
                    console.log('建议减少交易金额或分批交易');
                    break;
            }
            
            throw new Error(parsed.message);
        }
        
        // 其他错误
        console.error('未知错误:', error);
        throw error;
    }
}
```

---

## 8. 【面试必问】

### 问题1："Solidity的自定义Error和require有什么区别？什么时候用哪个？"

**普通回答（❌ 不出彩）：**

"自定义Error更省Gas，require用字符串。复杂场景用Error，简单场景用require。"

**出彩回答（✅ 推荐）：**

> **自定义Error和require有四个层面的区别：**
>
> **1. 语法层面**：
> ```solidity
> // require：条件 + 字符串
> require(balance >= amount, "Insufficient balance");
>
> // Error：定义 + revert
> error InsufficientBalance(uint256 available, uint256 required);
> if (balance < amount) revert InsufficientBalance(balance, amount);
> ```
>
> **2. Gas成本**：
> - **部署Gas**：Error更省（只存4字节选择器，不存字符串）
> - **执行Gas**：Error通常更省（ABI编码参数比字符串编码更高效）
> - 节省约30-50%的Gas（取决于字符串长度和参数数量）
>
> **3. 数据结构**：
> - require：非结构化字符串，难以程序化处理
> - Error：结构化类型，前端可以精确解析参数
>
> **4. 使用建议**：
> - **使用Error**：业务错误、需要参数、前端需解析、生产环境
> - **使用require**：简单输入验证、开发调试、极简场景
>
> **代码示例**：
> ```solidity
> // 推荐模式：混合使用
> function withdraw(uint256 amount) external {
>     require(amount > 0, "!amount"); // 简单验证
>     
>     if (balances[msg.sender] < amount) {
>         revert InsufficientBalance(balances[msg.sender], amount); // 业务错误
>     }
> }
> ```
>
> **在实际项目中**：OpenZeppelin 5.0+已经全面采用自定义Error。

**为什么这个回答出彩？**
1. ✅ 从语法、Gas、数据结构多角度对比
2. ✅ 给出了具体的使用建议
3. ✅ 提供了混合使用的代码示例
4. ✅ 提到了行业最佳实践（OpenZeppelin）

---

### 问题2："如何在前端捕获和处理Solidity自定义Error？"

**普通回答（❌ 不出彩）：**

"用try-catch捕获，然后解析错误数据。"

**出彩回答（✅ 推荐）：**

> **前端处理自定义Error需要三个步骤：**
>
> **1. 捕获错误数据**：
> ```javascript
> try {
>     await contract.withdraw(amount);
> } catch (error) {
>     // ethers.js v6中，错误数据在error.data
>     // 也可能在error.error.data（嵌套结构）
>     const errorData = error.data || error.error?.data;
> }
> ```
>
> **2. 解析错误**：
> ```javascript
> const iface = new ethers.Interface([
>     "error InsufficientBalance(uint256 available, uint256 required)"
> ]);
> const decoded = iface.parseError(errorData);
> // decoded.name = "InsufficientBalance"
> // decoded.args = [available, required]
> ```
>
> **3. 友好处理**：
> ```javascript
> const errorMessages = {
>     'InsufficientBalance': (args) => 
>         `余额不足：有 ${formatEther(args[0])}，需要 ${formatEther(args[1])}`
> };
> alert(errorMessages[decoded.name]?.(decoded.args) || '操作失败');
> ```
>
> **最佳实践**：
> - 在ABI中包含所有error定义
> - 建立错误消息映射表
> - 根据错误类型执行不同UI操作
> - 使用TypeScript获得类型安全
>
> **注意事项**：
> - 不同ethers版本的错误结构可能不同
> - 某些错误可能是节点层面的（如Gas不足），不是合约错误
> - 建议封装通用的错误处理函数

**为什么这个回答出彩？**
1. ✅ 给出了完整的三步流程
2. ✅ 包含具体代码示例
3. ✅ 提到了最佳实践
4. ✅ 考虑了边界情况和注意事项

---

## 9. 【化骨绵掌】

### 卡片1：直觉理解 - Error是什么？ 🎯

**一句话：** Error是给错误一个"名字"和"数据"，让错误信息更结构化、更省Gas。

**举例：**
```solidity
// 旧方式：一句话
require(false, "余额不足");

// 新方式：结构化
error InsufficientBalance(uint256 have, uint256 need);
revert InsufficientBalance(100, 500);
```

**应用：** 前端可以根据错误名称显示不同的UI，而不是解析字符串。

---

### 卡片2：形式化定义 - error语法 📐

**一句话：** 使用`error`关键字定义错误类型，`revert`关键字触发错误。

**语法：**
```solidity
// 定义
error ErrorName(type1 param1, type2 param2);

// 触发
if (condition) {
    revert ErrorName(value1, value2);
}
```

**应用：** Error可以全局定义或在合约内定义，参数类型任意。

---

### 卡片3：关键概念 - 错误选择器 🔢

**一句话：** Error有4字节选择器（类似函数选择器），用于唯一标识错误类型。

**举例：**
```solidity
error InsufficientBalance(uint256, uint256);
// 选择器 = keccak256("InsufficientBalance(uint256,uint256)")[0:4]
// → 0xcf479181
```

**应用：** 前端通过选择器识别错误类型，然后ABI解码参数。

---

### 卡片4：关键概念 - Gas优化 ⛽

**一句话：** Error比require字符串更省Gas，因为只存储4字节选择器而非完整字符串。

**对比：**
```
require("Insufficient balance")
→ 存储约20字节字符串

error InsufficientBalance()
→ 存储4字节选择器
```

**应用：** 大型合约中使用Error可显著降低部署成本。

---

### 卡片5：编程实现 - 常见模式 💻

**一句话：** `if + revert`是使用Error的标准模式。

**举例：**
```solidity
function withdraw(uint256 amount) external {
    if (balances[msg.sender] < amount) {
        revert InsufficientBalance(balances[msg.sender], amount);
    }
    // ...
}
```

**应用：** 在修饰符、内部函数中同样适用。

---

### 卡片6：对比区分 - Error vs require 🆚

**一句话：** Error是结构化类型，require是字符串；Error更省Gas，require更简洁。

**选择原则：**

| 场景 | 推荐 |
|-----|------|
| 简单输入验证 | require |
| 需要参数数据 | Error |
| 前端需解析 | Error |
| 开发调试 | require |

**应用：** 生产环境优先使用Error，开发阶段可混合使用。

---

### 卡片7：进阶理解 - Interface中的Error 📊

**一句话：** Error可以在Interface中定义，被实现合约使用。

**举例：**
```solidity
interface IVault {
    error InsufficientBalance(uint256 available, uint256 required);
    function withdraw(uint256 amount) external;
}

contract Vault is IVault {
    function withdraw(uint256 amount) external {
        if (...) revert InsufficientBalance(...);
    }
}
```

**应用：** 标准化接口的错误定义，便于前端统一处理。

---

### 卡片8：高级应用 - 前端解析 🖥️

**一句话：** 使用ethers.js的Interface.parseError解析Error数据。

**举例：**
```javascript
const iface = new ethers.Interface(["error InsufficientBalance(uint256,uint256)"]);
const decoded = iface.parseError(errorData);
console.log(decoded.name, decoded.args);
```

**应用：** 根据解析结果显示友好的错误消息。

---

### 卡片9：实战应用 - DeFi错误设计 🌐

**一句话：** DeFi项目应定义完整的错误类型集，便于用户理解失败原因。

**举例：**
```solidity
error SlippageExceeded(uint256 expected, uint256 actual);
error DeadlineExpired(uint256 deadline, uint256 current);
error InsufficientLiquidity(address token, uint256 required);
```

**应用：** Uniswap V3、Aave V3等都使用自定义Error。

---

### 卡片10：总结与延伸 🎓

**一句话：** Error是现代Solidity的错误处理标准，结合Gas优化和结构化数据的优势。

**核心要点：**
1. `error`定义，`revert`触发
2. 4字节选择器省Gas
3. 参数提供上下文
4. 前端可精确解析
5. 与require混合使用

**下一步学习：**
- receive/fallback
- try-catch错误处理
- 底层call的错误处理
- EIP-3156闪电贷错误标准

**记住：** Error = 结构化 + 省Gas + 易解析！

---

## 10. 【一句话总结】

**Error是Solidity 0.8.4+引入的自定义错误机制，通过`error`关键字定义带参数的错误类型，`revert`触发，使用4字节选择器替代字符串大幅节省Gas，同时提供结构化错误数据便于前端解析，是现代智能合约错误处理的最佳实践。**

---

## 📚 附录

### 学习检查清单

完成本知识点学习后，你应该能够：

- [ ] 使用`error`关键字定义自定义错误
- [ ] 使用`revert`正确触发错误
- [ ] 理解Error选择器的计算方式
- [ ] 在修饰符中使用自定义Error
- [ ] 理解Error与require的Gas差异
- [ ] 在Interface中定义Error
- [ ] 使用ethers.js解析前端错误
- [ ] 设计完整的错误处理架构
- [ ] 根据场景选择Error或require
- [ ] 遵循OpenZeppelin的错误命名规范

### 快速参考卡

**Error语法速查：**

```solidity
// 定义
error ErrorName();                           // 无参数
error ErrorName(type1 param1);              // 单参数
error ErrorName(type1 p1, type2 p2);        // 多参数

// 触发
revert ErrorName();
revert ErrorName(value1);
revert ErrorName(value1, value2);

// 在Interface中
interface IContract {
    error MyError(uint256 code);
}

// 全局定义
error GlobalError(string message);
```

**前端解析速查：**

```javascript
// ethers.js v6
const iface = new ethers.Interface(abi);
const decoded = iface.parseError(errorData);
console.log(decoded.name, decoded.args);
```

### 下一步学习

推荐按以下顺序继续学习：

1. **receive/fallback** - 接收ETH的特殊函数
2. **try-catch** - 外部调用的错误处理
3. **底层call** - 低级调用和错误处理
4. **代理合约** - 错误传播和处理

### 参考资源

**官方文档：**
- [Solidity Errors](https://docs.soliditylang.org/en/latest/contracts.html#errors-and-the-revert-statement)
- [Error Selector](https://docs.soliditylang.org/en/latest/abi-spec.html#errors)

**OpenZeppelin：**
- [Custom Errors](https://docs.openzeppelin.com/contracts/5.x/)
- [OpenZeppelin Contracts Source](https://github.com/OpenZeppelin/openzeppelin-contracts)

---

**版本：** v1.0
**创建日期：** 2025-12-07
**适用人群：** 前端工程师转Web3开发

---

**记住：** 自定义Error = 省Gas + 结构化 + 易解析，是现代Solidity的错误处理标准！🎯
