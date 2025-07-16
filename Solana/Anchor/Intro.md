# 宏

Anchor提供了四个基础宏

1. `declare_id`：确定程序部署在链上地址的声明式宏
2. `#[program]`: 确定一个个包含 instruction逻辑的 module的属性宏
3. `#[derive(Accounts)]`: 将instruction需要的数据放在一个struct中，这个struct通过该派生宏继承account trait
4. `#[account]`上述结构体中所用的属性宏，用于创建自定义的acount type

```rust
use anchor_lang::prelude::*;
 
declare_id!("11111111111111111111111111111111");
 
#[program]
mod hello_anchor {
    use super::*;
    pub fn initialize(ctx: Context<Initialize>, data: u64) -> Result<()> {
        //data是u64, copy trait 类型，不会转移所有权
        ctx.accounts.new_account.data = data; 
        
        msg!("Changed data to: {}!", data);
        
        Ok(())//原本的Result<T,E>被重写，返回的是Anchor打包好的Result<T>类型，T为()，类似void
        //最后返回的Result<()>是空值，Anchor会自动把Account<..>类型的结构序列化上lian
    }
}
 
#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init, payer = signer, space = 8 + 8)]
    pub new_account: Account<'info, NewAccount>,
    #[account(mut)]
    pub signer: Signer<'info>,
    pub system_program: Program<'info, System>,
}
 
#[account]
pub struct NewAccount {
    data: u64,
}
```

在`#[program]`下的第一个指令`initialize`，其中输入一个`Context<T>`的参数，这个参数是每个指令必须输入的第一参数

```rust
pub struct Context<'a, 'b, 'c, 'info, T: Bumps> {
    //当前正在执行的程序（smart contract）的公钥（Program ID）
    pub program_id: &'a Pubkey,
    
    //一个由你在 #[derive(Accounts)] 结构体里声明的账户类型实例（此案例中是Initialize）
    //此结构体拥有account trait,
    //是instruction中所需要的account
    pub accounts: &'b mut T,
    
    //给定但未被反序列化或验证的剩余账户
    pub remaining_accounts: &'c [AccountInfo<’info>],
    
    //由 #[derive(Accounts)] 生成。它表示约束验证期间发现的增量种子。
    pub bumps: T::Bumps,
}
```



`#[derive(Accounts)]`下的结构体运行自定义指令需要的所有accounts。

这个宏能使得这个结构继承account trait,省去了验证和序列化/去序列化的步骤。

同时我们会看见如`#[account(..)]`类似的属性在内部字段中，这被称为`account constrains`，而这些限制可以使得输入instruction中的accounts符合特定需求。

比如

- `#[account(init, payer = <target_account>, space = <num_bytes>)]`：

  init使得系统程序创建并初始化账户，payer指定了付租金的账户，space则写明了所占用的空间。

  

- `#[account(mut)]`：保证输入的account类型是mutable的，除了对一般的accounts进行验证外，还有对SPL（Solana Program Library）程序的constraints，便于开发者对如Token程序的验证。

[See More Constraints](https://docs.rs/anchor-lang/latest/anchor_lang/derive.Accounts.html#constraints)

另一方面，我们也能在该结构体下看见各种的account类型，anchor设置的不同的account类型保证了指令能收到预期的数据结构，这些account类型包括如下：

- [**Account<’info, T>**](https://docs.rs/anchor-lang/latest/anchor_lang/accounts/account/index.html)：Wrapper around [`AccountInfo`](https://docs.rs/anchor-lang/latest/anchor_lang/prelude/struct.AccountInfo.html) that verifies program ownership and deserializes underlying data into a Rust type.
- [**AccountInfo<’info>**](https://docs.rs/anchor-lang/latest/anchor_lang/accounts/account_info/index.html)：一个未经检查的账户，可以用作类型。但是，应该使用 [**UncheckedAccount**](https://docs.rs/anchor-lang/latest/anchor_lang/accounts/unchecked_account/struct.UncheckedAccount.html)，因为 [**AccountInfo** 可能会在将来的版本中消失](https://github.com/coral-xyz/anchor/issues/2794)
- [**AccountLoader<’info, T>**](https://docs.rs/anchor-lang/latest/anchor_lang/accounts/account_loader/index.html)：一种类型，用于按需[零拷贝反序列化](https://docs.rs/anchor-lang/latest/anchor_lang/attr.account.html#zero-copy-deserialization) 。这与开发人员使用 **Account** 不同，因为开发人员必须在初始化账户后调用 **load_init**，在账户不可变时调用 **load**，在账户可变时调用 **load_mut**
- [**Box>** 或 **Box>**](https://docs.rs/anchor-lang/latest/anchor_lang/accounts/boxed/index.html)：一种盒子类型，用于保存堆栈空间，因为有时账户对于堆栈来说太大，可能导致堆栈违规 - 对账户进行盒装可以帮助解决这个问题
- [**Interface<’info, T>**](https://docs.rs/anchor-lang/latest/anchor_lang/accounts/interface/index.html)：一种类型，用于在 **Program** 上包装，用于验证账户是否是给定程序集中的一个。它检查预期程序是否包含账户的密钥，以及账户是否可执行
- [**InterfaceAccount<’info, T>**](https://docs.rs/anchor-lang/latest/anchor_lang/accounts/interface_account/index.html)：一个账户容器，用于检查程序所有权并将底层数据反序列化为 Rust 类型
- [**Option>**](https://docs.rs/anchor-lang/latest/anchor_lang/accounts/option/index.html)：用于可选账户的选项类型
- [**Program<’info, T>**](https://docs.rs/anchor-lang/latest/anchor_lang/accounts/program/index.html)：一种类型，用于验证账户是否是给定程序
- [**Signer<’info>**](https://docs.rs/anchor-lang/latest/anchor_lang/accounts/signer/index.html)：一种类型，用于验证账户是否签署了交易
- [**SystemAccount<’info>**](https://docs.rs/anchor-lang/latest/anchor_lang/accounts/system_account/index.html)：一种类型，用于验证账户是否由[系统程序](https://docs.solana.com/developing/runtime-facilities/programs#system-program)拥有
- [**Sysvar<’info, T>**](https://docs.rs/anchor-lang/latest/anchor_lang/accounts/sysvar/struct.Sysvar.html)：一种类型，用于验证账户是否是 [sysvar](https://docs.rs/solana-program/latest/solana_program/sysvar/index.html)，即账户是否是包含有关网络集群、区块链历史和执行交易的动态更新数据的特殊类型。**clock**、**epoch_schedule**、**instructions** 和 **rent** sysvar 对于程序开发很有用
- [**UncheckedAccount<’info>**](https://docs.rs/anchor-lang/latest/anchor_lang/accounts/unchecked_account/index.html)：一个账户容器，特别强调对指定账户未执行任何检查





`#[account]`中，我们可以自定义一个结构体，这个结构体就是变量的合集，`#[account]`宏会创建一个存数据的账户，并设置这个数据账户的owner为`declare_id`中声明的程序地址。

## 宏生成的代码

`#[program]`会产生什么？它会产生如下的代码：

```rust
entrypoint!(entry);

pub fn entry<'info>(
    program_id: &Pubkey,
    accounts: &'info [AccountInfo<'info>],
    data: &[u8],
) -> anchor_lang::solana_program::entrypoint::ProgramResult {
    try_entry(program_id, accounts, data).map_err(|e| {
        // 这里通常会把 Anchor 错误映射成 Solana 原生错误码
        e.into()
    })
}

fn try_entry<'info>(
    program_id: &Pubkey,
    accounts: &'info [AccountInfo<'info>],
    data: &[u8],
) -> anchor_lang::Result<()> {
    // 校验调用的 program_id 是否和 declare_id! 宏中的 ID 一致
    if *program_id != ID {
        return Err(ProgramError::IncorrectProgramId.into());
    }
    // 至少要有 8 字节来承载每个指令的 discriminator
    if data.len() < 8 {
        return Err(ProgramError::InvalidInstructionData.into());
    }
    // 如果上述校验通过，则进入真正的 dispatch 分发逻辑
    dispatch(program_id, accounts, data)
}

fn dispatch<'info>(
    program_id: &Pubkey,
    accounts: &'info [AccountInfo<'info>],
    data: &[u8],
) -> anchor_lang::Result<()> {
    // 前 8 字节是指令的 discriminator（selector）
    let sig_hash: [u8; 8] = data[..8].try_into().unwrap();
    match sig_hash {
        // 如果和 Initialize 指令的 discriminator 匹配，即
        //#[account]
		//pub struct MyState {
  		//pub counter: u64,}对应的8字节哈希

        instruction::Initialize::DISCRIMINATOR => {
            __private::initialize(program_id, accounts, &data[8..])
        }
        // 如果是 IDL 定义的 fallback 调度（动态指令）
        anchor_lang::idl::IDL_IX_TAG_LE => {
            __private::__idl::idl_dispatch(program_id, accounts, &data[8..])
        }
        // 如果都不匹配，则报错
        _ => Err(ProgramError::InvalidInstructionData.into()),
    }
}

```



`#[derive(Accounts)]`生成的implement

```rust
#[derive(Accounts)]
pub struct Initialize<'info> {
  #[account(init, payer = signer, space = 8 + 8)]
  pub new_account: Account<'info, NewAccount>,

  #[account(mut)]
  pub signer: Signer<'info>,

  pub system_program: Program<'info, System>,
}
```

```rust
impl<'info> anchor_lang::Accounts<'info, InitializeBumps> for Initialize<'info> {
  fn try_accounts(
    program_id: &Pubkey,
    accounts: &mut &'info [AccountInfo<'info>],
    ix_data: &[u8],
    bumps: &mut InitializeBumps,
  ) -> anchor_lang::Result<Self> {
    // —— 第一步：拿定义中的 signer
    let signer: Signer<'info> =
      anchor_lang::Accounts::try_accounts(program_id, accounts, ix_data, bumps)
        .map_err(/* 转成 Anchor 错误 */)?;
    
    // —— 第二步：拿 system_program
    let system_program: Program<'info, System> =
      anchor_lang::Accounts::try_accounts(program_id, accounts, ix_data, bumps)
        .map_err(/* 转成 Anchor 错误 */)?;
    
    // ……如果还有其它字段，就继续 try_accounts 拿对应的类型……

    // 所有字段都拿到了，且约束都通过了，就把它们组装成 Initialize 实例返回
    Ok(Initialize { signer, system_program })
  }
}

```



`#[accounts]`的生成的代码：

```rust
// 原始结构体，派生了 Anchor 的序列化／反序列化 trait
#[derive(AnchorSerialize, AnchorDeserialize, Clone)]
pub struct DataAccount {
    pub authority: Pubkey,
}//将账户owner设置为declare!中的地址而准备的struct

// 为链上账户数据写入实现 AccountSerialize
impl anchor_lang::AccountSerialize for DataAccount {
    fn try_serialize<W: std::io::Write>(&self, writer: &mut W) -> anchor_lang::Result<()> {
        // 1. 先写入 8 字节的 discriminator（类型区分符）
        if writer
            .write_all(&[85, 240, 182, 158, 76, 7, 18, 233])
            .is_err()
        {
            return Err(AccountDidNotSerialize.into());
        }
        // 2. 然后再把 struct 里的字段按 AnchorSerialize 规则写入
        if AnchorSerialize::serialize(self, writer).is_err() {
            return Err(AccountDidNotSerialize.into());
        }
        Ok(())
    }
}

// 为链上账户数据读出实现 AccountDeserialize
impl anchor_lang::AccountDeserialize for DataAccount {
    // 带校验的“安全”反序列化：先检查 discriminator
    fn try_deserialize(buf: &mut &[u8]) -> anchor_lang::Result<Self> {
        // 1. buffer 至少要有 discriminator 的长度
        if buf.len() < [85, 240, 182, 158, 76, 7, 18, 233].len() {
            return Err(AccountDiscriminatorNotFound.into());
        }
        // 2. 确保前 8 字节正好是这个类型的 discriminator
        if &[85, 240, 182, 158, 76, 7, 18, 233] != &buf[..8] {
            return Err(AccountDiscriminatorMismatch.into());
        }
        // 3. 如果校验通过，调用“不带校验”的反序列化逻辑
        Self::try_deserialize_unchecked(buf)
    }

    // 不带 discriminator 校验，直接把剩余字节交给 AnchorDeserialize
    fn try_deserialize_unchecked(buf: &mut &[u8]) -> anchor_lang::Result<Self> {
        AnchorDeserialize::deserialize(&buf[8..])
            .map_err(AccountDidNotDeserialize)
    }
}

```





https://learnblockchain.cn/article/7386

https://www.anchor-lang.com/docs/