## Account Model
1. Solana 有三种账户类型：a. Data accounts: System-Ownered EOA 账户，PDA（Program derived Address）EOA 衍生的地址。 b .智能合约System Program c. Native account: 硬编码的程序，例如System，Token，Stake，Vote
2. Solana采用账户模型，Address->AccountInfo的Key->Value数据存储方式，其中，每个账户最多存储10MB 的数据（可执行的代码Code或者 状态数据）。
3. 账户需要预存代币，和数据存储量成正比，预存的代币作为数据存储的抵押物，在账户销户的时候会进行退款。
4. 每个账户都拥有一个Owner,只有当前程序的Owner才能更新数据或者转账操作。
5. Programs 就是 Solana 智能合约，是无状态的可执行Code，全部状态数据存储在独立的数据账号。内部逻辑函数被称为Instructions,构建交易需要执行ProgramID 和 内部函数以及函数输入。智能合约作为Programs挂载在账户下，因此内部的数据块可以被具有更新权限的账户更新。
![image](https://hackmd.io/_uploads/H1g_AHjOR.png)
6. Native program硬编码到Solan源码，作为 Validator程序的一部分并且提供整个网络的部分核心功能，当在Solane网络开发时，至少需要和 两个NativeProgram（System Program, BPF Loader）交互.
7. Program accounts
a. System Program: 带有特殊权限的智能合约。所有新账户的Owner都是SystemProgram(只有System Program可以创建新的账户)，分配数据账户的数据存储空间以及转移数据账户的Owner（自定义账户的权限），执行EOA账户的balance transfer交易（Solana链上的转账也是通过System Program 合约执行的）.

b. 为了完成 EOA transfer操作，System Program 合约定义了 transfer endpoint，endpoint需要 from，to,lamports参数。检查from地址是否通过 is_signer签名交易，双边账户是否 is_writable 允许更新余额状态。完成校验后，作为EOA 账户的Owner，removes from余额，增加to余额。

c. 增加账户余额不需要限制条件。因此，from账户的Owner必须是System Program（涉及账户余额的消减），但是to地址的Owner可以是任意Program。

d. EOA 钱包地址（SystemAccount）的Owner是System Program,内部不存储任何可执行Code, Lamports的值就是钱包余额的大小,只有EOA地址可以发起交易并付费GasFee
![image](https://hackmd.io/_uploads/SkexVtid0.png)

### Account
1. 账户都是通过System program 创建（EOA的Owner就是System Program，其余智能合约Programs的Owner可以转移Owner）。创建账户一般需要两步：Create&Initialize。新创建的账户内部初始化一定的存储size 和 rent banlace. 
2. 在每轮epoch以及 处在当前交易时都会去计算当前的rentFee(如果当前余额小于RentFee的话，该账户会被destroyed，状态数据被销毁)
3. 创建账户一般有两种方式：create&initialize的Instructions组合 或者 CPI Instructions。
a. 组合，create 和 initialize 是独立的两个instruction，两者之间的数据不互通
![image](https://hackmd.io/_uploads/BkAT1EwFC.png)
b. 通过CPI（Cross-Program Invocations）完成一个instruction中一个program直接call另一个program的组合。因此，两者的数据互通，在进行初始化的时候，如果账户已经被初始化过的话，会因为执行了数据的修改导致整个交易失败回滚，因此不用单独执行初始化的校验工作。
```solana=
/// pseudo code
fn initialize(accounts) {
    accounts.system_program.create_account(accounts.payer, accounts.counter);
    let counter = deserialize(accounts.counter);
    counter.count = 0;
}
```

3. Solana账户地址采用 Ed25519加密计算方式下的32位公钥，公钥作为Key值，用于定位链上的数据状态。私钥用于签名交易。
![image](https://hackmd.io/_uploads/HJqGAJCOC.png)
![image](https://hackmd.io/_uploads/ryn4AridR.png)

### AccountInfo
1. AccountInfo 存储账户Key值对应的数据，包含Data，Executable,Lamports,Owner四部分数据：
a. Data: bytes数组，存储当前账户的状态数据（如果当前账户是Smart contract,存储的是之智能合约代码）
b. Executable: 表示当前账户时候是Program account(内部只存储可执行代码)
c. lamports: Solana链上的最小gas单位，表示当前账户的余额，其中（1sol = 1 billion lamports）
d. owner: 表示 当前账户的Program Owner（publicKey）
2. Solana 链上每一个账户都拥有一个特定的program Owner, 只有当前AccountInfo的ProgramOwner可以更新当前账户结构体中的Data和转账Lamports.
3. 其中，在账户中存储数据需要质押一定的SOL，质押数量和数据大小成正比。质押金额可以在关闭账户数据的时候全部退化
### ProgramAccount
1. 当部署新的Programs(smart contract)的时候，会创建一下三个账户
a. Program Account（Program ID 公钥）:主账户，存储 可执行代码数据的账户
b. Program Executable Data Account: 账户地址存储所有合约Code
c. Buffer Account: 临时的账户，当部署合约的时候，合约code会临时存储在Buffer账户，当逻辑处理完成后，合约code会被转移到  b 账户，Buffer Account 被关闭
![image](https://hackmd.io/_uploads/H1E58KiOR.png)

### Data Account
1. Solana 的Program Account仅存储当前合约的Code,不会存储当前合约执行过程中的状态，为了记录和更新合约状态，需要创建新的 数据账户:
![image](https://hackmd.io/_uploads/HytQ_KsuC.png)
2. 只有System Program可以创建新的账户，因此一个数据账户的创建包含下面两个步骤：调用SystemProgram创建一个新的账户，并将Owner转移到自定义的程序（合约）；由自定义的程序初始化当前新创建的DataAccount

### Program Derived Address
1. 根据ProgramID 和 custom-defined seeds 以及 bump seeds(255~0) 计算得到不在Ed25519曲线上的账户，因此不存在 公私密钥对。
![image](https://hackmd.io/_uploads/SJZdC10_0.png)
2. PDA该账户没有私钥控制，因此不存在任何外部账户能够发起PDA账户的签名交易，Solana只允许derived-from ProgramID进行Sign和管理该账户。
![image](https://hackmd.io/_uploads/Skiqh1Cd0.png)
3. PDA账户必须通过 cross program invocations 发起 instructions创建
4. 创建PDA账户需要三个参数：
a. Optional seeds: 用户自定义的  (e.g. string, number, other account addresses) seeds，seeds转成bytes执行下一步计算
b. Bump seeds: PDA 是落在 curve之外的曲线，因此遍历 Bump（255~0）的值，将Bump值添加进 OperationalSeeds,循环找出Fall out Curve 的公钥值
c. ProgramID: PAD是基于某个Program的衍生账户，只有Program ID 可以签名该账户交易
![image](https://hackmd.io/_uploads/ryIc1eRu0.png)
5. findProgramAddressSync 返回 遍历Bump中第一个有效的Bump值和生成的PDA地址
```js=
import { PublicKey } from "@solana/web3.js";

const programId = new PublicKey("11111111111111111111111111111111");
const string = "helloNiKulas";

const [PDA, bump] = PublicKey.findProgramAddressSync(
  [Buffer.from(string)],
  programId
);

console.log(`PDA: ${PDA}`);
console.log(`Bump: ${bump}`);
```
底层通过 createProgramAddressSync 函数执行遍历，并且返回第一个有效值。通过下面代码，可循环遍历出全部有效的Bump值和基于 Bump、Seeds、ProgramID计算得出的PDA
```js=
import { PublicKey } from "@solana/web3.js";
 
const programId = new PublicKey("11111111111111111111111111111111");
const string = "helloWorld";
 
// Loop through all bump seeds for demonstration
for (let bump = 255; bump >= 0; bump--) {
  try {
    const PDA = PublicKey.createProgramAddressSync(
      [Buffer.from(string), Buffer.from([bump])],
      programId,
    );
    console.log("bump " + bump + ": " + PDA);
  } catch (error) {
    console.log("bump " + bump + ": " + error);
  }
}
```
6. 遍历过程可能会找出多个有效的Bump Nonce值，第一个有效的Nonce值被称为 Canonical Bump,在生成和校验PDA的时候，建议使用Canonical Bump值生成的PDA

## Transactions
1. 交易执行具有 instructions order 和 atomicity, 输入的参数在在合约Programs中按照字节码顺序执行。
2. Sonala一笔交易中可能会调用不同的ProgramsID(合约)，每个instructions需要表明合约ProgramID以及调用函数输入的参数
3. Solana网络包大小为 1240，其中交易大小限制为 1232 bytes （1232 bytes 佛如transactions + 40 bytes for IPV6 + 8 bytes for fragment header）
4. 交易会读/写账户状态数据，Solana未来保证高吞吐量，会首先检查交易涉及的内存区是否存在交叉，如果全是读数据或者不存在交叉的话，Solana会并行全部交易。
5. 为了达到并行吞吐量的目标，Solana交易中的账户存在下面字段：
```solana=
#[repr(C)]
pub struct AccountInfo<'a> {
    pub key: &'a Pubkey,
    pub lamports: Rc<RefCell<&'a mut u64>>,
    pub data: Rc<RefCell<&'a mut [u8]>>,
    pub owner: &'a Pubkey,
    pub rent_epoch: Epoch,
    pub is_signer: bool,
    pub is_writable: bool,
    pub executable: bool,
}
```
a. key: ed25519的公钥账户地址
b. is_signer:标识是否使用key对应的私钥签名当前交易
c. is_writable: 标识当前交易的源数据是否会更新并存储在数据状态中
d. data: 和账户账户相关的调用数据

### Transfer Transactions example
1. 只拥有SOL的EOA（System Account）的Owner是System Program,转账的流程是调用System Program验证 sender账户的签名(is_signer)和账户余额，然后由SystemProgram修改Sender和Receiver的余额（is_writable）
![image](https://hackmd.io/_uploads/SJunS5sO0.png)

``` solana
import {
  LAMPORTS_PER_SOL,
  SystemProgram,
  Transaction,
  sendAndConfirmTransaction,
  Keypair,
} from "@solana/web3.js";

// Use Playground cluster connection
const connection = pg.connection;

// Use Playground wallet as sender, generate random keypair as receiver
const sender = pg.wallet.keypair;
const receiver = new Keypair();

// Check and log balance before transfer
const preBalance1 = await connection.getBalance(sender.publicKey);
const preBalance2 = await connection.getBalance(receiver.publicKey);

console.log("sender prebalance:", preBalance1 / LAMPORTS_PER_SOL);
console.log("receiver prebalance:", preBalance2 / LAMPORTS_PER_SOL);
console.log("\n");

// Define the amount to transfer
const transferAmount = 0.01; // 0.01 SOL

// Create a transfer instruction for transferring SOL from wallet_1 to wallet_2
const transferInstruction = SystemProgram.transfer({
  fromPubkey: sender.publicKey,
  toPubkey: receiver.publicKey,
  lamports: transferAmount * LAMPORTS_PER_SOL, // Convert transferAmount to lamports
});

// Add the transfer instruction to a new transaction
const transaction = new Transaction().add(transferInstruction);

// Send the transaction to the network
const transactionSignature = await sendAndConfirmTransaction(
  connection,
  transaction,
  [sender] // signer
);

// Check and log balance after transfer
const postBalance1 = await connection.getBalance(sender.publicKey);
const postBalance2 = await connection.getBalance(receiver.publicKey);

console.log("sender postbalance:", postBalance1 / LAMPORTS_PER_SOL);
console.log("receiver postbalance:", postBalance2 / LAMPORTS_PER_SOL);
console.log("\n");

console.log(
  "Transaction Signature:",
  `https://explorer.solana.com/tx/${transactionSignature}?cluster=devnet`
);

```
2. Transactions中Sender必须作为Signer,签名有效，通过校验的话才会继续执行Instructions

### Transactions structure
![image](https://hackmd.io/_uploads/r1ibi5jOA.png)
1. Solana交易由两部分组成：签名数组 + Instructions Message
2. 签名数组： 交易可能涉及对不同ProgramsID的调用，不同的写调用需要不同的签名数据，因此交易开头需要一个签名array数组
3. 签名后跟随的是全部Instruactions Message(表明ProgramsID、Sender、Receiver、Data、Value)
![image](https://hackmd.io/_uploads/S1S9O5odR.png)
a. header: 表明签名的数量（ProgramID的数量）和 账户的操作权限（读/写） 以及 需要和不需要签名的的账户数量
![image](https://hackmd.io/_uploads/SJ3Ps9o_0.png)
b. Account address: Instauctions中加载的账户数组(一个交易最多32个地址)，其中 Compact 的压缩模式由两部分组成（the length of th earray + items listend by sequentially）
![image](https://hackmd.io/_uploads/rk4BrjsO0.png)
c. Recent Blockhash: blockhash指的是当前slot最新的POH hash(Proof of History),当交易创建的时候会包含这个slot poh hash. Solana通过POH作为世界时钟，因此POH hash可以作为链上timestamp时间戳，用来标识交易的有效期。
- POH hash计算公式：next_hash = hash(prev_hash, hash(transaction_ids))
- POH hash是时序递归计算的，后续的hash值取决于之前的hash值（Solana链上每个区块都有有一个BlockHash 和 ticks列表(hash of checkPoints)） 
- Solana 全局维护 BlockhashQueue 列表，大小 300，每当产生最新的区块，就会进表尾，挤出表头数据。
- 在Validator打包交易时，根据全局BlockhashQueue列表判断当前交易的blockhash是否是在最新的 150 区块中，（60~90s -> 400~600ms/slot），150区块之前的交易因为过期不会被处理。也就是交易需要在一定时间内执行，否则就会过期。
```sonala
const { blockhash, lastValidBlockHeight } =
  await pg.connection.getLatestBlockhash();

console.log("Blockhash:", blockhash);
console.log("Last Valid Block Height:", lastValidBlockHeight);
```
d. Instructions array:包含交易执行过程中全部的Instauctions数组, 调用ProgramsID的顺序执行的Instructions数组(必须标识ProgramID、调用的AccountAddress的Index数组、调用函数输入的data参数)
![image](https://hackmd.io/_uploads/HJ1KaijOC.png)
![image](https://hackmd.io/_uploads/rk0AisoOA.png)
4. Transactions Lifecycle
a. 构造account 和instructions数组
b. 获取当前最新的区块hash，基于这三个数据构建交易Message
c. 模拟执行该交易，评估交易执行结果是否符合预期
d. Sender使用私钥签名该交易
e. 交易通过RPC发送到Block producer(validators)
f. 等待矿工校验、装包该交易
g. 交易进块确认或者交易blockHash过期废弃

交易示例
``` json
"transaction": {
    "message": {
      "header": {
        "numReadonlySignedAccounts": 0,
        "numReadonlyUnsignedAccounts": 1,
        "numRequiredSignatures": 1
      },
      "accountKeys": [
        "3z9vL1zjN6qyAFHhHQdWYRTFAcy69pJydkZmSFBKHg1R",
        "5snoUseZG8s8CDFHrXY2ZHaCrJYsW457piktDmhyb5Jd",
        "11111111111111111111111111111111"
      ],
      "recentBlockhash": "DzfXchZJoLMG3cNftcf2sw7qatkkuwQf4xH15N5wkKAb",
      "instructions": [
        {
          "accounts": [
            0,
            1
          ],
          "data": "3Bxs4NN8M2Yn4TLb",
          "programIdIndex": 2,
          "stackHeight": null
        }
      ],
      "indexToProgramIds": {}
    },
    "signatures": [
      "5LrcE2f6uvydKRquEJ8xp19heGxSvqsVbcqUeFoiWbXe8JNip7ftPQNTAVPyTK7ijVdpkzmKKaAQR7MWMmujAhXD"
    ]
  }
```

### Transactions instructions
1. 构造交易的Instruction包含Account address列表，其中涉及的每个Account都必须包含下面三个值：
a. PublicKey: Key值，在链上唯一标识当前Account
b. is_signer:标识是否把当前账户作为transaction的signer(赋予签名权限)
c. is_writable:状态是否允许更改（只读或可更新）
![image](https://hackmd.io/_uploads/rJOdJ2sOC.png)
构建转账交易的Instruaction的交易
```solana=
// Define the amount to transfer
const transferAmount = 0.01; // 0.01 SOL
 
// Instruction index for the SystemProgram transfer instruction
const transferInstructionIndex = 2;
 
// Create a buffer for the data to be passed to the transfer instruction
const instructionData = Buffer.alloc(4 + 8); // uint32 + uint64
// Write the instruction index to the buffer
instructionData.writeUInt32LE(transferInstructionIndex, 0);
// Write the transfer amount to the buffer
instructionData.writeBigUInt64LE(BigInt(transferAmount * LAMPORTS_PER_SOL), 4);
 
// Manually create a transfer instruction for transferring SOL from sender to receiver
const transferInstruction = new TransactionInstruction({
  keys: [
    { pubkey: sender.publicKey, isSigner: true, isWritable: true },
    { pubkey: receiver.publicKey, isSigner: false, isWritable: true },
  ],
  programId: SystemProgram.programId,
  data: instructionData,
});
 
// Add the transfer instruction to a new transaction
const transaction = new Transaction().add(transferInstruction);
```

## Transactions Fee
Solana 交易手续费由三部分组成：TransactionsFee + PriorityFee + RentFee

### transactions fee
1. transaction fee 是网络Validator执行交易的固有手续费
2. 固定的 5k lamports/ signature + 动态的在执行交易过程中消耗的计算资源
3. Burn 50% fee, 剩余的50% 发放到处理交易的validators
4. 每笔交易至少有一个signer,writable的账户，默认account addressder的第一个地址作为 fee payer
5.每笔交易中涉及多个Instructions,整个交易的固有最大 gasLimit 是 1.4M，每个Instructions的最大gasLimit是 200 k。Sender发送交易时可以自己设定GasLimit（SetComputeUnitLimit）,但是不能超过最大值。如果不额外设定GasLimit的话，整笔交易会采用默认的gasLimit。

### Priority Fees
1. 矿工小费，用户加速交易的打包执行
2. 通过 set_compute_unit_price 设置 gasPrice, 此时的 gasLimit * GasPrice就是矿工小费。 如果不设置 gasPrice的话，就意味着不额外付出矿工小费，拥有最低的Priority Fee,对于矿工只能赚取交易执行后的固有gasFee。
3. 可以通过 getRecentPrioritizationFees RPC 获取当前网络中的最小小费值

### Rent Fees
1.Solana链上数据存储需要账户质押特定的余额，账户为存储数据需要质押一定金额的SOL（Owner可以关闭账户并且Claim回质押的余额，此后，该账户的余额为0，相关的状态数据也会被回收）
2. 创建新的Solana账户的时候，必须Deposit最低的Rent费用。每次Transfer SOL的时候，Runtime会检测是否该账户的剩余余额可以覆盖数据的Rent费用，如果减到0的话，该账户会被垃圾回收，数据也会被清空。
3. 账户当前的最低RentFee取决于当前网络的RentRate和当前账户存储数据的大小，可以通过 getMinimumBalanceForRentExemption RPC 获取特定数据大小下的RentFee

## Cross Program Invocation(CPI)
1. 区块链合约可以互相调用，在Solana transaction instructions 中可以调用另一个Program的函数
![image](https://hackmd.io/_uploads/By6TkVAO0.png)
2. 一个Instruction中的Invocation最深4层（Sonale交易栈深最大5层，除去第一次的函数invocation，允许外币调用的层数只有4层）
3. 采用 call的方式进行Invocation，每层调用的Sender都是Signer
4. 当Invoke 外部的Program（CPI）时，Program可以为自己衍生出的PDA账户签名
5. CPI 表示在一个Instruction中对于另一个Program的调用，因此CPI交易和Transactiosn交易一样需要指明下面三个参数：
a. Invoke的外币Program 地址
b. Accounts ayyay:Instauctions中涉及的账户数组
c. Instruction Data: 表明调用的程序函数和函数输入
6. Solana程序中，通过 invoke crate构建没有PDA的CPI交易，通过 invoke_signed构建存在PDA签名的CPI交易

## Tokens
1. The token Extensions Program含有全部ERC20/nft的函数Instructions
2. A mint account 存储代币合约的全部状态（总量、Decimal、Mint/Freeze权限）
3. Token Account 内部挂在token数据结构，能够接收和转账Token. 通过随机生成的Curve点创建的账户，然后阿静Owner转移的SystemProgram,之后初始化账户的Token结构体数据
4. Associated Token account: PDA 账户，有Sender和Token address共同计算得出。创建账户需要Invoke SystemProgram计算并创建PDA账户，初始化账户的Token数据结构。PDA账户下没有私钥，由基底的Sender发起交易签名。
![image](https://hackmd.io/_uploads/By74pBAOA.png)

## Validators
1. validators runs Solana Program 记录Solana链上的全部clusters账户信息并且验证网络transactions.
2. Validators分为 RPC node 和 voting/consensus node, RPC node用于和区块链交互，用户发送和广播交易，但是不参与Voting共识。 Voting validators 产生和投票区块
3. Solana 采用POS共识，用户可以给自己信任的validators质押SOL,质押成功后，会返回一定数量的Token奖励。已经质押的用户可以随时解质押。Validators基于质押的份额进行Voting，选举出来出块的Validators被称为Leader.
4. Solana 交易采用POH计算，让并行无序的网络交易能够更快的Finalization. 
5. 