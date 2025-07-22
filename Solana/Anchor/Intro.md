

[TOC]



# Native 与Anchor

当我们打开SolanaPlayGround时，选择native会有如下的代码案例自动生成：

```rust
use borsh::{BorshDeserialize, BorshSerialize};
use solana_program::{
    account_info::{next_account_info, AccountInfo},
    entrypoint,
    entrypoint::ProgramResult,
    msg,
    program_error::ProgramError,
    pubkey::Pubkey,
};

/// Define the type of state stored in accounts
#[derive(BorshSerialize, BorshDeserialize, Debug)]
pub struct GreetingAccount {
    /// number of greetings
    pub counter: u32,
}

// Declare and export the program's entrypoint
entrypoint!(process_instruction);

// Program entrypoint's implementation
pub fn process_instruction(
    program_id: &Pubkey, // Public key of the account the hello world program was loaded into
    accounts: &[AccountInfo], // The account to say hello to
    _instruction_data: &[u8], // Ignored, all helloworld instructions are hellos
) -> ProgramResult {
    msg!("Hello World Rust program entrypoint");

    // Iterating accounts is safer than indexing
    let accounts_iter = &mut accounts.iter();

    // Get the account to say hello to
    let account = next_account_info(accounts_iter)?;

    // The account must be owned by the program in order to modify its data
    if account.owner != program_id {
        msg!("Greeted account does not have the correct program id");
        return Err(ProgramError::IncorrectProgramId);
    }

    // Increment and store the number of times the account has been greeted
    let mut greeting_account = GreetingAccount::try_from_slice(&account.data.borrow())?;
    greeting_account.counter += 1;
    greeting_account.serialize(&mut *account.data.borrow_mut())?;

    msg!("Greeted {} time(s)!", greeting_account.counter);

    Ok(())
}
```

结合我们之前在交易中了解的知识，就明白`process_instruction`中的三个输入在instruction中非常重要。

```rust
program_id: &Pubkey, 
accounts: &[AccountInfo], 
_instruction_data: &[u8], 
```



但是我们创建一个默认的anchor项目时，会发现这三个元素似乎消失不见了

```rust
use anchor_lang::prelude::*;

// This is your program's public key and it will update
// automatically when you build the project.
declare_id!("11111111111111111111111111111111");

#[program]
mod hello_anchor {
    use super::*;
    pub fn initialize(ctx: Context<Initialize>, data: u64) -> Result<()> {
        ctx.accounts.new_account.data = data;
        msg!("Changed data to: {}!", data); // Message will show up in the tx logs
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    // We must specify the space in order to initialize an account.
    // First 8 bytes are default account discriminator,
    // next 8 bytes come from NewAccount.data being type u64.
    // (u64 = 64 bits unsigned integer = 8 bytes)
    #[account(init, payer = signer, space = 8 + 8)]
    pub new_account: Account<'info, NewAccount>,
    #[account(mut)]
    pub signer: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[account]
pub struct NewAccount {
    data: u64
}
```

但是实际上，anchor把这三个重要的输入拆分到了各个地方：

1. 原先的programId被放在了`declare_id!`中。

2. Account相关的array其实被隐式地放在了每个instruction的第一参数`Context<T>`中，而其中的T就是`#[derive(Accounts)]`下的结构体，它会明确各个需要的账户类型和限制，相比native使用`iter`更加简单和清晰

   > 使用iter就表示你得保证交易中输入instruction中的accounts和函数内的赋值顺序一致,并且需要在程序中写好检查.

3. `#[program]`会隐式地生成类似的`_instruction_data: &[u8]`输入口，同时本来被传入的字节序列`Instruction_data`，需要在native的instruction中parse并序列化。但是在anchor我们只需要使用IDL的方式调用

可以看出Anchor的宏是一个非常重要的内容，而且Anchor的IDL还在其中发挥了作用，下面我们分别聊聊Anchor的这两个部分。



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



# IDL

IDL(Interface Description Language)是一个在使用Anchor SDK客户端（[@coral-xyz/anchor](https://github.com/coral-xyz/anchor/tree/0e5285aecdf410fa0779b7cd09a47f235882c156/ts/packages/anchor)）交互时非常重要的文件，它被用来作为标准化的接口来创建`Program`实例，当我们使用`anchor build`时自动生成此文件。

其作用类似以太坊中的ABI，但区别是：

ABI是以太坊的原生客户端的通用接口文件；

IDL是这是Anchor独有的，而非Solana原生的；其作用是用于和Anchor的SDK进行自动化客户端代码生成，账户和指令序列化和反序列化而引入的元数据格式（简单来说`@coral-xyz/anchor`在`@solana/web3.js`上做了大量封装，从而需要一个接口文件作为输入值而已）。

如果使用native则不会产生或者需要此类文件，而是直接使用solana原生的SDK`@solana/web3.js`等自己进行操作。



## Anchor程序到IDL

以下示例程序中：

```rust
use anchor_lang::prelude::*;
 
declare_id!("BYFW1vhC1ohxwRbYoLbAWs86STa25i9sD5uEusVjTYNd");
 
#[program]
mod hello_anchor {
    use super::*;
    pub fn initialize(ctx: Context<Initialize>, data: u64) -> Result<()> {
        ctx.accounts.new_account.data = data;
        msg!("Changed data to: {}!", data);
        Ok(())
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

对应的IDL是（Anchor 0.31.1版本）：

```json
{
  "address": "BYFW1vhC1ohxwRbYoLbAWs86STa25i9sD5uEusVjTYNd",//programId
  "metadata": {
    "name": "hello_anchor",
    "version": "0.1.0",
    "spec": "0.1.0",
    "description": "Created with Anchor"
  },
  "instructions": [
    {
      "name": "initialize",
      "discriminator": [175, 175, 109, 31, 13, 152, 155, 237],
      "accounts": [
        {
          "name": "new_account",
          "writable": true,
          "signer": true
        },
        {
          "name": "signer",
          "writable": true,
          "signer": true
        },
        {
          "name": "system_program",
          "address": "11111111111111111111111111111111"
        }
      ],
      "args": [
        {
          "name": "data",
          "type": "u64"
        }
      ]
    }
  ],
  "accounts": [
    {
      "name": "NewAccount",
      "discriminator": [176, 95, 4, 118, 91, 177, 125, 232]
    }
  ],
  "types": [
    {
      "name": "NewAccount",
      "type": {
        "kind": "struct",
        "fields": [
          {
            "name": "data",
            "type": "u64"
          }
        ]
      }
    }
  ]
}
```

上面的IDL中，我们注意到有两个`accounts`字段，

- 第一个`accounts`是在instruction中的，就是我们对应native输入三要素的instruction所需要的各个`accountMeta`。

- 第二个`accounts`则对应的是在Anchor中定义的`#[account]`属性的struct，`#[account]`下的结构体就是我们创建的account所存储的信息

  - 其中[Discriminators](https://www.anchor-lang.com/docs/basics/idl#discriminators)则作为类似（函数）签名的hash的功能，以分辨instruction/account的。其算法是类型固定前缀+名称的hash的前8字节

    如：Instruction:`sha256("global:initialize")`的前八字节

    ​	Accounts: `sha256("account:NewAccount")`的前八字节

- 后面跟随的`types`则详细阐述了每个`#[account]`属性下的数据结构类型和结构下的各个字段。



# PDA

在之前的学习中，我们知道PDA账户是由ProgramId+Seed得到的，而`bump`则是保证生成的PDA账户是没有对应的私钥的。

在一个Native项目中，创建pda主要需要`invoke_signed`

一个涉及pda创建的案例如下 ([Ref](https://github.com/solana-developers/program-examples/tree/main/basics/pda-rent-payer)):

```rust
use borsh::{BorshDeserialize, BorshSerialize};
use solana_program::{
    account_info::{next_account_info, AccountInfo},
    entrypoint,
    entrypoint::ProgramResult,
    program::invoke_signed,
    pubkey::Pubkey,
    rent::Rent,
    system_instruction,
    sysvar::Sysvar,
};

//structs
#[derive(BorshDeserialize, BorshSerialize, Debug)]
pub struct RentVault {}
impl RentVault {
    pub const SEED_PREFIX: &'static str = "rent_vault";//@notice: 这里是第一个seed
}

#[derive(BorshSerialize, BorshDeserialize)]
pub enum MyInstruction {
    InitRentVault(InitRentVaultArgs),
    CreateNewAccount,
}

#[derive(BorshDeserialize, BorshSerialize)]
pub struct InitRentVaultArgs {
    fund_lamports: u64,
}

entrypoint!(process_instruction);

//functions
pub fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    input: &[u8],
) -> ProgramResult {
    let instruction = MyInstruction::try_from_slice(input)?;
    match instruction {
        MyInstruction::InitRentVault(args) => init_rent_vault(program_id, accounts, args),
        MyInstruction::CreateNewAccount => create_new_account(program_id, accounts),
    }
}
//这个函数会创建一个pda
pub fn init_rent_vault(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    args: InitRentVaultArgs,
) -> ProgramResult {
    let accounts_iter = &mut accounts.iter();
    let rent_vault = next_account_info(accounts_iter)?;
    let payer = next_account_info(accounts_iter)?;
    let system_program = next_account_info(accounts_iter)?;

    let (rent_vault_pda, rent_vault_bump) =
        Pubkey::find_program_address(&[RentVault::SEED_PREFIX.as_bytes()], program_id);
    assert!(rent_vault.key.eq(&rent_vault_pda));

    // Lamports for rent on the vault, plus the desired additional funding
    //
    let lamports_required = (Rent::get()?).minimum_balance(0) + args.fund_lamports;
	
    //使用invoke_signed,而非invoke来执行systemProgram的指令`create_account`
    //因为pda没有私钥,无法签名,所以这是一个pda专属的invoke方法
    invoke_signed(
        &system_instruction::create_account(
            payer.key,
            rent_vault.key,
            lamports_required,
            0,
            program_id,
        ),//instruction
        &[payer.clone(), rent_vault.clone(), system_program.clone()],//accounts array
        &[&[RentVault::SEED_PREFIX.as_bytes(), &[rent_vault_bump]]],//pda seeds + bump
    )?;

    Ok(())
}
//这个函数通过seed + programId来找到pda地址和bump
pub fn create_new_account(program_id: &Pubkey, accounts: &[AccountInfo]) -> ProgramResult {
    let accounts_iter = &mut accounts.iter();
    let new_account = next_account_info(accounts_iter)?;
    let rent_vault = next_account_info(accounts_iter)?;
    let _system_program = next_account_info(accounts_iter)?;

    let (rent_vault_pda, 
        _rent_vault_bump) =
    Pubkey::find_program_address(&[RentVault::SEED_PREFIX.as_bytes()], program_id);
    
    assert!(rent_vault.key.eq(&rent_vault_pda));//保证我们拿到的地址是pda地址.

    
    // Assuming this account has no inner data (size 0)
    let lamports_required_for_rent = (Rent::get()?).minimum_balance(0);
	
    //转账自动调用SystemProgram::CreateAccount创建账户
    **rent_vault.lamports.borrow_mut() -= lamports_required_for_rent;
    **new_account.lamports.borrow_mut() += lamports_required_for_rent;
    
//The key here is accounts on Solana are automatically created under ownership of the System Program when you transfer lamports to them. So, you can just transfer lamports from your PDA to the new account's public key!
    Ok(())
    //这种创建账户的方法会创建systemAccount属性的账户，意思是该账户的owner是systemProgram
}


```

而在Anchor下，创建pda只需要在`#[derive(Accounts)]`的结构体下的accont属性中加入种子和bump，下面是一个创建pda的anchor案例（[see more](https://www.anchor-lang.com/docs/basics/pda#anchor-pda-constraints)）

```rust
#![allow(clippy::result_large_err)]
use anchor_lang::prelude::*;
use anchor_lang::system_program::{create_account, transfer, CreateAccount, Transfer};

declare_id!("AvKoqYU7yhpddJriNSVcg2AMnDfpkWTWGU8LsRKXXrtE");

#[program]
pub mod pda_rent_payer {
    use super::*;

    pub fn init_rent_vault(ctx: Context<InitRentVault>, fund_lamports: u64) -> Result<()> {
        instructions::_init_rent_vault(ctx, fund_lamports)
    }

    pub fn create_new_account(ctx: Context<CreateNewAccount>) -> Result<()> {
        instructions::_create_new_account(ctx)
    }
    
    pub mod instructions {
        use super::*;
        // When lamports are transferred to a new address (without and existing account),
        // An account owned by the system program is created by default
        pub fn _init_rent_vault(ctx: Context<InitRentVault>, fund_lamports: u64) -> Result<()> {
            transfer(
                CpiContext::new(
                    ctx.accounts.system_program.to_account_info(),
                    Transfer {
                        from: ctx.accounts.payer.to_account_info(),
                        to: ctx.accounts.rent_vault.to_account_info(),
                    },
                ),
                fund_lamports,
            )?;
            Ok(())
        }

        pub fn _create_new_account(ctx: Context<CreateNewAccount>) -> Result<()> {
            // PDA signer seeds
            let signer_seeds: &[&[&[u8]]] = &[&[b"rent_vault", &[ctx.bumps.rent_vault]]];

            // The minimum lamports for rent exemption
            let lamports = (Rent::get()?).minimum_balance(0);

            // Create the new account, transferring lamports from the rent vault to the new account
            create_account(
                CpiContext::new(
                    ctx.accounts.system_program.to_account_info(),
                    CreateAccount {
                        from: ctx.accounts.rent_vault.to_account_info(), // From pubkey
                        to: ctx.accounts.new_account.to_account_info(),  // To pubkey
                    },
                ).with_signer(signer_seeds),
                lamports,                           // Lamports
                0,                                  // Space
                &ctx.accounts.system_program.key(), // Owner Program
            )?;
            Ok(())
        }

    }

}

#[derive(Accounts)]
pub struct CreateNewAccount<'info> {
    #[account(mut)]
    new_account: Signer<'info>,
    #[account(
        mut,
        seeds = [
            b"rent_vault",
        ],
        bump,
    )]//这里就是pda创建所需要的seeds&bump，
    //如果加入init&payer&space字段
    //则会像anchor默认案例一样自动创建对应的账户
    
    rent_vault: SystemAccount<'info>,
    system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct InitRentVault<'info> {
    #[account(mut)]
    payer: Signer<'info>,
    #[account(
        mut,
        seeds = [
            b"rent_vault",
        ],
        bump,
    )]
    rent_vault: SystemAccount<'info>,
    system_program: Program<'info, System>,
}

```

# CPI

Cross Program Invocations (CPI)其实就是通过程序互相调用来完成指令的过程（the process of one program invoking instructions of another program）。





# Reference

https://learnblockchain.cn/article/7386

https://www.anchor-lang.com/docs/

https://solana.com/developers/courses