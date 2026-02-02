# wagmi + viem

## 1. 【30字核心】

**wagmi是React Hooks库，viem是轻量级TypeScript以太坊客户端，二者组合是现代DApp前端开发的首选方案，提供类型安全、高性能的区块链交互能力。**

---

## 2. 【第一性原理】

### 什么是第一性原理？

**第一性原理**：回到事物最基本的真理，从源头思考问题

### wagmi + viem 的第一性原理 🎯

#### 1. 最基础的定义

**wagmi = React Hooks + 以太坊交互**
**viem = TypeScript + JSON-RPC 客户端**

核心本质：
- **wagmi**：将区块链操作封装成 React Hooks，让前端开发者用熟悉的方式与区块链交互
- **viem**：提供底层的以太坊通信能力，处理 JSON-RPC 请求、ABI 编解码、交易签名等

仅此而已！没有更基础的了。

#### 2. 为什么需要 wagmi + viem？

**核心问题：前端如何优雅地与区块链交互？**

传统方案的问题：
- **web3.js**：包体积大（>1MB）、类型支持差、API 设计老旧
- **ethers.js**：虽然更好，但不是为 React 设计，状态管理需要自己处理
- **直接用 JSON-RPC**：太底层，需要处理大量细节

**解决方案**：
- viem 替代 ethers.js 作为底层库（更轻、更快、类型更好）
- wagmi 提供 React 层封装（状态管理、缓存、重试机制内置）

#### 3. wagmi + viem 的三层价值

##### 价值1：开发体验 - React 原生方式

```typescript
// 传统方式：手动管理状态
const [account, setAccount] = useState('');
const [loading, setLoading] = useState(false);
const [error, setError] = useState(null);

useEffect(() => {
  const fetchAccount = async () => {
    setLoading(true);
    try {
      const accounts = await ethereum.request({ method: 'eth_accounts' });
      setAccount(accounts[0]);
    } catch (e) {
      setError(e);
    } finally {
      setLoading(false);
    }
  };
  fetchAccount();
}, []);

// wagmi 方式：一行搞定
const { address, isConnecting, isConnected } = useAccount();
```

##### 价值2：类型安全 - 编译时发现错误

```typescript
// viem 的类型推断
const balance = await publicClient.readContract({
  address: '0x...',
  abi: erc20Abi,
  functionName: 'balanceOf',  // ✅ 自动补全
  args: ['0x...']              // ✅ 参数类型检查
});
// balance 的类型自动推断为 bigint
```

##### 价值3：性能优化 - 内置最佳实践

```typescript
// wagmi 自动处理：
// - 请求去重（同时多个组件请求同一数据，只发一次请求）
// - 智能缓存（相同请求复用缓存）
// - 自动重试（网络错误时指数退避重试）
// - 后台刷新（数据过期时后台更新）
const { data } = useBalance({ address });
```

#### 4. 从第一性原理推导 wagmi + viem 设计

**推理链：**

```
1. 前提：DApp 需要与区块链交互
   ↓
2. 推导：需要处理 JSON-RPC 通信
   ↓
3. 推导：需要 ABI 编解码、地址处理、签名等
   ↓
4. 推导：这些是底层能力 → viem 提供
   ↓
5. 推导：前端需要状态管理（loading/error/data）
   ↓
6. 推导：React 使用 Hooks 管理状态
   ↓
7. 推导：需要缓存、去重、重试机制
   ↓
8. 推导：这些是 React 层能力 → wagmi 提供
   ↓
9. 最终：viem（底层）+ wagmi（React层）= 完整 DApp 前端方案
```

#### 5. 一句话总结第一性原理

**viem 是底层的以太坊通信引擎，wagmi 是 React 层的状态管理框架，二者分工明确、配合使用，提供了类型安全、高性能、开发体验优秀的 DApp 前端解决方案。**

---

## 3. 【3个核心概念】

### 核心概念1：Config 配置 ⚙️

**一句话定义：** Config 是 wagmi 的核心配置对象，定义了链、连接器、传输层等所有基础设置。

```typescript
import { createConfig, http } from 'wagmi';
import { mainnet, sepolia } from 'wagmi/chains';
import { injected, walletConnect } from 'wagmi/connectors';

// 创建 wagmi 配置
const config = createConfig({
  // 支持的链
  chains: [mainnet, sepolia],

  // 连接器（钱包）
  connectors: [
    injected(),  // MetaMask 等浏览器钱包
    walletConnect({ projectId: 'YOUR_PROJECT_ID' }),
  ],

  // 传输层（RPC 配置）
  transports: {
    [mainnet.id]: http('https://eth-mainnet.g.alchemy.com/v2/YOUR_KEY'),
    [sepolia.id]: http('https://eth-sepolia.g.alchemy.com/v2/YOUR_KEY'),
  },
});

export default config;
```

**配置结构可视化：**

```
wagmi Config
├── chains: Chain[]          // 支持哪些链
│   ├── mainnet (id: 1)
│   ├── sepolia (id: 11155111)
│   └── ...
├── connectors: Connector[]  // 支持哪些钱包
│   ├── injected (MetaMask)
│   ├── walletConnect
│   └── ...
└── transports: Transport    // 每条链用什么 RPC
    ├── 1 → http(mainnet-rpc)
    └── 11155111 → http(sepolia-rpc)
```

**在 DApp 中的应用：**
- Config 是整个应用的基础，决定了支持哪些链和钱包
- 通过 WagmiProvider 注入到 React 组件树

---

### 核心概念2：Client 客户端 🔌

**一句话定义：** Client 是与区块链通信的实际执行者，分为 publicClient（读取）和 walletClient（写入）。

```typescript
import { createPublicClient, createWalletClient, http } from 'viem';
import { mainnet } from 'viem/chains';

// ===== Public Client：只读操作 =====
// 不需要钱包，任何人都可以读取区块链数据
const publicClient = createPublicClient({
  chain: mainnet,
  transport: http('https://eth-mainnet.g.alchemy.com/v2/YOUR_KEY'),
});

// 读取区块号
const blockNumber = await publicClient.getBlockNumber();

// 读取合约数据
const balance = await publicClient.readContract({
  address: '0x...',
  abi: erc20Abi,
  functionName: 'balanceOf',
  args: ['0x...'],
});

// ===== Wallet Client：写入操作 =====
// 需要钱包签名，用于发送交易
const walletClient = createWalletClient({
  chain: mainnet,
  transport: custom(window.ethereum),  // 使用 MetaMask
});

// 发送交易
const hash = await walletClient.writeContract({
  address: '0x...',
  abi: erc20Abi,
  functionName: 'transfer',
  args: ['0x...', 100n],
});
```

**两种 Client 对比：**

| 特性 | Public Client | Wallet Client |
|-----|--------------|---------------|
| 用途 | 读取数据 | 发送交易 |
| 需要钱包 | ❌ | ✅ |
| Gas 费用 | 免费 | 需要付费 |
| 典型操作 | getBalance, readContract | sendTransaction, writeContract |

**在 wagmi 中的使用：**

```typescript
import { usePublicClient, useWalletClient } from 'wagmi';

function MyComponent() {
  const publicClient = usePublicClient();
  const { data: walletClient } = useWalletClient();

  // 使用 publicClient 读取数据
  // 使用 walletClient 发送交易
}
```

---

### 核心概念3：Hooks 钩子 🪝

**一句话定义：** Hooks 是 wagmi 的核心 API，将区块链操作封装成 React Hooks，自动处理状态管理和缓存。

```typescript
import {
  useAccount,       // 账户信息
  useConnect,       // 连接钱包
  useDisconnect,    // 断开钱包
  useBalance,       // 查询余额
  useReadContract,  // 读取合约
  useWriteContract, // 写入合约
  useWaitForTransactionReceipt,  // 等待交易确认
} from 'wagmi';

// ===== 账户相关 Hooks =====
function AccountInfo() {
  const { address, isConnected, chain } = useAccount();
  const { connect, connectors } = useConnect();
  const { disconnect } = useDisconnect();

  if (!isConnected) {
    return (
      <button onClick={() => connect({ connector: connectors[0] })}>
        连接钱包
      </button>
    );
  }

  return (
    <div>
      <p>地址: {address}</p>
      <p>链: {chain?.name}</p>
      <button onClick={() => disconnect()}>断开</button>
    </div>
  );
}

// ===== 数据查询 Hooks =====
function BalanceDisplay({ address }) {
  const { data, isLoading, error } = useBalance({ address });

  if (isLoading) return <div>加载中...</div>;
  if (error) return <div>错误: {error.message}</div>;

  return <div>余额: {data?.formatted} {data?.symbol}</div>;
}

// ===== 合约交互 Hooks =====
function TokenTransfer() {
  const { writeContract, data: hash, isPending } = useWriteContract();

  const { isLoading: isConfirming, isSuccess } = useWaitForTransactionReceipt({
    hash,
  });

  const handleTransfer = () => {
    writeContract({
      address: '0x...',
      abi: erc20Abi,
      functionName: 'transfer',
      args: ['0x...', 100n],
    });
  };

  return (
    <div>
      <button onClick={handleTransfer} disabled={isPending}>
        {isPending ? '确认中...' : '转账'}
      </button>
      {isConfirming && <div>等待确认...</div>}
      {isSuccess && <div>交易成功！</div>}
    </div>
  );
}
```

**常用 Hooks 速查表：**

| Hook | 用途 | 返回值 |
|------|-----|--------|
| useAccount | 获取账户信息 | address, isConnected, chain |
| useConnect | 连接钱包 | connect, connectors, isPending |
| useDisconnect | 断开钱包 | disconnect |
| useBalance | 查询 ETH 余额 | data, isLoading, error |
| useReadContract | 读取合约 | data, isLoading, refetch |
| useWriteContract | 写入合约 | writeContract, data (hash), isPending |
| useWaitForTransactionReceipt | 等待交易确认 | isLoading, isSuccess, data |

---

## 4. 【最小可用】

掌握以下内容，就能构建基础 DApp 前端：

### 4.1 项目初始化

```bash
# 创建 React 项目
npm create vite@latest my-dapp -- --template react-ts

# 安装依赖
npm install wagmi viem @tanstack/react-query
```

### 4.2 配置 wagmi

```typescript
// src/config.ts
import { createConfig, http } from 'wagmi';
import { mainnet, sepolia } from 'wagmi/chains';
import { injected } from 'wagmi/connectors';

export const config = createConfig({
  chains: [mainnet, sepolia],
  connectors: [injected()],
  transports: {
    [mainnet.id]: http(),
    [sepolia.id]: http(),
  },
});
```

### 4.3 设置 Provider

```typescript
// src/main.tsx
import { WagmiProvider } from 'wagmi';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { config } from './config';
import App from './App';

const queryClient = new QueryClient();

ReactDOM.createRoot(document.getElementById('root')!).render(
  <WagmiProvider config={config}>
    <QueryClientProvider client={queryClient}>
      <App />
    </QueryClientProvider>
  </WagmiProvider>
);
```

### 4.4 连接钱包

```typescript
// src/ConnectButton.tsx
import { useAccount, useConnect, useDisconnect } from 'wagmi';

export function ConnectButton() {
  const { address, isConnected } = useAccount();
  const { connect, connectors } = useConnect();
  const { disconnect } = useDisconnect();

  if (isConnected) {
    return (
      <div>
        <span>{address?.slice(0, 6)}...{address?.slice(-4)}</span>
        <button onClick={() => disconnect()}>断开</button>
      </div>
    );
  }

  return (
    <button onClick={() => connect({ connector: connectors[0] })}>
      连接钱包
    </button>
  );
}
```

### 4.5 读取合约

```typescript
// src/Balance.tsx
import { useReadContract } from 'wagmi';
import { formatUnits } from 'viem';

const ERC20_ABI = [
  {
    name: 'balanceOf',
    type: 'function',
    inputs: [{ name: 'account', type: 'address' }],
    outputs: [{ name: 'balance', type: 'uint256' }],
    stateMutability: 'view',
  },
] as const;

export function TokenBalance({ tokenAddress, userAddress }) {
  const { data, isLoading } = useReadContract({
    address: tokenAddress,
    abi: ERC20_ABI,
    functionName: 'balanceOf',
    args: [userAddress],
  });

  if (isLoading) return <span>加载中...</span>;
  return <span>{formatUnits(data ?? 0n, 18)} 代币</span>;
}
```

---

**这些知识足以：**
- ✅ 搭建 DApp 前端项目
- ✅ 实现钱包连接/断开
- ✅ 读取链上数据和合约状态
- ✅ 为进阶功能（写入合约、交易状态）打基础

---

## 5. 【1个类比】

### 类比1：wagmi + viem = 银行服务系统 🏦

#### 生活场景类比：银行柜台服务

想象你去银行办业务：

**银行服务对应关系：**

| 银行概念 | wagmi/viem | 说明 |
|---------|------------|------|
| 银行总部系统 | viem | 底层处理所有业务的核心系统 |
| 柜台服务窗口 | wagmi | 用户友好的服务接口 |
| 叫号排队系统 | 请求去重/缓存 | 相同请求不重复处理 |
| 银行卡 | 钱包地址 | 你的身份标识 |
| 查询余额 | publicClient | 免费查看，不需要签字 |
| 转账汇款 | walletClient | 需要签字（私钥签名） |
| 业务回执 | 交易哈希 | 证明操作已执行 |

**举例：**

```
去银行转账的流程：
1. 拿银行卡 → 连接钱包 (useConnect)
2. 到柜台 → 调用 wagmi Hook
3. 填转账单 → 设置交易参数
4. 签字确认 → 钱包签名
5. 等待处理 → 等待交易确认
6. 拿回执 → 获取交易哈希
```

```typescript
// 对应的代码流程
const { connect } = useConnect();           // 1. 拿银行卡
const { writeContract } = useWriteContract(); // 2. 到柜台

writeContract({                             // 3-4. 填单+签字
  address: tokenAddress,
  abi: erc20Abi,
  functionName: 'transfer',
  args: [recipient, amount],
});

const { isSuccess } = useWaitForTransactionReceipt({ hash }); // 5-6. 等待+回执
```

---

#### 前端领域类比：React Query / SWR

如果你用过 React Query 或 SWR，wagmi 的设计理念非常相似：

**React Query 对比：**

```typescript
// React Query 获取 API 数据
const { data, isLoading, error } = useQuery({
  queryKey: ['user', userId],
  queryFn: () => fetch(`/api/user/${userId}`).then(r => r.json()),
});

// wagmi 读取合约数据
const { data, isLoading, error } = useReadContract({
  address: contractAddress,
  abi: contractAbi,
  functionName: 'balanceOf',
  args: [userAddress],
});
```

**相似的设计模式：**

| React Query | wagmi | 说明 |
|-------------|-------|------|
| queryKey | address + functionName + args | 缓存键 |
| queryFn | 内置 JSON-RPC 调用 | 数据获取函数 |
| staleTime | 可配置 | 数据过期时间 |
| refetch | refetch | 手动刷新 |
| useQuery | useReadContract | 读取数据 |
| useMutation | useWriteContract | 修改数据 |

**代码对比：**

```typescript
// ===== React Query 模式 =====
// 读取
const { data } = useQuery({ queryKey: ['balance'], queryFn: fetchBalance });
// 写入
const { mutate } = useMutation({ mutationFn: sendTransfer });

// ===== wagmi 模式 =====
// 读取
const { data } = useReadContract({ functionName: 'balanceOf', ... });
// 写入
const { writeContract } = useWriteContract();
```

**viem 对比 axios：**

```typescript
// axios 发送 HTTP 请求
const response = await axios.get('/api/data');

// viem 发送 JSON-RPC 请求
const balance = await publicClient.getBalance({ address });
```

---

### 类比2：Config = 路由配置 🛣️

#### 生活场景类比：GPS 导航设置

想象你的车载 GPS 导航：

```
GPS 设置：
├── 支持的地图: 高德、百度、谷歌
├── 语音助手: 小度、Siri
└── 路线偏好: 高速优先、避开收费
```

```
wagmi Config：
├── chains: mainnet, sepolia    // 支持的"地图"
├── connectors: MetaMask, WC    // 连接"助手"
└── transports: RPC endpoints   // "路线"配置
```

#### 前端领域类比：React Router 配置

```typescript
// React Router 配置
const router = createBrowserRouter([
  { path: '/', element: <Home /> },
  { path: '/about', element: <About /> },
]);

// wagmi 配置
const config = createConfig({
  chains: [mainnet, sepolia],
  connectors: [injected()],
  transports: { ... },
});

// 使用方式也相似
<RouterProvider router={router} />
<WagmiProvider config={config} />
```

---

### 类比总结表

| wagmi/viem 概念 | 生活场景类比 | 前端领域类比 |
|----------------|-------------|-------------|
| Config | GPS 导航设置 | React Router 配置 |
| WagmiProvider | 银行服务窗口 | QueryClientProvider |
| publicClient | 免费查询服务 | axios.get |
| walletClient | 需签字的业务 | axios.post (带认证) |
| useReadContract | 查询余额 | useQuery |
| useWriteContract | 转账汇款 | useMutation |
| useAccount | 银行卡信息 | useAuth |
| useConnect | 插卡认证 | useLogin |
| 交易哈希 | 业务回执单 | 响应 ID |

---

## 6. 【反直觉点】

### 误区1：wagmi 和 viem 必须一起使用 ❌

**为什么错？**

虽然 wagmi 内部使用 viem，但它们可以独立使用：
- **只用 viem**：不需要 React，在 Node.js 或纯 JS 环境中使用
- **只用 wagmi**：大多数场景不需要直接使用 viem API

```typescript
// 场景1：Node.js 脚本，只用 viem
import { createPublicClient, http } from 'viem';
import { mainnet } from 'viem/chains';

const client = createPublicClient({
  chain: mainnet,
  transport: http(),
});

const blockNumber = await client.getBlockNumber();
```

```typescript
// 场景2：React DApp，主要用 wagmi
import { useBalance } from 'wagmi';

function App() {
  // 不需要直接导入 viem
  const { data } = useBalance({ address: '0x...' });
  return <div>{data?.formatted}</div>;
}
```

**为什么人们容易这样错？**

因为文档经常把它们放在一起介绍，容易认为是"捆绑销售"。实际上它们是两个独立的库，只是配合得很好。

**正确理解：**

| 场景 | 推荐方案 |
|-----|---------|
| React DApp | wagmi（内置 viem） |
| Node.js 脚本 | 只用 viem |
| 复杂底层操作 | wagmi + 直接使用 viem |

---

### 误区2：wagmi v2 和 v1 的 API 相同 ❌

**为什么错？**

wagmi v2 进行了重大重构，很多 API 都改名或改变了用法：

```typescript
// ===== wagmi v1 (旧) =====
import { useContractRead, useContractWrite } from 'wagmi';

const { data } = useContractRead({
  addressOrName: '0x...',  // 旧参数名
  contractInterface: abi,   // 旧参数名
  functionName: 'balanceOf',
});

const { write } = useContractWrite({
  mode: 'recklesslyUnprepared',  // v2 已移除
  // ...
});

// ===== wagmi v2 (新) =====
import { useReadContract, useWriteContract } from 'wagmi';

const { data } = useReadContract({
  address: '0x...',        // 新参数名
  abi: abi,                // 新参数名
  functionName: 'balanceOf',
});

const { writeContract } = useWriteContract();
// 调用方式也变了
writeContract({ address, abi, functionName, args });
```

**主要变化：**

| v1 API | v2 API | 变化说明 |
|--------|--------|---------|
| useContractRead | useReadContract | 改名 |
| useContractWrite | useWriteContract | 改名 + 调用方式变化 |
| usePrepareContractWrite | 移除 | 不再需要预准备 |
| useWaitForTransaction | useWaitForTransactionReceipt | 改名 |
| write() | writeContract() | 函数名变化 |

**为什么人们容易这样错？**

网上很多教程和 StackOverflow 答案还是 v1 的代码，直接复制会报错。

**正确理解：**

1. 检查 wagmi 版本：`npm list wagmi`
2. v2 查看官方文档：https://wagmi.sh
3. 注意 import 来源的变化

---

### 误区3：useReadContract 每次渲染都会发请求 ❌

**为什么错？**

wagmi 内置了智能缓存机制，相同的请求不会重复发送：

```typescript
function Component() {
  // 这个请求会被缓存
  const { data } = useReadContract({
    address: '0x...',
    abi: erc20Abi,
    functionName: 'balanceOf',
    args: ['0x...'],
  });

  return <div>{data}</div>;
}

function App() {
  return (
    <>
      {/* 虽然渲染了两个组件，但只发一次请求 */}
      <Component />
      <Component />
    </>
  );
}
```

**缓存机制说明：**

```typescript
// 缓存键 = address + abi + functionName + args + chainId
// 相同的缓存键会复用结果

// 这两个请求共享缓存
useReadContract({ address: '0xA', functionName: 'balanceOf', args: ['0x1'] });
useReadContract({ address: '0xA', functionName: 'balanceOf', args: ['0x1'] });

// 这两个请求是独立的（args 不同）
useReadContract({ address: '0xA', functionName: 'balanceOf', args: ['0x1'] });
useReadContract({ address: '0xA', functionName: 'balanceOf', args: ['0x2'] });
```

**为什么人们容易这样错？**

来自传统 useEffect + fetch 的心智模型，担心重复请求。wagmi 基于 TanStack Query，自动处理了这些问题。

**正确理解：**

- 相同请求自动去重
- 数据会被缓存（可配置过期时间）
- 需要刷新时使用 `refetch()`

```typescript
const { data, refetch } = useReadContract({ ... });

// 手动刷新数据
<button onClick={() => refetch()}>刷新</button>
```

---

## 7. 【实战代码】

### 基础实现：完整的 DApp 项目结构

```typescript
// ===== 1. 配置文件 src/config.ts =====

import { createConfig, http } from 'wagmi';
import { mainnet, sepolia } from 'wagmi/chains';
import { injected, walletConnect } from 'wagmi/connectors';

// WalletConnect 项目 ID（从 https://cloud.walletconnect.com 获取）
const projectId = 'YOUR_WALLETCONNECT_PROJECT_ID';

export const config = createConfig({
  chains: [mainnet, sepolia],
  connectors: [
    injected(),
    walletConnect({ projectId }),
  ],
  transports: {
    [mainnet.id]: http(),
    [sepolia.id]: http(),
  },
});

// 导出类型供 TypeScript 使用
declare module 'wagmi' {
  interface Register {
    config: typeof config;
  }
}
```

```typescript
// ===== 2. 入口文件 src/main.tsx =====

import React from 'react';
import ReactDOM from 'react-dom/client';
import { WagmiProvider } from 'wagmi';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { config } from './config';
import App from './App';
import './index.css';

// 创建 QueryClient 实例
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60, // 1分钟后数据过期
      retry: 3,             // 失败重试3次
    },
  },
});

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <WagmiProvider config={config}>
      <QueryClientProvider client={queryClient}>
        <App />
      </QueryClientProvider>
    </WagmiProvider>
  </React.StrictMode>
);
```

```typescript
// ===== 3. 钱包连接组件 src/components/ConnectWallet.tsx =====

import { useAccount, useConnect, useDisconnect, useEnsName } from 'wagmi';

export function ConnectWallet() {
  const { address, isConnected, chain } = useAccount();
  const { connect, connectors, isPending, error } = useConnect();
  const { disconnect } = useDisconnect();
  const { data: ensName } = useEnsName({ address });

  // 已连接状态
  if (isConnected) {
    return (
      <div className="wallet-info">
        <div className="address">
          {ensName ?? `${address?.slice(0, 6)}...${address?.slice(-4)}`}
        </div>
        <div className="chain">
          {chain?.name ?? 'Unknown Chain'}
        </div>
        <button onClick={() => disconnect()} className="disconnect-btn">
          断开连接
        </button>
      </div>
    );
  }

  // 未连接状态：显示可用钱包列表
  return (
    <div className="connect-options">
      <h3>选择钱包</h3>
      {connectors.map((connector) => (
        <button
          key={connector.uid}
          onClick={() => connect({ connector })}
          disabled={isPending}
          className="wallet-btn"
        >
          {connector.name}
          {isPending && ' (连接中...)'}
        </button>
      ))}
      {error && <div className="error">{error.message}</div>}
    </div>
  );
}
```

```typescript
// ===== 4. 余额显示组件 src/components/Balance.tsx =====

import { useAccount, useBalance } from 'wagmi';
import { formatUnits } from 'viem';

export function Balance() {
  const { address, isConnected } = useAccount();

  // 查询 ETH 余额
  const {
    data: ethBalance,
    isLoading: ethLoading,
    error: ethError,
    refetch: refetchEth,
  } = useBalance({
    address,
  });

  if (!isConnected) {
    return <div>请先连接钱包</div>;
  }

  if (ethLoading) {
    return <div>加载余额中...</div>;
  }

  if (ethError) {
    return <div>获取余额失败: {ethError.message}</div>;
  }

  return (
    <div className="balance-card">
      <h3>账户余额</h3>
      <div className="balance-item">
        <span className="label">ETH:</span>
        <span className="value">
          {ethBalance?.formatted} {ethBalance?.symbol}
        </span>
      </div>
      <button onClick={() => refetchEth()}>刷新</button>
    </div>
  );
}
```

```typescript
// ===== 5. 合约读取组件 src/components/TokenBalance.tsx =====

import { useReadContract } from 'wagmi';
import { formatUnits } from 'viem';

// ERC20 ABI（只需要用到的函数）
const erc20Abi = [
  {
    name: 'balanceOf',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'account', type: 'address' }],
    outputs: [{ name: '', type: 'uint256' }],
  },
  {
    name: 'symbol',
    type: 'function',
    stateMutability: 'view',
    inputs: [],
    outputs: [{ name: '', type: 'string' }],
  },
  {
    name: 'decimals',
    type: 'function',
    stateMutability: 'view',
    inputs: [],
    outputs: [{ name: '', type: 'uint8' }],
  },
] as const;

interface TokenBalanceProps {
  tokenAddress: `0x${string}`;
  userAddress: `0x${string}`;
}

export function TokenBalance({ tokenAddress, userAddress }: TokenBalanceProps) {
  // 并行读取多个合约数据
  const { data: balance, isLoading: balanceLoading } = useReadContract({
    address: tokenAddress,
    abi: erc20Abi,
    functionName: 'balanceOf',
    args: [userAddress],
  });

  const { data: symbol } = useReadContract({
    address: tokenAddress,
    abi: erc20Abi,
    functionName: 'symbol',
  });

  const { data: decimals } = useReadContract({
    address: tokenAddress,
    abi: erc20Abi,
    functionName: 'decimals',
  });

  if (balanceLoading) {
    return <span>加载中...</span>;
  }

  const formattedBalance = balance && decimals
    ? formatUnits(balance, decimals)
    : '0';

  return (
    <div className="token-balance">
      <span>{formattedBalance}</span>
      <span>{symbol ?? 'TOKEN'}</span>
    </div>
  );
}
```

```typescript
// ===== 6. 主应用 src/App.tsx =====

import { ConnectWallet } from './components/ConnectWallet';
import { Balance } from './components/Balance';
import { TokenBalance } from './components/TokenBalance';
import { useAccount } from 'wagmi';

// USDC 合约地址（主网）
const USDC_ADDRESS = '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48';

function App() {
  const { address, isConnected } = useAccount();

  return (
    <div className="app">
      <header>
        <h1>我的 DApp</h1>
        <ConnectWallet />
      </header>

      <main>
        {isConnected && (
          <>
            <Balance />

            <div className="token-section">
              <h3>代币余额</h3>
              <TokenBalance
                tokenAddress={USDC_ADDRESS}
                userAddress={address!}
              />
            </div>
          </>
        )}

        {!isConnected && (
          <div className="welcome">
            <h2>欢迎使用 DApp</h2>
            <p>请连接钱包开始使用</p>
          </div>
        )}
      </main>
    </div>
  );
}

export default App;
```

**运行输出示例：**

```
┌─────────────────────────────────────┐
│  我的 DApp          [已连接: 0x12...ab] │
├─────────────────────────────────────┤
│                                     │
│  账户余额                            │
│  ETH: 1.234 ETH         [刷新]      │
│                                     │
│  代币余额                            │
│  USDC: 1,000.00 USDC                │
│                                     │
└─────────────────────────────────────┘
```

---

### 进阶：使用 viem 直接操作

```typescript
// ===== 在 React 组件外使用 viem =====

import { createPublicClient, http, parseAbi } from 'viem';
import { mainnet } from 'viem/chains';

// 创建公共客户端
const publicClient = createPublicClient({
  chain: mainnet,
  transport: http('https://eth.llamarpc.com'),
});

// 批量读取示例
async function getMultipleBalances(addresses: `0x${string}`[]) {
  const results = await publicClient.multicall({
    contracts: addresses.map(address => ({
      address: '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48', // USDC
      abi: parseAbi(['function balanceOf(address) view returns (uint256)']),
      functionName: 'balanceOf',
      args: [address],
    })),
  });

  return results.map((result, i) => ({
    address: addresses[i],
    balance: result.status === 'success' ? result.result : 0n,
  }));
}

// 监听区块
const unwatch = publicClient.watchBlocks({
  onBlock: (block) => {
    console.log('新区块:', block.number);
  },
});

// 取消监听
// unwatch();
```

---

## 8. 【面试必问】

### 问题1："为什么选择 wagmi + viem 而不是 ethers.js？"

**普通回答（❌ 不出彩）：**

"wagmi 更新，viem 更轻量。"

**出彩回答（✅ 推荐）：**

> **选择 wagmi + viem 有三个核心原因：**
>
> **1. 类型安全**
> viem 是 TypeScript-first 设计，提供完整的类型推断：
> ```typescript
> // ethers.js：返回类型是 any
> const balance = await contract.balanceOf(address);
>
> // viem：自动推断返回类型为 bigint
> const balance = await publicClient.readContract({
>   abi: erc20Abi,
>   functionName: 'balanceOf',
>   args: [address],
> }); // balance: bigint
> ```
>
> **2. 包体积**
> - ethers.js v6: ~120KB (gzipped)
> - viem: ~35KB (gzipped)
> - viem 小约 70%，对 DApp 首屏加载很重要
>
> **3. React 集成**
> wagmi 专门为 React 设计，内置：
> - 请求去重和缓存（基于 TanStack Query）
> - 自动重试和后台刷新
> - 状态管理（loading/error/data）
>
> 而 ethers.js 需要自己用 useEffect + useState 管理状态。
>
> **实际项目中的体验：**
> - 开发效率提升：Hooks API 更符合 React 心智模型
> - 维护成本降低：类型错误在编译时就能发现
> - 用户体验更好：更小的包体积 + 内置缓存优化

**为什么这个回答出彩？**
1. ✅ 给出了三个具体的技术原因
2. ✅ 有代码对比说明类型安全
3. ✅ 有具体的数据（包体积）
4. ✅ 联系了实际项目体验

---

### 问题2："wagmi 的缓存机制是怎么工作的？"

**普通回答（❌ 不出彩）：**

"wagmi 会缓存请求结果，相同请求不会重复发。"

**出彩回答（✅ 推荐）：**

> **wagmi 的缓存基于 TanStack Query，有几个关键机制：**
>
> **1. 缓存键生成**
> ```typescript
> // 缓存键 = chainId + address + functionName + args
> useReadContract({
>   address: '0xA',
>   functionName: 'balanceOf',
>   args: ['0x1'],
> });
> // 缓存键类似: '1:0xA:balanceOf:0x1'
> ```
>
> **2. 去重机制**
> ```typescript
> // 同时多个组件请求同一数据，只发一次请求
> <ComponentA /> // 发起请求
> <ComponentB /> // 共享 ComponentA 的结果
> ```
>
> **3. 过期策略**
> - `staleTime`: 数据被认为"新鲜"的时间，新鲜期内不会重新请求
> - `gcTime`: 数据在缓存中保留的时间
>
> ```typescript
> useReadContract({
>   address: '0x...',
>   functionName: 'balanceOf',
>   args: [address],
>   query: {
>     staleTime: 1000 * 60,  // 1分钟内认为数据新鲜
>     gcTime: 1000 * 60 * 5, // 5分钟后清除缓存
>   },
> });
> ```
>
> **4. 后台刷新**
> 数据过期后，wagmi 会在后台刷新，同时显示旧数据，新数据到达后更新 UI。
>
> **实际应用：**
> - 高频数据（区块号）：设置短 staleTime
> - 稳定数据（代币 symbol）：可以设置长 staleTime 甚至永不过期
> - 用户余额：根据业务需求平衡实时性和性能

**为什么这个回答出彩？**
1. ✅ 解释了底层原理（TanStack Query）
2. ✅ 说明了缓存键的生成方式
3. ✅ 区分了不同的配置项
4. ✅ 给出了实际应用场景

---

## 9. 【化骨绵掌】

### 卡片1：直觉理解 - 它们是什么？ 🎯

**一句话：** viem 是与区块链通信的底层库，wagmi 是 React Hooks 封装。

**举例：**
```
viem  = axios（发请求的工具）
wagmi = React Query（管理请求状态的框架）
```

**应用：** React DApp 开发首选 wagmi + viem 组合。

---

### 卡片2：形式化定义 - Config 配置 📐

**一句话：** Config 定义了支持的链、钱包和 RPC，是整个应用的基础设置。

```typescript
const config = createConfig({
  chains: [mainnet],
  connectors: [injected()],
  transports: { [mainnet.id]: http() },
});
```

**应用：** 通过 WagmiProvider 注入到 React 组件树。

---

### 卡片3：关键概念1 - Public Client 🔍

**一句话：** publicClient 用于只读操作，不需要钱包，任何人都可以查询区块链数据。

```typescript
const blockNumber = await publicClient.getBlockNumber();
const balance = await publicClient.readContract({ ... });
```

**应用：** 读取余额、查询合约状态、获取区块信息。

---

### 卡片4：关键概念2 - Wallet Client 💳

**一句话：** walletClient 用于写入操作，需要钱包签名，会消耗 Gas。

```typescript
const hash = await walletClient.writeContract({
  functionName: 'transfer',
  args: [recipient, amount],
});
```

**应用：** 转账、调用状态变更函数、部署合约。

---

### 卡片5：编程实现 - Hooks 使用 🪝

**一句话：** wagmi 提供一系列 React Hooks，自动处理 loading/error/data 状态。

```typescript
const { data, isLoading, error } = useReadContract({ ... });
const { writeContract, isPending } = useWriteContract();
```

**应用：** 在 React 组件中优雅地处理区块链交互。

---

### 卡片6：对比区分 - vs ethers.js 🆚

**一句话：** viem 更轻（35KB vs 120KB）、类型更好，wagmi 专为 React 设计。

| 特性 | ethers.js | wagmi + viem |
|-----|----------|--------------|
| 包大小 | ~120KB | ~35KB |
| TypeScript | 一般 | 优秀 |
| React 集成 | 需手动 | 内置 |

**应用：** 新项目优先选择 wagmi + viem。

---

### 卡片7：进阶理解 - 缓存机制 🗄️

**一句话：** wagmi 基于 TanStack Query，自动去重、缓存、后台刷新。

```typescript
// 同一数据多个组件请求，只发一次
useReadContract({ ... }); // 组件 A
useReadContract({ ... }); // 组件 B（共享缓存）
```

**应用：** 无需手动优化请求，wagmi 自动处理。

---

### 卡片8：高级应用 - Multicall 批量读取 📦

**一句话：** 使用 multicall 将多个读取请求合并为一次 RPC 调用。

```typescript
const results = await publicClient.multicall({
  contracts: [
    { address, abi, functionName: 'balanceOf', args: [user1] },
    { address, abi, functionName: 'balanceOf', args: [user2] },
  ],
});
```

**应用：** 批量读取多个地址的余额，减少 RPC 调用次数。

---

### 卡片9：实际项目中的应用 🌐

**一句话：** 主流 DApp（Uniswap、OpenSea）都在使用 wagmi。

**典型架构：**
```
React App
├── WagmiProvider（配置）
├── QueryClientProvider（缓存）
├── ConnectButton（钱包连接）
├── 业务组件（使用 Hooks）
```

**应用：** 参考成熟项目的架构设计自己的 DApp。

---

### 卡片10：总结与延伸 🎓

**一句话：** wagmi + viem 是现代 DApp 前端的最佳实践。

**核心要点：**
1. viem = 底层库，wagmi = React 封装
2. Config 定义链、钱包、RPC
3. publicClient 读，walletClient 写
4. Hooks 自动管理状态
5. 内置缓存和去重

**下一步学习：**
- 钱包连接的完整流程
- 合约读写的高级用法
- 交易状态管理

---

## 10. 【一句话总结】

**wagmi 是专为 React 设计的以太坊交互 Hooks 库，viem 是轻量级 TypeScript 以太坊客户端，二者组合提供了类型安全、高性能、开发体验优秀的 DApp 前端解决方案，是 ethers.js 的现代替代品。**

---

## 📚 附录

### 学习检查清单

完成本知识点学习后，你应该能够：

- [ ] 理解 wagmi 和 viem 的定位和区别
- [ ] 创建 wagmi Config 配置
- [ ] 使用 WagmiProvider 包装应用
- [ ] 使用 useAccount 获取账户信息
- [ ] 使用 useReadContract 读取合约数据
- [ ] 区分 publicClient 和 walletClient
- [ ] 理解 wagmi 的缓存机制

### 快速参考卡

**安装：**
```bash
npm install wagmi viem @tanstack/react-query
```

**配置：**
```typescript
import { createConfig, http } from 'wagmi';
import { mainnet } from 'wagmi/chains';
import { injected } from 'wagmi/connectors';

const config = createConfig({
  chains: [mainnet],
  connectors: [injected()],
  transports: { [mainnet.id]: http() },
});
```

**常用 Hooks：**
```typescript
useAccount()      // 账户信息
useConnect()      // 连接钱包
useDisconnect()   // 断开钱包
useBalance()      // ETH 余额
useReadContract() // 读取合约
useWriteContract() // 写入合约
```

### 下一步学习

推荐按以下顺序继续学习：

1. **钱包连接** - 深入了解连接器和状态管理
2. **合约读写** - 掌握完整的合约交互流程
3. **交易状态处理** - 处理 pending/success/error
4. **事件监听** - 实时更新 UI

---

**版本：** v1.0
**创建日期：** 2025-12-12
**作者：** Web3学习助手
**适用人群：** 前端工程师转Web3开发
