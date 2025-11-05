[pinocchio - Rust](https://docs.rs/pinocchio/latest/pinocchio/index.html)

[Blueshift | Pinocchio for Dummies | Pinocchio 101](https://learn.blueshift.gg/zh-CN/courses/pinocchio-for-dummies/pinocchio-101)





**Allocator in solana**

pinocchio使用的[bump allocator]([Bump allocation - Writing Interpreters in Rust: a Guide](https://rust-hosted-langs.github.io/book/chapter-simple-bump.html))来进行内存的调度和分配，bump allocator是线性分配器或指针碰撞分配器。它通过维护一个指针（通常称为`offset`或`top`）来工作。这个指针指向内存池中下一个可用的地址。当需要分配内存时，分配器将指针向前移动所需的大小（并考虑对齐要求），然后返回移动前的指针值。释放内存通常不是逐个对象进行，而是通过重置指针到某个之前的状态来释放所有或一批对象。

举个例子说明其工作流：

初始状态

```
内存区域：[0, 1, 2, 3, ..., 99]  (共100字节)
offset = 0（指针指向0区域）
```

### 第一步：`alloc(3, align=1)`

- 大小：3字节
- 内存对齐：1字节（任意地址都满足）
- 计算：
  - 当前 offset = 0
  - 对齐后地址 = 0（因为0已经是1的倍数）
  - 检查：0 + 3 = 3 ≤ 100 ✅
- **结果**：返回地址0，offset变为3

text

```
内存使用：[已用, 已用, 已用, 空闲, 空闲...]
          0    1    2    3    4...
```



### 第二步：`alloc(8, align=8)`

- 大小：8字节
- 对齐：8字节（地址必须是8的倍数，内存对其的意义在于直接读取而无需拼接数据，不懂搜b站）
- 计算：
  - 当前 offset = 3
  - 对齐检查：3不是8的倍数，需要找到下一个8的倍数
  - 对齐后地址 = 8
  - 检查：8 + 8 = 16 ≤ 100 ✅
- **结果**：返回地址8，offset变为16

```
内存使用：[已用, 已用, 已用, 填充, 填充, 填充, 填充, 填充, 已用, 已用...]
          0    1     2    3    4    5    6    7    8    9...
          ↑第一个分配                                ↑第二个分配从8开始
```



### 第三步：`alloc(5, align=4)`

- 大小：9字节（数据是结构体struct {u32,u8,u32}）
- 对齐：4字节（地址必须是4的倍数）
- 计算：
  - 当前 offset = 16
  - 对齐检查：16正好是4的倍数 ✅
  - 对齐后地址 = 16
  - 检查：16 + 5 = 21 ≤ 100 ✅
- **结果**：返回地址16，offset变为21

text

```
内存使用：[... u32,u32,u32,u32, u8, 空，空，空， u32,u32,u32,u32]
内存使用：[... 16, 17, 18, 19,  20, 21,22,23,  24,25,26,27]
             ↑第三个分配从这里开始
```