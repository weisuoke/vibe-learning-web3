# Hardhat 基础配置

## 1. 【30字核心】

**Hardhat 是以太坊开发框架，提供编译、测试、部署智能合约的完整工具链，通过 hardhat.config.js 配置项目环境。**

---

## 2. 【第一性原理】

### 什么是第一性原理？

**第一性原理**：回到事物最基本的真理，从源头思考问题

### Hardhat 的第一性原理 🎯

#### 1. 最基础的定义

**Hardhat = 智能合约开发环境 + 任务运行器 + 本地测试网络**

仅此而已！没有更基础的了。

#### 2. 为什么需要 Hardhat？

**核心问题：如何高效地开发、测试和部署智能合约？**

在没有开发框架之前：
- 手动编译 Solidity 代码（调用 solc 编译器）
- 手动管理 ABI 和字节码
- 没有本地测试环境，只能部署到测试网
- 调试困难，无法看到详细错误信息
- 部署脚本需要从零编写

#### 3. Hardhat 的三层价值

##### 价值1：开发效率（编译与自动化）

**问题**：每次修改代码都要手动编译、手动处理 ABI。

**解决方案**：`npx hardhat compile` 一键编译，自动生成 artifacts（ABI + 字节码）。

**示例**：
```bash
# 一行命令完成编译
npx hardhat compile

# 自动生成：
# artifacts/contracts/MyToken.sol/MyToken.json
# 包含 ABI 和 bytecode
```

##### 价值2：测试能力（本地网络）

**问题**：真实测试网速度慢、需要测试 ETH、每次测试都上链。

**解决方案**：Hardhat Network 提供内存级本地网络，瞬间确认交易，无限测试 ETH。

**示例**：
```javascript
// 本地测试：毫秒级响应
const [owner] = await ethers.getSigners();
// 自动拥有 10000 ETH 测试资金

// 测试网部署：需要等待 15-30 秒确认
```

##### 价值3：调试体验（错误追踪）

**问题**：智能合约报错只有 "revert"，不知道具体原因。

**解决方案**：Hardhat 提供 Solidity 堆栈跟踪和 console.log。

**示例**：
```solidity
import "hardhat/console.sol";

function transfer(address to, uint256 amount) public {
    console.log("Sender balance:", balanceOf(msg.sender));
    console.log("Transfer amount:", amount);
    // 现在可以看到具体执行过程！
}
```

#### 4. 从第一性原理推导 Hardhat 设计

**推理链：**

```
1. 前提：需要编写和部署智能合约到以太坊
   ↓
2. 推导：需要编译器将 Solidity 转换为字节码 → 集成 solc 编译器
   ↓
3. 推导：需要测试合约逻辑 → 提供本地测试网络（Hardhat Network）
   ↓
4. 推导：需要自动化测试 → 集成 Mocha/Chai 测试框架
   ↓
5. 推导：需要与合约交互 → 集成 ethers.js 库
   ↓
6. 推导：需要部署到不同网络 → 提供网络配置系统
   ↓
7. 推导：需要扩展功能 → 设计插件系统（plugins）
   ↓
8. 最终实现：hardhat.config.js 统一配置所有功能
```

#### 5. 一句话总结第一性原理

**Hardhat 是智能合约开发的"IDE"，将编译、测试、部署、调试整合为统一的开发体验，让开发者专注于业务逻辑而非工具链。**

---

## 3. 【3个核心概念】

### 核心概念1：hardhat.config.js 配置文件 ⚙️

**一句话定义：** hardhat.config.js 是 Hardhat 项目的核心配置文件，定义编译器版本、网络配置、插件等所有项目设置。

```javascript
// hardhat.config.js 完整示例
require("@nomicfoundation/hardhat-toolbox");
require("dotenv").config();

/** @type import('hardhat/config').HardhatUserConfig */
module.exports = {
  // ===== 1. Solidity 编译器配置 =====
  solidity: {
    version: "0.8.20",
    settings: {
      optimizer: {
        enabled: true,
        runs: 200  // 优化次数，影响 Gas 消耗
      }
    }
  },

  // ===== 2. 网络配置 =====
  networks: {
    // 本地开发网络（默认）
    hardhat: {
      chainId: 31337
    },
    // Sepolia 测试网
    sepolia: {
      url: process.env.SEPOLIA_RPC_URL,
      accounts: [process.env.PRIVATE_KEY],
      chainId: 11155111
    },
    // 以太坊主网
    mainnet: {
      url: process.env.MAINNET_RPC_URL,
      accounts: [process.env.PRIVATE_KEY],
      chainId: 1
    }
  },

  // ===== 3. Etherscan 验证配置 =====
  etherscan: {
    apiKey: process.env.ETHERSCAN_API_KEY
  },

  // ===== 4. 路径配置 =====
  paths: {
    sources: "./contracts",      // 合约源码目录
    tests: "./test",             // 测试文件目录
    cache: "./cache",            // 编译缓存
    artifacts: "./artifacts"     // 编译产物
  }
};
```

**详细解释：**

- **solidity**: 配置编译器版本和优化选项。`runs` 值越高，部署成本越高但调用成本越低。
- **networks**: 定义不同的部署目标网络，每个网络需要 RPC URL 和账户私钥。
- **etherscan**: 用于合约源码验证，需要在 Etherscan 申请 API Key。
- **paths**: 自定义项目目录结构。

**在智能合约开发中的应用：**

```bash
# 使用不同网络
npx hardhat compile                    # 编译合约
npx hardhat test                       # 本地测试
npx hardhat run scripts/deploy.js --network sepolia  # 部署到 Sepolia
```

---

### 核心概念2：Hardhat Network（本地测试网络）🔧

**一句话定义：** Hardhat Network 是内置的本地以太坊网络，提供即时交易确认、无限测试 ETH、丰富的调试功能。

```javascript
// hardhat.config.js 中的 Hardhat Network 配置
module.exports = {
  networks: {
    hardhat: {
      // 链 ID
      chainId: 31337,
      
      // 初始账户配置
      accounts: {
        mnemonic: "test test test test test test test test test test test junk",
        count: 20,                    // 生成 20 个测试账户
        accountsBalance: "10000000000000000000000"  // 每个账户 10000 ETH
      },
      
      // 挖矿配置
      mining: {
        auto: true,                   // 自动挖矿（每笔交易立即确认）
        interval: 0                   // 或设置固定出块时间（毫秒）
      },
      
      // Fork 主网配置
      forking: {
        url: "https://eth-mainnet.g.alchemy.com/v2/YOUR_KEY",
        blockNumber: 18000000         // 从特定区块 fork
      }
    }
  }
};
```

**详细解释：**

Hardhat Network 的核心优势：

1. **即时确认**：交易立即被"挖矿"，无需等待
2. **无限资金**：默认账户拥有 10000 ETH
3. **可 Fork 主网**：可以复制主网状态进行测试
4. **时间控制**：可以模拟时间流逝（evm_increaseTime）
5. **堆栈跟踪**：Solidity 错误显示详细调用栈

**在智能合约开发中的应用：**

```javascript
const { ethers } = require("hardhat");

async function main() {
  // 获取测试账户
  const [owner, user1, user2] = await ethers.getSigners();
  
  console.log("Owner:", owner.address);
  console.log("Balance:", ethers.formatEther(await owner.provider.getBalance(owner.address)));
  // Balance: 10000.0 ETH（无限测试资金）
  
  // 部署合约
  const Token = await ethers.getContractFactory("MyToken");
  const token = await Token.deploy();
  await token.waitForDeployment();
  
  console.log("Token deployed to:", await token.getAddress());
  // 部署瞬间完成（本地网络）
}
```

---

### 核心概念3：插件系统（Plugins）🔌

**一句话定义：** Hardhat 插件系统允许扩展功能，常用插件包括 hardhat-toolbox（集成包）、hardhat-etherscan（合约验证）等。

```javascript
// 安装常用插件
// npm install @nomicfoundation/hardhat-toolbox

// hardhat.config.js 中引入插件
require("@nomicfoundation/hardhat-toolbox");
// 这一行包含了以下所有插件：
// - @nomicfoundation/hardhat-ethers（ethers.js 集成）
// - @nomicfoundation/hardhat-chai-matchers（Chai 断言扩展）
// - @nomicfoundation/hardhat-network-helpers（网络操作助手）
// - hardhat-gas-reporter（Gas 报告）
// - solidity-coverage（测试覆盖率）
// - @typechain/hardhat（TypeScript 类型生成）
```

**常用插件列表：**

| 插件名 | 功能 | 使用场景 |
|-------|------|---------|
| hardhat-toolbox | 集成常用插件 | 新项目标配 |
| hardhat-etherscan | 合约源码验证 | 部署后验证 |
| hardhat-gas-reporter | Gas 使用报告 | 优化 Gas |
| solidity-coverage | 测试覆盖率 | 提高测试质量 |
| hardhat-deploy | 部署管理 | 复杂部署流程 |

**详细解释：**

插件通过 `require()` 引入后，会扩展 Hardhat 的功能：

```javascript
// 使用 hardhat-etherscan 验证合约
// hardhat.config.js
require("@nomicfoundation/hardhat-verify");

module.exports = {
  etherscan: {
    apiKey: "YOUR_ETHERSCAN_API_KEY"
  }
};

// 命令行验证
// npx hardhat verify --network sepolia DEPLOYED_ADDRESS "Constructor Arg1" "Arg2"
```

**在智能合约开发中的应用：**

```javascript
// 使用 hardhat-network-helpers 操作网络
const { time, mine, loadFixture } = require("@nomicfoundation/hardhat-network-helpers");

describe("TimeLock", function() {
  it("should unlock after time passes", async function() {
    // 模拟时间流逝
    await time.increase(3600); // 增加 1 小时
    
    // 手动挖矿
    await mine(10); // 挖 10 个区块
    
    // 获取当前时间
    const latestTime = await time.latest();
  });
});
```

---

## 4. 【最小可用】

掌握以下内容，就能用 Hardhat 开发智能合约：

### 4.1 项目初始化

```bash
# 创建新项目
mkdir my-project && cd my-project
npm init -y

# 安装 Hardhat
npm install --save-dev hardhat

# 初始化 Hardhat 项目
npx hardhat init
# 选择 "Create a JavaScript project"

# 项目结构
my-project/
├── contracts/          # Solidity 合约
│   └── Lock.sol
├── test/               # 测试文件
│   └── Lock.js
├── scripts/            # 部署脚本
│   └── deploy.js
├── hardhat.config.js   # 配置文件
└── package.json
```

### 4.2 基础配置

```javascript
// hardhat.config.js - 最小配置
require("@nomicfoundation/hardhat-toolbox");

module.exports = {
  solidity: "0.8.20"
};
```

### 4.3 常用命令

```bash
# 编译合约
npx hardhat compile

# 运行测试
npx hardhat test

# 运行部署脚本
npx hardhat run scripts/deploy.js

# 启动本地节点
npx hardhat node

# 在本地节点上部署
npx hardhat run scripts/deploy.js --network localhost

# 查看所有可用任务
npx hardhat help
```

### 4.4 环境变量配置

```bash
# 安装 dotenv
npm install dotenv

# 创建 .env 文件
SEPOLIA_RPC_URL=https://sepolia.infura.io/v3/YOUR_KEY
PRIVATE_KEY=your_private_key_here
ETHERSCAN_API_KEY=your_etherscan_key
```

```javascript
// hardhat.config.js
require("dotenv").config();

module.exports = {
  networks: {
    sepolia: {
      url: process.env.SEPOLIA_RPC_URL,
      accounts: [process.env.PRIVATE_KEY]
    }
  }
};
```

### 4.5 编写简单合约并测试

```solidity
// contracts/SimpleStorage.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract SimpleStorage {
    uint256 private value;

    event ValueChanged(uint256 newValue);

    function setValue(uint256 _value) public {
        value = _value;
        emit ValueChanged(_value);
    }

    function getValue() public view returns (uint256) {
        return value;
    }
}
```

```javascript
// test/SimpleStorage.js
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("SimpleStorage", function () {
  it("Should store and retrieve value", async function () {
    const SimpleStorage = await ethers.getContractFactory("SimpleStorage");
    const storage = await SimpleStorage.deploy();

    await storage.setValue(42);
    expect(await storage.getValue()).to.equal(42);
  });
});
```

---

**这些知识足以：**
- ✅ 初始化 Hardhat 项目
- ✅ 编译 Solidity 合约
- ✅ 编写和运行测试
- ✅ 部署到本地网络和测试网
- ✅ 为进阶功能（Fork、验证、Gas优化）打基础

---

## 5. 【1个类比】

### 类比1：Hardhat 项目结构 🏗️

#### 生活场景类比：Hardhat = 建筑工地的项目管理

想象你要建一栋房子：

**传统建筑（无项目管理）：**
- 自己找工人、买材料、画图纸
- 没有统一标准，每次都从零开始
- 出问题不知道找谁

**现代建筑公司（有项目管理）：**
- 项目经理统一协调一切
- 有标准化的工作流程
- 每个环节都有专人负责

**对应关系：**

| 建筑项目 | Hardhat 项目 | 说明 |
|---------|-------------|------|
| 项目计划书 | hardhat.config.js | 统一配置 |
| 设计图纸 | contracts/ | 合约源码 |
| 质量检测 | test/ | 测试文件 |
| 施工流程 | scripts/ | 部署脚本 |
| 建材仓库 | node_modules/ | 依赖包 |
| 成品验收 | artifacts/ | 编译产物 |

**举例：**

建房子流程：
```
1. 看设计图纸 → 编写 Solidity 合约
2. 质量检测 → 运行测试
3. 开始施工 → 部署合约
4. 验收交付 → 验证源码
```

---

#### 前端领域类比：Hardhat = Create React App (CRA)

如果你熟悉前端开发，Hardhat 就像智能合约领域的 **Create React App**：

```bash
# 前端项目初始化
npx create-react-app my-app

# 智能合约项目初始化
npx hardhat init
```

**对应关系：**

| 前端概念 (CRA/Vite) | Hardhat 概念 | 说明 |
|-------------------|-------------|------|
| package.json | hardhat.config.js | 项目配置 |
| src/ | contracts/ | 源代码目录 |
| \_\_tests\_\_/ | test/ | 测试目录 |
| public/ | scripts/ | 公共资源/脚本 |
| build/ | artifacts/ | 构建产物 |
| npm run build | npx hardhat compile | 编译命令 |
| npm run test | npx hardhat test | 测试命令 |
| npm run start | npx hardhat node | 启动开发服务 |
| .env | .env | 环境变量 |
| webpack plugins | hardhat plugins | 插件扩展 |

**代码对比：**

```javascript
// 前端：package.json 配置
{
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test"
  },
  "dependencies": {
    "react": "^18.0.0"
  }
}

// Hardhat：hardhat.config.js 配置
module.exports = {
  solidity: "0.8.20",
  networks: {
    sepolia: {
      url: process.env.SEPOLIA_RPC_URL,
      accounts: [process.env.PRIVATE_KEY]
    }
  }
};
```

**工作流对比：**

```bash
# 前端开发流程
npm run start   # 启动开发服务器
# 修改代码 → 热更新
npm run test    # 运行测试
npm run build   # 构建生产版本

# 智能合约开发流程
npx hardhat node        # 启动本地节点
# 修改合约 → 重新编译
npx hardhat test        # 运行测试
npx hardhat run scripts/deploy.js --network mainnet  # 部署到主网
```

---

### 类比2：hardhat.config.js 配置 📝

#### 生活场景类比：配置文件 = 餐厅菜单

想象你开一家餐厅：

**菜单（配置文件）定义了：**
- 有哪些菜品（合约）
- 用什么食材（编译器版本）
- 支持哪些支付方式（网络配置）
- 有哪些特色服务（插件）

```javascript
// 餐厅"配置文件"
const restaurantConfig = {
  // 菜品类型（编译器）
  cuisine: {
    type: "中餐",
    version: "川菜-2024版"
  },
  
  // 支付方式（网络）
  payment: {
    cash: { enabled: true },          // 本地测试
    alipay: { apiKey: "xxx" },        // 测试网
    wechat: { apiKey: "yyy" }         // 主网
  },
  
  // 特色服务（插件）
  services: ["外卖", "包间", "停车"]
};
```

---

#### 前端领域类比：hardhat.config.js = webpack.config.js

```javascript
// webpack.config.js（前端）
module.exports = {
  entry: './src/index.js',
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: 'bundle.js'
  },
  module: {
    rules: [
      { test: /\.js$/, use: 'babel-loader' }
    ]
  },
  plugins: [
    new HtmlWebpackPlugin()
  ]
};

// hardhat.config.js（智能合约）
module.exports = {
  solidity: "0.8.20",               // 类似 entry 和 loader
  paths: {
    sources: "./contracts",          // 类似 entry
    artifacts: "./artifacts"         // 类似 output
  },
  networks: {                        // 类似 devServer
    hardhat: { chainId: 31337 },
    sepolia: { url: "...", accounts: ["..."] }
  }
  // 插件通过 require() 引入，类似 plugins
};
```

---

### 类比总结表

| Hardhat 概念 | 生活场景类比 | 前端领域类比 | 核心相似性 |
|-------------|-------------|-------------|-----------|
| **Hardhat** | 建筑项目管理公司 | Create React App | 统一的开发框架 |
| **hardhat.config.js** | 项目规划书/餐厅菜单 | webpack.config.js | 中心化配置 |
| **contracts/** | 设计图纸 | src/ 源码目录 | 核心代码位置 |
| **test/** | 质量检测部门 | \_\_tests\_\_/ | 测试代码位置 |
| **artifacts/** | 建成的房子/成品 | build/dist/ | 构建产物 |
| **Hardhat Network** | 模拟沙盘 | 开发服务器 (localhost) | 本地测试环境 |
| **plugins** | 专业分包商 | webpack plugins | 功能扩展 |
| **npx hardhat compile** | 开工建设 | npm run build | 构建命令 |
| **npx hardhat test** | 验收检查 | npm run test | 测试命令 |
| **--network sepolia** | 在测试区建样板房 | staging 环境 | 预发布环境 |

---

## 6. 【反直觉点】

### 误区1：Hardhat 只是编译器，和 Remix 差不多 ❌

**为什么错？**

很多初学者认为 Hardhat 和 Remix IDE 功能相同，都是"编译 Solidity 的工具"：

- **Remix**：浏览器 IDE，适合快速原型和学习
- **Hardhat**：完整的开发框架，适合生产级项目

Hardhat 提供的远不止编译：

```javascript
// Hardhat 独有功能

// 1. 本地测试网络（可配置、可 Fork）
npx hardhat node --fork https://eth-mainnet.alchemyapi.io/v2/KEY

// 2. 自动化测试框架
describe("MyContract", () => {
  it("should work", async () => {
    // 完整的测试生命周期
  });
});

// 3. 部署脚本管理
// scripts/deploy.js - 版本控制、可重复执行

// 4. 插件生态
require("hardhat-gas-reporter");
require("solidity-coverage");

// 5. TypeScript 支持
// 自动生成类型定义
```

**为什么人们容易这样错？**

因为刚接触智能合约时通常先用 Remix（在线、零配置），误以为所有开发工具都是"编译器"。

**正确理解：**

| 功能 | Remix | Hardhat |
|------|-------|---------|
| 编译 | ✅ | ✅ |
| 部署 | ✅（手动）| ✅（脚本化）|
| 测试 | 基础 | 完整（Mocha/Chai）|
| 本地网络 | ❌ | ✅（可配置）|
| 主网 Fork | ❌ | ✅ |
| 插件生态 | ❌ | ✅（丰富）|
| CI/CD 集成 | 困难 | 容易 |
| 版本控制 | 困难 | Git 友好 |

---

### 误区2：hardhat.config.js 中的私钥可以直接写 ❌

**为什么错？**

```javascript
// ❌ 错误做法：私钥硬编码
module.exports = {
  networks: {
    sepolia: {
      url: "https://sepolia.infura.io/v3/...",
      accounts: ["0x1234567890abcdef..."] // 私钥直接写在代码里！
    }
  }
};
```

这会导致：
1. **私钥泄露**：代码推送到 GitHub 后，私钥暴露
2. **资金被盗**：机器人扫描 GitHub 寻找私钥，秒转走所有资产
3. **无法撤销**：区块链交易不可逆，资金无法找回

**为什么人们容易这样错？**

因为教程为了简化经常展示硬编码私钥，初学者不了解 `.env` 和 `.gitignore` 的重要性。

**正确理解：**

```bash
# 1. 安装 dotenv
npm install dotenv

# 2. 创建 .env 文件（不要提交到 Git）
# .env
PRIVATE_KEY=0x1234567890abcdef...
SEPOLIA_RPC_URL=https://sepolia.infura.io/v3/...

# 3. 添加到 .gitignore
echo ".env" >> .gitignore
```

```javascript
// hardhat.config.js
require("dotenv").config();

module.exports = {
  networks: {
    sepolia: {
      url: process.env.SEPOLIA_RPC_URL,
      accounts: [process.env.PRIVATE_KEY]  // ✅ 从环境变量读取
    }
  }
};
```

---

### 误区3：npx hardhat test 和普通 JavaScript 测试一样 ❌

**为什么错？**

虽然 Hardhat 使用 Mocha/Chai，但智能合约测试有其独特性：

```javascript
// ❌ 错误理解：当作普通 JS 测试
describe("Token", () => {
  it("should transfer", () => {
    const result = transfer(100);  // 同步调用？
    expect(result).to.equal(true);
  });
});

// ✅ 正确理解：异步 + 区块链特性
describe("Token", () => {
  it("should transfer", async () => {
    const [owner, recipient] = await ethers.getSigners();
    const Token = await ethers.getContractFactory("Token");
    const token = await Token.deploy();
    
    // 所有合约操作都是异步的！
    await token.transfer(recipient.address, 100);
    
    // 需要查询链上状态
    expect(await token.balanceOf(recipient.address)).to.equal(100);
  });
});
```

**区别：**

| 普通 JS 测试 | 智能合约测试 |
|-------------|-------------|
| 同步操作为主 | 几乎全是异步 |
| 内存中执行 | 区块链上执行 |
| 直接断言返回值 | 需查询链上状态 |
| 无 Gas 概念 | 需考虑 Gas 消耗 |
| 无账户概念 | 需要 signers |

**为什么人们容易这样错？**

前端工程师习惯了同步的单元测试，不熟悉区块链的异步特性和状态模型。

**正确理解：**

```javascript
const { expect } = require("chai");
const { ethers } = require("hardhat");
const { loadFixture } = require("@nomicfoundation/hardhat-network-helpers");

describe("Token", function () {
  // 使用 fixture 优化测试性能
  async function deployTokenFixture() {
    const [owner, user1, user2] = await ethers.getSigners();
    const Token = await ethers.getContractFactory("Token");
    const token = await Token.deploy(1000000);
    return { token, owner, user1, user2 };
  }

  it("Should transfer tokens", async function () {
    const { token, owner, user1 } = await loadFixture(deployTokenFixture);
    
    // 1. 执行转账（异步）
    await token.transfer(user1.address, 100);
    
    // 2. 验证链上状态（异步）
    expect(await token.balanceOf(user1.address)).to.equal(100);
  });

  it("Should emit Transfer event", async function () {
    const { token, owner, user1 } = await loadFixture(deployTokenFixture);
    
    // 3. 验证事件（Hardhat 特有的 Chai 扩展）
    await expect(token.transfer(user1.address, 100))
      .to.emit(token, "Transfer")
      .withArgs(owner.address, user1.address, 100);
  });

  it("Should revert on insufficient balance", async function () {
    const { token, user1, user2 } = await loadFixture(deployTokenFixture);
    
    // 4. 验证 revert（Hardhat 特有）
    await expect(token.connect(user1).transfer(user2.address, 100))
      .to.be.revertedWith("Insufficient balance");
  });
});
```

---

## 7. 【实战代码】

### 基础实现：从零搭建 Hardhat 项目

```bash
# ===== 1. 创建项目目录 =====
mkdir my-token-project
cd my-token-project

# ===== 2. 初始化 npm =====
npm init -y

# ===== 3. 安装 Hardhat =====
npm install --save-dev hardhat

# ===== 4. 初始化 Hardhat 项目 =====
npx hardhat init
# 选择 "Create a JavaScript project"
# 按 Enter 接受所有默认选项

# ===== 5. 安装额外依赖 =====
npm install --save-dev dotenv
```

```javascript
// ===== hardhat.config.js =====
require("@nomicfoundation/hardhat-toolbox");
require("dotenv").config();

module.exports = {
  solidity: {
    version: "0.8.20",
    settings: {
      optimizer: {
        enabled: true,
        runs: 200
      }
    }
  },
  networks: {
    hardhat: {
      chainId: 31337
    },
    sepolia: {
      url: process.env.SEPOLIA_RPC_URL || "",
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : []
    }
  },
  etherscan: {
    apiKey: process.env.ETHERSCAN_API_KEY
  }
};
```

```solidity
// ===== contracts/MyToken.sol =====
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "hardhat/console.sol";  // Hardhat 调试工具

contract MyToken {
    string public name = "MyToken";
    string public symbol = "MTK";
    uint8 public decimals = 18;
    uint256 public totalSupply;
    
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    
    constructor(uint256 _initialSupply) {
        totalSupply = _initialSupply * 10 ** decimals;
        balanceOf[msg.sender] = totalSupply;
        console.log("Deploying MyToken with supply:", totalSupply);
    }
    
    function transfer(address to, uint256 amount) public returns (bool) {
        console.log("Transfer from %s to %s: %s", msg.sender, to, amount);
        require(balanceOf[msg.sender] >= amount, "Insufficient balance");
        
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        
        emit Transfer(msg.sender, to, amount);
        return true;
    }
    
    function approve(address spender, uint256 amount) public returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }
    
    function transferFrom(address from, address to, uint256 amount) public returns (bool) {
        require(balanceOf[from] >= amount, "Insufficient balance");
        require(allowance[from][msg.sender] >= amount, "Allowance exceeded");
        
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        allowance[from][msg.sender] -= amount;
        
        emit Transfer(from, to, amount);
        return true;
    }
}
```

```javascript
// ===== test/MyToken.test.js =====
const { expect } = require("chai");
const { ethers } = require("hardhat");
const { loadFixture } = require("@nomicfoundation/hardhat-network-helpers");

describe("MyToken", function () {
  const INITIAL_SUPPLY = 1000000;  // 100万代币

  async function deployTokenFixture() {
    const [owner, user1, user2] = await ethers.getSigners();
    const MyToken = await ethers.getContractFactory("MyToken");
    const token = await MyToken.deploy(INITIAL_SUPPLY);
    return { token, owner, user1, user2 };
  }

  describe("Deployment", function () {
    it("Should set the correct name and symbol", async function () {
      const { token } = await loadFixture(deployTokenFixture);
      expect(await token.name()).to.equal("MyToken");
      expect(await token.symbol()).to.equal("MTK");
    });

    it("Should assign total supply to owner", async function () {
      const { token, owner } = await loadFixture(deployTokenFixture);
      const expectedSupply = ethers.parseUnits(INITIAL_SUPPLY.toString(), 18);
      expect(await token.balanceOf(owner.address)).to.equal(expectedSupply);
    });
  });

  describe("Transfers", function () {
    it("Should transfer tokens between accounts", async function () {
      const { token, owner, user1 } = await loadFixture(deployTokenFixture);
      const amount = ethers.parseUnits("100", 18);

      await token.transfer(user1.address, amount);
      expect(await token.balanceOf(user1.address)).to.equal(amount);
    });

    it("Should emit Transfer event", async function () {
      const { token, owner, user1 } = await loadFixture(deployTokenFixture);
      const amount = ethers.parseUnits("100", 18);

      await expect(token.transfer(user1.address, amount))
        .to.emit(token, "Transfer")
        .withArgs(owner.address, user1.address, amount);
    });

    it("Should fail if sender has insufficient balance", async function () {
      const { token, user1, user2 } = await loadFixture(deployTokenFixture);
      const amount = ethers.parseUnits("100", 18);

      await expect(
        token.connect(user1).transfer(user2.address, amount)
      ).to.be.revertedWith("Insufficient balance");
    });
  });
});
```

```javascript
// ===== scripts/deploy.js =====
const { ethers } = require("hardhat");

async function main() {
  console.log("Deploying MyToken...");

  const [deployer] = await ethers.getSigners();
  console.log("Deploying with account:", deployer.address);

  const balance = await deployer.provider.getBalance(deployer.address);
  console.log("Account balance:", ethers.formatEther(balance), "ETH");

  const MyToken = await ethers.getContractFactory("MyToken");
  const token = await MyToken.deploy(1000000);  // 100万初始供应

  await token.waitForDeployment();
  const address = await token.getAddress();

  console.log("MyToken deployed to:", address);
  console.log("Total supply:", ethers.formatUnits(await token.totalSupply(), 18), "MTK");

  return address;
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

**运行命令：**

```bash
# 编译合约
npx hardhat compile

# 运行测试
npx hardhat test

# 启动本地节点
npx hardhat node

# 在另一个终端部署到本地节点
npx hardhat run scripts/deploy.js --network localhost
```

**运行输出示例：**

```
$ npx hardhat test

  MyToken
    Deployment
Deploying MyToken with supply: 1000000000000000000000000
      ✔ Should set the correct name and symbol (1234ms)
      ✔ Should assign total supply to owner
    Transfers
Transfer from 0xf39Fd6e51... to 0x70997970...: 100000000000000000000
      ✔ Should transfer tokens between accounts
      ✔ Should emit Transfer event
      ✔ Should fail if sender has insufficient balance

  5 passing (2s)
```

---

### 进阶：Fork 主网测试

```javascript
// hardhat.config.js - 添加 Fork 配置
module.exports = {
  networks: {
    hardhat: {
      forking: {
        url: process.env.MAINNET_RPC_URL,
        blockNumber: 18500000  // 固定区块号保证测试可复现
      }
    }
  }
};
```

```javascript
// test/fork.test.js - 测试真实 Uniswap 合约
const { expect } = require("chai");
const { ethers } = require("hardhat");

const UNISWAP_ROUTER = "0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D";
const WETH = "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2";
const USDC = "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48";

describe("Uniswap Fork Test", function () {
  it("Should get WETH/USDC price from real Uniswap", async function () {
    const router = await ethers.getContractAt(
      ["function getAmountsOut(uint amountIn, address[] memory path) view returns (uint[] memory amounts)"],
      UNISWAP_ROUTER
    );

    const amountIn = ethers.parseEther("1");  // 1 WETH
    const amounts = await router.getAmountsOut(amountIn, [WETH, USDC]);
    
    console.log("1 WETH =", ethers.formatUnits(amounts[1], 6), "USDC");
    expect(amounts[1]).to.be.gt(0);
  });
});
```

---

## 8. 【面试必问】

### 问题1："Hardhat 和 Foundry 有什么区别？你更推荐哪个？"

**普通回答（❌ 不出彩）：**

"Hardhat 用 JavaScript，Foundry 用 Solidity 写测试。Hardhat 生态更成熟，Foundry 更快。"

**出彩回答（✅ 推荐）：**

> **Hardhat 和 Foundry 是两个主流的智能合约开发框架，各有优势：**
>
> **1. 语言和测试方式**：
> - Hardhat 使用 JavaScript/TypeScript 编写测试，前端工程师上手快
> - Foundry 使用 Solidity 编写测试（forge test），更贴近合约逻辑
>
> **2. 性能差异**：
> - Foundry 编译和测试速度极快（用 Rust 编写，比 Hardhat 快 10-100 倍）
> - Hardhat 在大型项目中可能较慢，但有缓存机制优化
>
> **3. 生态和工具**：
> - Hardhat 插件生态丰富（etherscan 验证、gas reporter、coverage）
> - Foundry 工具更专注（forge、cast、anvil 各司其职）
>
> **4. 调试体验**：
> - Hardhat 有 console.log 和详细堆栈跟踪
> - Foundry 有 `forge test -vvvv` 查看调用栈，支持断点调试
>
> **5. 我的选择**：
> - 新项目或前端团队：推荐 Hardhat（熟悉 JavaScript 生态）
> - 追求测试速度和 Solidity 纯粹性：推荐 Foundry
> - 实际工作中：两者可以结合使用（用 Hardhat 部署，用 Foundry 测试）
>
> **实际案例**：
> - Uniswap V4 同时使用 Hardhat 和 Foundry
> - OpenZeppelin 合约库主要用 Hardhat
> - Paradigm 系项目偏好 Foundry

**为什么这个回答出彩？**
1. ✅ 多维度对比（语言、性能、生态、调试）
2. ✅ 给出了具体选择建议（不同场景）
3. ✅ 提到了可以结合使用
4. ✅ 引用真实项目案例

---

### 问题2："如何配置 Hardhat 项目以支持多链部署？"

**普通回答（❌ 不出彩）：**

"在 networks 里配置不同的 RPC URL 和私钥就行了。"

**出彩回答（✅ 推荐）：**

> **多链部署需要从三个层面配置：**
>
> **1. 网络配置层**：
> ```javascript
> // hardhat.config.js
> networks: {
>   ethereum: { url: process.env.ETH_RPC, chainId: 1 },
>   polygon: { url: process.env.POLYGON_RPC, chainId: 137 },
>   arbitrum: { url: process.env.ARB_RPC, chainId: 42161 },
>   optimism: { url: process.env.OP_RPC, chainId: 10 }
> }
> ```
>
> **2. 验证配置层**：
> ```javascript
> etherscan: {
>   apiKey: {
>     mainnet: process.env.ETHERSCAN_KEY,
>     polygon: process.env.POLYGONSCAN_KEY,
>     arbitrumOne: process.env.ARBISCAN_KEY
>   }
> }
> ```
>
> **3. 部署脚本层**：
> ```javascript
> // 根据网络动态调整配置
> const chainId = network.config.chainId;
> const config = {
>   1: { weth: "0xC02a...", router: "0x7a25..." },
>   137: { weth: "0x0d50...", router: "0xa5E0..." }
> };
> ```
>
> **最佳实践**：
> - 使用 hardhat-deploy 插件管理多链部署
> - 每个链的地址存储在 deployments/[network]/ 目录
> - 使用 GitHub Actions 自动化多链部署
> - 部署后自动验证源码

**为什么这个回答出彩？**
1. ✅ 分层次回答（网络、验证、脚本）
2. ✅ 给出了具体代码示例
3. ✅ 提到了最佳实践和自动化
4. ✅ 展示了实际项目经验

---

## 9. 【化骨绵掌】

### 卡片1：Hardhat 是什么？ 🎯

**一句话：** Hardhat 是以太坊智能合约开发框架，集成编译、测试、部署、调试功能。

**举例：**
```bash
npx hardhat init  # 一行命令创建项目
npx hardhat test  # 运行所有测试
npx hardhat run scripts/deploy.js --network sepolia  # 部署
```

**应用：** 所有专业 DApp 项目都使用 Hardhat 或 Foundry 开发，替代 Remix 在线 IDE。

---

### 卡片2：hardhat.config.js 核心配置 📐

**一句话：** hardhat.config.js 是项目入口，定义编译器、网络、插件等所有配置。

**举例：**
```javascript
module.exports = {
  solidity: "0.8.20",           // 编译器版本
  networks: {                    // 网络配置
    sepolia: { url: "...", accounts: ["..."] }
  },
  etherscan: { apiKey: "..." }  // 验证配置
};
```

**应用：** 修改此文件即可切换编译器版本、添加新网络、启用新插件。

---

### 卡片3：Hardhat Network 本地测试网 🔧

**一句话：** Hardhat Network 是内置的以太坊本地网络，即时确认、无限 ETH、可 Fork 主网。

**举例：**
```bash
npx hardhat node    # 启动本地节点（端口 8545）
# 默认 20 个账户，每个 10000 ETH
```

**应用：** 开发时在本地测试，无需等待真实网络确认，调试速度提升 100 倍。

---

### 卡片4：插件系统 🔌

**一句话：** Hardhat 插件扩展功能，hardhat-toolbox 是最常用的集成包。

**举例：**
```javascript
require("@nomicfoundation/hardhat-toolbox");
// 包含：ethers 集成、Chai 扩展、Gas 报告、覆盖率
```

**应用：** 安装插件后可以使用新命令，如 `npx hardhat verify` 验证合约源码。

---

### 卡片5：编译合约 ⚙️

**一句话：** `npx hardhat compile` 将 Solidity 编译为字节码和 ABI，存入 artifacts/。

**举例：**
```bash
npx hardhat compile
# 生成 artifacts/contracts/MyToken.sol/MyToken.json
# 包含 abi 和 bytecode
```

**应用：** 编译后可以在测试和部署脚本中使用 `ethers.getContractFactory("MyToken")`。

---

### 卡片6：console.log 调试 🐛

**一句话：** Hardhat 提供 Solidity 版 console.log，在本地测试时输出调试信息。

**举例：**
```solidity
import "hardhat/console.sol";

function transfer(address to, uint256 amount) public {
    console.log("From:", msg.sender);
    console.log("Amount:", amount);
}
```

**应用：** 调试复杂合约逻辑时，可以打印变量值，找出 bug 位置。

---

### 卡片7：环境变量安全 🔐

**一句话：** 永远不要在代码中硬编码私钥，使用 .env 文件 + dotenv 库。

**举例：**
```bash
# .env（添加到 .gitignore）
PRIVATE_KEY=0x1234...
SEPOLIA_RPC_URL=https://...
```
```javascript
require("dotenv").config();
accounts: [process.env.PRIVATE_KEY]
```

**应用：** 防止私钥泄露到 GitHub，保护链上资产安全。

---

### 卡片8：Fork 主网测试 🍴

**一句话：** Hardhat 可以 Fork 主网状态，在本地测试真实 DeFi 协议。

**举例：**
```javascript
networks: {
  hardhat: {
    forking: {
      url: "https://eth-mainnet.g.alchemy.com/v2/KEY",
      blockNumber: 18000000
    }
  }
}
```

**应用：** 测试与 Uniswap、Aave 等协议的集成，无需真实交易。

---

### 卡片9：部署脚本最佳实践 📜

**一句话：** 部署脚本应该可重复执行、支持多网络、输出合约地址。

**举例：**
```javascript
async function main() {
  const Token = await ethers.getContractFactory("Token");
  const token = await Token.deploy(1000000);
  await token.waitForDeployment();
  console.log("Deployed to:", await token.getAddress());
}
```

**应用：** 使用 `--network` 参数切换部署目标：localhost、sepolia、mainnet。

---

### 卡片10：Hardhat vs Foundry 选择 🆚

**一句话：** Hardhat 适合 JavaScript 团队和初学者，Foundry 适合追求性能和 Solidity 纯粹性。

**对比：**

| 特性 | Hardhat | Foundry |
|------|---------|---------|
| 测试语言 | JavaScript | Solidity |
| 编译速度 | 较慢 | 极快 |
| 插件生态 | 丰富 | 精简 |
| 学习曲线 | 平缓 | 较陡 |

**应用：** 大型项目可同时使用两者，取长补短。

---

## 10. 【一句话总结】

**Hardhat 是以太坊智能合约开发的核心框架，通过 hardhat.config.js 配置编译器、网络和插件，提供本地测试网络、console.log 调试、Fork 主网等功能，是从 Remix 迁移到专业开发的必经之路。**

---

## 📚 附录

### 学习检查清单

完成本知识点学习后，你应该能够：

- [ ] 使用 `npx hardhat init` 创建新项目
- [ ] 配置 hardhat.config.js 的编译器和网络
- [ ] 使用 .env 管理私钥等敏感信息
- [ ] 编写并运行智能合约测试
- [ ] 使用 console.log 调试合约
- [ ] 部署合约到本地和测试网
- [ ] 配置 Fork 主网进行集成测试
- [ ] 使用 hardhat-etherscan 验证合约

### 快速参考卡

**常用命令：**

```bash
npx hardhat init              # 初始化项目
npx hardhat compile           # 编译合约
npx hardhat test              # 运行测试
npx hardhat test --grep "transfer"  # 运行匹配的测试
npx hardhat node              # 启动本地节点
npx hardhat run scripts/deploy.js   # 运行脚本
npx hardhat run scripts/deploy.js --network sepolia  # 指定网络
npx hardhat verify --network sepolia ADDRESS "arg1"  # 验证合约
npx hardhat clean             # 清除缓存和编译产物
npx hardhat help              # 查看帮助
```

**项目结构：**

```
my-project/
├── contracts/          # Solidity 合约
├── test/               # 测试文件
├── scripts/            # 部署脚本
├── artifacts/          # 编译产物（自动生成）
├── cache/              # 编译缓存（自动生成）
├── hardhat.config.js   # 配置文件
├── package.json        # npm 配置
└── .env                # 环境变量（不要提交到 Git）
```

### 下一步学习

推荐按以下顺序继续学习：

1. **编写测试** - 学习 Mocha/Chai 测试智能合约
2. **获取不同签名者** - 测试多账户交互场景
3. **模拟时间流逝** - 测试时间锁和锁仓逻辑
4. **部署脚本进阶** - 多网络部署和合约升级
5. **Gas 优化** - 使用 hardhat-gas-reporter

### 参考资源

**官方文档：**
- [Hardhat 官方文档](https://hardhat.org/docs)
- [Hardhat 插件列表](https://hardhat.org/hardhat-runner/plugins)

**教程：**
- [Hardhat Tutorial](https://hardhat.org/tutorial)
- [Alchemy Hardhat Guide](https://docs.alchemy.com/docs/how-to-develop-an-nft-smart-contract-erc721-with-alchemy)

**开发工具：**
- [Etherscan](https://etherscan.io/) - 合约验证和浏览
- [Alchemy](https://www.alchemy.com/) - RPC 提供商
- [Infura](https://infura.io/) - RPC 提供商

---

**版本：** v1.0
**创建日期：** 2025-12-08
**作者：** Droid
**适用人群：** 前端工程师转 Web3 开发
