[TOC]



## 1. EVM数据存储方式

### 1.1 计算机数据存储体系

在虚拟机中，主要有三种虚拟机架构：

- Register Based Virtual Machines（寄存器）

- Stack Based Virtual Machines（堆栈）
- 混合架构（如**WebAssembly**）

而EVM是Stack-Based虚拟机，Solana则是基于寄存器的rBPF虚拟机运行的。**栈式虚拟机**胜在**简单性**和**可移植性**，适合快速实现和内存敏感场景。**寄存器型虚拟机**赢在**性能**和**优化潜力**，适合高性能计算和底层系统。

本文将主要介绍Stack-based Virtural Machines的运作原理



#### 1.1.1 栈（Stack）

Stack是EVM的主要工作空间。所有从 "storage(存储)"、"memory(内存)"或 "calldata "加载的东西都是使用与每个数据位置相关的操作码（`SLOAD`用于存储，`MLOAD `用于内存，`CALLDATALOAD `用于calldata）加载到堆栈。

**基础特性**：

- LIFO（后进先出）结构，最大深度1024元素
- 操作成本最低（gas消耗远低于memory/storage）
- 每个元素为256-bit字长
- 核心操作指令：
  - `PUSHx`（x=1~32字节数据长度）
  - `POP`/`DUP`/`SWAP`系列指令

**计算案例**：

```assembly
PUSH1 0x03    ; 栈顶: [3]
PUSH1 0x05    ; 栈顶: [5, 3]
ADD           ; 弹出5和3，压入8
```

#### 1.1.2 内存（Memory）

**关键特征**：

- 动态扩展的字节数组（每次扩展32字节）
- 易失性存储（交易结束后清零）
- 访问模式：
  - `MLOAD`：读取32字节（起始地址需32字节对齐）
  - `MSTORE`：写入32字节
  - `MSTORE8`：写入单个字节

**Gas成本**：

- 初始访问：3 gas/32字节
- 扩展内存：按quadratic公式计算

#### 1.1.3 调用数据（Calldata）

**核心特点**：

- 只读数据区（不可修改）
- 存储交易原始输入数据
- 高效访问方式：
  - `CALLDATALOAD`：从指定偏移量读取32字节
  - `CALLDATASIZE`：获取数据总长度
  - `CALLDATACOPY`：复制到内存

**字节序特性**：

- 大端序存储（网络字节序）
- 案例：数值`0x12345678`在内存中的存储：
  - 地址0x00: 0x12
  - 地址0x01: 0x34
  - 地址0x02: 0x56
  - 地址0x03: 0x78

### 1.2 合约字节码结构
#### 合约创建代码（Contract Creation Code）

- 功能：部署合约到区块链
- 核心指令：
  - `CODECOPY`：复制运行时代码到内存
  - `RETURN`：返回运行时代码

#### 运行时代码（Runtime Code）

- 包含实际业务逻辑
- 必须包含函数调度器（Function Dispatcher）

#### 元数据（Metadata）

- 可选部分（Solidity默认添加）
- 包含ABI信息、源码哈希等

### 1.3 函数调度机制

**处理流程**：
1. 提取calldata前4字节作为函数选择器
2. 匹配合约函数签名哈希
3. 跳转到对应函数入口

**Huff实现示例**：
```huff
#define macro MAIN() = takes(0) returns(0) {
    0x00             // 偏移量0
    calldataload     // 加载32字节calldata
    0xe0             // 右移224位（32-4字节）
    shr              // 得到4字节函数选择器
    0xcdfead2e       // 目标函数签名
    eq               // 比较选择器
    <jumpdest> jumpi // 条件跳转
}
```

## 2. 关键操作码解析
### 2.1 CALLDATALOAD
**语法**：`CALLDATALOAD <offset>`
- 从指定偏移量读取32字节
- 不足32字节时右侧补零

**案例**：
```
Calldata: 0xFFFFFFFF...FFF1 (32字节)
PUSH1 0x01
CALLDATALOAD → 0xFF...F100
```

### 2.2 SHR
**位移操作**：
- 语法：`SHR <shift_bits> <value>`
- 应用场景：提取函数选择器

**操作示例**：
```assembly
PUSH4 0x12345678  // 原始数据
PUSH1 0x04         // 移位量（4 bits）
SHR                // 结果：0x01234567
```

**函数选择器提取**：
```assembly
0x00
calldataload      // 加载完整32字节
0xe0              // 224位偏移量（32-4字节）
shr               // 结果保留前4字节
```

## 3. 开发实践要点
### 3.1 内存管理
- 预先计算内存需求，避免频繁扩展
- 使用`MSIZE`获取当前内存大小
- 注意32字节对齐访问提升效率

### 3.2 Calldata处理
- 验证`CALLDATASIZE`确保参数完整
- 使用掩码处理动态类型参数：
  ```assembly
  // 处理uint256参数
  calldataload
  0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
  and
  ```

### 3.3 Gas优化技巧
- 优先使用栈进行中间计算
- 合并内存写操作（批量写入32字节）
- 避免不必要的calldata复制

---

**附录：EVM内存布局示例**
```
+------------------+
| 0x00 - 0x20     | ← MSTORE(0x00, value)
| 32-byte slot     |
+------------------+
| 0x20 - 0x40     |
| Next memory slot |
+------------------+
| ...              |
+------------------+
```

**参考资源**：
- [EVM Opcodes Reference](https://ethereum.org/en/developers/docs/evm/opcodes)
- [Huff Language Documentation](https://docs.huff.sh/)