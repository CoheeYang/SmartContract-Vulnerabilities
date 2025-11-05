

# 宏

当我们了解了[Solana中的交易](https://solana.com/zh/docs/core/transactions#account-addresses)就会知道，最终交易中的账户信息和instructiondata会被传入程序，

```rust
fn process_instruction(
    program_id: &Pubkey,
    _accounts: &[AccountInfo],
    _instruction_data: &[u8],
)->...{
    ...
}
```

账户信息最终会以`AccountInfo`的形式传入，但是native program中没有对这些账户信息的检查，`AccountInfo`是一个完全外来的值，需要手写一些检查代码。而Anchor的一些宏则是生成对这些信息检查，以减少重xie代码：

1. **#[derive(Accounts)]**

`#[derive(Accounts)]`宏应用在一个结构体上，该结构体包含了指令所需的所有账户。这个宏会为结构体生成一个`Accounts` trait的实现，用于反序列化和验证账户。

`Account trait`是一个核心trait，拥有`try_accounts`的方法，该方法会用于：

- **验证账户**：检查账户的所有约束条件（签名、权限、空间等）
- **反序列化**：将原始 `AccountInfo` 转换为类型安全的账户结构体
- **防止重用**：通过消耗账户切片来确保同一账户不会被多次使用

```rust
fn try_accounts(
    program_id: &Pubkey,
    accounts: &mut &'info [AccountInfo<'info>],
    ix_data: &[u8],
    bumps: &mut B,
    reallocs: &mut BTreeSet<Pubkey>,
) -> Result<Self>
```

































































# Ref

[anchor-lang 0.32.1 - Docs.rs](https://docs.rs/crate/anchor-lang/latest)

[Blueshift | Anchor for Dummies | Anchor Accounts](https://learn.blueshift.gg/zh-CN/courses/anchor-for-dummies/anchor-accounts)

[solana-developers/developer-bootcamp-2024](https://github.com/solana-developers/developer-bootcamp-2024/tree/main)

[solana-foundation/curriculum: A collection of resources to teach Solana in universities and bootcamps](https://github.com/solana-foundation/curriculum)