# Substrate 设计总览 （二）

本文是《Substrate 设计总览》的第二部分，上一篇文章已经介绍了Substrate作为一个区块链框架提出的Runtime与Substrate Core的概念，本文将会简要介绍Substrate的项目结构。

**目前Substrate的文档十分缺乏，本文的介绍相当于是Substrate的一种文档。** 

## Substrate的项目结构

截至2019年2月初，Substrate的修改速度已经减缓，开始添加一些测试工具。可以看作1.0beta的Substrate走向稳定了。本文尝试以于master分支上的提交`bfd040ef0e5272cdb0dc56106786861cdbe8a397`进行项目结构的分析。

### 根级目录

* `ci` github 运行ci的脚本
* `core` **Substrate Core目录**，涵盖了上一篇文章描述的“链的系统基础部分”的功能，是该框架提供的核心功能
* `node` Substrate项目框架中自带的一个使用实例，**启动一条新链除Runtime部分之外可以直接把这里的代码拷贝过去**，同时也是熟悉框架的案例，调试框架的入口点。
* `node-template`一个更精简的`node`，似乎用来测试，作用不大，参考`node`更好
* `scripts`一些项目构建脚本，一般情况下无需关心
* `srml` Substrate**默认提供的一些Runtime的模块**，其中在启动自己的链时**有一些Runtime模块是必须的，有一些模块可以不用**。但是要注意**模块之间是有依赖关系的**，在是否采用Substrate提供的模块是必须深入考虑，这是一个比较重要的坑，要注意。ps：`srml`是 `Substrate Runtime Module library` 的缩写。构建自己的链时命名可以参考这个缩写。
* `subkey`生成公私钥的小工具
* `test-utils` 测试工具集合

在根目录下最重要的目录就是`core`与`srml`这两个目录，分别代表了Substrate框架提供的“区块链基础设置”与“Substrate Runtime”两个模块。`node`目录下的是运行框架的示例，也是框架的入口。

注意在根目录下的`Cargo.toml`中：

```toml
[[bin]]
name = "substrate"  # node 将会编译成 substrate 可以执行文件，位于 target/(debug|release)/目录下
path = "node/src/main.rs"  # bin 指向的是node目录

[package]
name = "substrate"
version = "0.10.0"
authors = ["Parity Technologies <admin@parity.io>"]
build = "build.rs"
# ...
```

所以在Substrate目录下直接执行`cargo run` 运行起来的程序是`node`节点，调试跟踪的时候注意这一点。

接下来详细介绍 `core`与`srml`目录

### `core`目录

core目录挑一些重要的进行介绍。注意如果在熟悉依赖的情况下，实际上可以在自己的项目中对引用不被依赖束缚的的模块进行扩充。

* 共识相关组件
  * `basic-authorship`  提供进入共识模块的提议者`Proposer`的构建，可以理解为这个节点的出块者，将会作为参数传入共识组件中。
  * `consensus` Substrate提出的新共识的核心`aura`，事实上这个共识模块能够进行扩充，其首先定义了一些共识阶段所需要的基础接口，然后使用`aura`算法实现了这里的共识接口。**目前这部分比较耦合**，若想采用自己设计的共识可能需要考虑直接fork Substrate项目在这个目录下进行更改。
  * `finality-grandpa` 与`aura`配对的区块一致性认证，与`aura`联系紧密。若不采用`aura`算法则这部分可以舍弃。
* `cli` 命令行输入的解析。若需要添加自己的命令参考`node`里的`cli`进行构架，不需要更改这部分。
* `client` client 是节点的**存储**与当前运行**节点区块与状态**。可以理解为是一个节点启动后代表当前节点的实例。client实例将会传入到交易池，网络连接等组件中，整个程序运行周期中只有一个client。（ps：client的这个抽象集合与以太坊的命名与功能类似）
* `executor`执行器，包含了wasm的执行构建功能。
* `inherents` 内部交易的一些工具定义，暂时不在本文中详细解释
* `keystore`与`keyring` 前者是提供公私钥工具，后者提供一些默认的私钥
* `network` 与 `network-libp2p` 底层网络设置，运用了著名的libp2p。若不涉及底层网络管理的事情无需关心这部分
* `primitives` 原语定义。substrate 很多定义基础数据结构的命名都以 primitives 定义。这里需要注意这个目录定义的是`substrate-primitives` 后续还会出现 `runtime-primitives`。这里的primitives代表的是整个substrate的原语定义。
* `rpc` 与 `rpc-servers` rpc 与 **websocket** 的功能，另外指出，substrate提供的rpc只含的很原始的rpc接口，无法对原始数据做一定包装。今后的文章会剖析如何利用rpc完成自己的功能。
* `service` 链一些定义的数据结构 与 各项服务启动的入口。所有的服务线程（交易池，p2p，共识...）都是在这个模块中启动。
* `transaction-pool` 交易池。
* `trie` modified merkle patricia trie，MPT，“世界状态”。substrate在这一块沿用了以太坊的设计，不过在substrate里面只有状态树，即使是合约的功能也不专门存在合约的存储树。
* `state-db`与`state-machine` 前者包含状态数据库，且有**对状态进行裁剪的功能**，后者是“世界状态”的变化修改，与以太坊的模式类似。
* Runtime相关的定义。在 Substrate的发展中一开始命名为`Runtime`，后续命名为`Substrate Runtime`，故缩写为 `sr`，在代码中沿用之前的写法会出现 把 `sr`前缀 重命名为 `runtime `的情况。
  * `sr-std`，`sr-primitives`与`sr-io` 定义了 兼容 std/wasm 的std 库 ，在 runtime中使用的原语，能够允许std/wasm范围相同存储的io库。这里的`sr-primitives` 就是 `runtime-primitives` 。`substrate-primitives`是  `runtime-primitives` 的子集。
  * `sr-sandbox` 运用于 wasm 执行器的沙盒
  * `sr-api-macros`与`sr-version` 前者是提供Runtime的一些宏，后者是wasm与native执行时的版本判定的一些相关组件

### `srml` 目录

`srml`是Substrate默认提供的 `runtime module`。部分模块不可缺少，其他的可以根据自己需求使用。

这里说不可缺少也并不是绝对的，如果自己的需求超出了当前这套以k-v为基石的模型，那么构建一套Runtime的基础设施也是可以的，如果没有这个需求和能力的话那就使用Substrate提供的Runtime的这些工具就好了。

本章同样只挑选重要的进行讲解。

首先列举核心的module

* `system`，`support` 与`metadata` 。这三个模块不可缺少的原因是它们负责了一些核心组件，并且部分被隐藏在`Runtime module` 定义一些结构的宏里。如果使用Substrate Runtime提供的工具，那么这些模块不可缺少。`system`用于和“区块”，“交易”相关，例如有一些Event定义，当前区块的hash，prev hash等等。`support`重点在于提供Runtime Module数据结构的宏的定义，提供了`decl_storage!`这个最关键的宏，这个模块很关键也很复杂，后续文章会进行讲解。`metadata`是用于导出Runtime Module的一些描述定义。
* `timestamp` 提供区块时间戳，runtime时间的模块。由于Substrate设计为Header里面不包含时间戳，而是使用一条交易包含时间，所以这个模块一般需要引入。
* `balance` 和`assets` 前者用于定义和资金相关的模块，后者定义类似token的相关模块。由于`balance`在很多地方都存在依赖关系，建议引入。
* `consensus`,`aura`和`grandpa` 共识相关模块，可以理解为可以反馈一些底层共识的信息与设置一些参数，提供内部交易等，不是共识算法的逻辑。
* `session`和 `staking` 定义区块的session间隔与权益相关的计算
* `council`，`democracy` ，`treasury`民主提议，财政等相关
* `sudo` sudo 权限，可以指定一个账户去执行 root 交易，后续文章会做介绍。
* `indices` 数字账户系统，相当于给公钥分配一个数字。
* `contract` 合约模块。这个模块与以太坊的智能合约功能近似。
* `example` 顾名思义，提供 Runtime Module 的编写示例。

总结`srml`：

如果考虑采用Substrate 提供的Runtime的工具集及它的模型，则需要引入一些必须的模块。若不使用，则需要构建许多Runtime Module 的基础工具集合且需要对整个框架有比较深入了解。而其他的模块可以根据自己链的需求进行选择。对于Runtime Module 的编写后续文章会进行介绍。

### `node `目录

`node`是Substrate的一个示例。这里要首先指出，对于一条链而言，首先需要编译出Runtime的wasm代码，然后需要把wasm的执行文件一同编译进入节点，成为genesis中的数据。

* `cli` 针对 node的一些命令行，及**genesis的定义**
* `executor` 执行器的定义，注意这里需要读取wasm的执行文件并引入
* `primitives` node 的原语，进行了许多基础类型的定义。
* `runtime` 一条链的Runtime，在这个Runtime中引入了这条链需要的Runtime Module。注意这个模块下有`src`与`wasm`两个目录，其中Runtime的代码由`src`编译而来，而`wasm`使用`src`的代码编译出了wasm的执行文件。编译wasm需要切换到 `wasm`的路径下，并执行这个路径下的`build.sh`脚本。编译后会在`wasm/target/wasm32-unknown-unknown/release` 目录下发现一个`node_runtime.compact.wasm`文件，这个文件就是Runtime的wasm执行文件。由于不同的机子编译出的wasm会在符号上有一些不一致，所以**注意这个文件需要进入git的管理** ，以防止由于需要引入wasm到genesis中导致生成的genesis不一致。

**********

以上为本文内容，具体介绍了Substrate的项目结构。

在对项目结构有一定了解后，就可以开始进行了解Substrate了。