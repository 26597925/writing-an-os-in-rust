>原文：https://os.phil-opp.com/freestanding-rust-binary/
>
>原作者：@phil-opp
>
>译者：洛佳  华中科技大学

# 使用Rust编写操作系统（一）：独立式可执行程序

创建一个不连接标准库的Rust可执行文件，将是我们迈出的第一步。无需底层操作系统的支撑，这将能让在**裸机**（[bare metal](https://en.wikipedia.org/wiki/Bare_machine)）上运行Rust代码成为现实。

## 简介

要编写一个操作系统内核，我们需要不基于任何操作系统特性的代码。这意味着我们不能使用线程、文件、堆内存、网络、随机数、标准输出，或其它任何需要操作系统抽象和特定硬件的特性；这其实讲得通，因为我们正在编写自己的操作系统和硬件驱动。

实现这一点，意味着我们不能使用[Rust标准库](https://doc.rust-lang.org/std/)的大部分；但还有很多Rust特性是我们依然可以使用的。比如说，我们可以使用[迭代器](https://doc.rust-lang.org/book/ch13-02-iterators.html)、[闭包](https://doc.rust-lang.org/book/ch13-01-closures.html)、[模式匹配](https://doc.rust-lang.org/book/ch06-00-enums.html)、[Option](https://doc.rust-lang.org/core/option/)、[Result](https://doc.rust-lang.org/core/result/index.html)、[字符串格式化](https://doc.rust-lang.org/core/macro.write.html)，当然还有[所有权系统](https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html)。这些功能让我们能够编写表达性强、高层抽象的操作系统，而无需操心[未定义行为](https://www.nayuki.io/page/undefined-behavior-in-c-and-cplusplus-programs)和[内存安全](https://tonyarcieri.com/it-s-time-for-a-memory-safety-intervention)。

为了用Rust编写一个操作系统内核，我们需要创建一个独立于操作系统的可执行程序。这样的可执行程序常被称作**独立式可执行程序**（freestanding executable）或**裸机程序**(bare-metal executable)。

在这篇文章里，我们将逐步地创建一个独立式可执行程序，并且详细解释为什么每个步骤都是必须的。如果读者只对最终的代码感兴趣，可以跳转到本篇文章的小结部分。

## 安装 Nightly Rust

Rust语言有三个**发行频道**（release channel），分别是stable、beta和nightly。《Rust程序设计语言》中对这三个频道的区别解释得很详细，可以前往[这里](https://doc.rust-lang.org/book/appendix-07-nightly-rust.html)看一看。为了搭建一个操作系统，我们需要一些只有nightly会提供的实验性功能，所以我们需要安装一个nightly版本的Rust。

要管理安装好的Rust，我强烈建议使用[rustup](https://www.rustup.rs/)：它允许你同时安装nightly、beta和stable版本的编译器，而且让更新Rust变得容易。你可以输入`rustup override add nightly`来选择在当前目录使用nightly版本的Rust。或者，你也可以在项目根目录添加一个名称为`rust-toolchain`、内容为`nightly`的文件。要检查你是否已经安装了一个nightly，你可以运行`rustc --version`：返回的版本号末尾应该包含`-nightly`。

Nightly版本的编译器允许我们在源码的开头插入**特性标签**（feature flag），来自由选择并使用大量实验性的功能。举个栗子，要使用实验性的[内联汇编（asm!宏）](https://doc.rust-lang.org/nightly/unstable-book/language-features/asm.html)，我们可以在`main.rs`的顶部添加`#![feature(asm)]`。要注意的是，这样的实验性功能**不稳定**（unstable），意味着未来的Rust版本可能会修改或移除这些功能，而不会有预先的警告过渡。因此我们只有在绝对必要的时候，才应该使用这些特性。

## 禁用标准库

在默认情况下，所有的Rust**包**（crate）都会链接**标准库**（[standard library](https://doc.rust-lang.org/std/)），而标准库依赖于操作系统功能，如线程、文件系统、网络。标准库还与**Rust的C语言标准库实现库**（libc）相关联，它也是和操作系统紧密交互的。既然我们的计划是编写自己的操作系统，我们就可以不使用任何与操作系统相关的库——因此我们必须禁用**标准库自动引用**（automatic inclusion）。使用[`no_std`属性](https://doc.rust-lang.org/book/first-edition/using-rust-without-the-standard-library.html)可以实现这一点。

我们可以从创建一个新的cargo项目开始。最简单的办法是使用下面的命令：

```bash
> cargo new blog_os
```

在这里我把项目命名为`blog_os`，当然读者也可以选择自己的项目名称。这里，cargo默认为我们添加了`--bin`选项，说明我们将要创建一个可执行文件（而不是一个库）；cargo还为我们添加了`--edition 2018`标签，指明项目的包要使用Rust的**2018版次**（[2018 edition](https://rust-lang-nursery.github.io/edition-guide/rust-2018/index.html)）。当我们执行这行指令的时候，cargo为我们创建的目录结构如下：

```
blog_os
├── Cargo.toml
└── src
    └── main.rs
```

在这里，`Cargo.toml`文件包含了包的**配置**（configuration），比如包的名称、作者、[semver版本](http://semver.org/)和项目依赖项；`src/main.rs`文件包含包的**根模块**（root module）和main函数。我们可以使用`cargo build`来编译这个包，然后在`target/debug`文件夹内找到编译好的`blog_os`二进制文件。

### no_std属性

现在我们的包依然隐式地与标准库链接。为了禁用这种链接，我们可以尝试添加[`no_std`属性](https://doc.rust-lang.org/book/first-edition/using-rust-without-the-standard-library.html)：

```rust
// main.rs

#![no_std]

fn main() {
    println!("Hello, world!");
}
```

看起来非常顺利。当我们使用`cargo build`来编译的时候，却出现了下面的错误：

```rust
error: cannot find macro `println!` in this scope
 --> src\main.rs:4:5
  |
4 |     println!("Hello, world!");
  |     ^^^^^^^
```

出现这个错误的原因是，[`println!`宏](https://doc.rust-lang.org/std/macro.println.html)是标准库的一部分，而我们的项目不再依赖于标准库。我们选择不再打印字符串。这也能解释得通，因为`println!`将会向**标准输出**（[standard output](https://en.wikipedia.org/wiki/Standard_streams#Standard_output_.28stdout.29)）打印字符，它依赖于特殊的文件描述符，而这是由操作系统提供的特性。

所以我们可以移除这行代码，使用一个空的main函数再次尝试编译：

```rust
// main.rs

#![no_std]

fn main() {}
```

```
> cargo build
error: `#[panic_handler]` function required, but not found
error: language item required, but not found: `eh_personality`
```

现在我们发现，编译器缺少一个`#[panic_handler]`函数和一个**语言项**（language item）。

## 实现panic处理函数

`panic_handler`属性定义了一个函数，它会在一个panic发生时被调用。标准库中提供了自己的panic处理函数，但在`no_std`环境中，我们需要定义一个自己的panic处理函数：

```rust
// in main.rs

use core::panic::PanicInfo;

/// 这个函数将在panic时被调用
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

类型为[`PanicInfo`](https://doc.rust-lang.org/nightly/core/panic/struct.PanicInfo.html)的参数包含了panic发生的文件名、代码行数和可选的错误信息。这个函数从不返回，所以他被标记为**发散函数**（[diverging function](https://doc.rust-lang.org/book/first-edition/functions.html#diverging-functions)）。发散函数的返回类型称作**Never类型**（["never" type](https://doc.rust-lang.org/nightly/std/primitive.never.html)），记为`!`。对这个函数，我们目前能做的事情很少，所以我们只需编写一个无限循环`loop {}`。

## eh_personality语言项

语言项是一些编译器需求的特殊函数或类型。举例来说，Rust的[`Copy`](https://doc.rust-lang.org/nightly/core/marker/trait.Copy.html) trait是一个这样的语言项，告诉编译器哪些类型需要遵循**复制语义**（[copy semantics](https://doc.rust-lang.org/nightly/core/marker/trait.Copy.html)）——当我们查找`Copy` trait的[实现](https://github.com/rust-lang/rust/blob/485397e49a02a3b7ff77c17e4a3f16c653925cb3/src/libcore/marker.rs#L296-L299)时，我们会发现，一个特殊的`#[lang = "copy"]`属性将它定义为了一个语言项，达到与编译器联系的目的。

我们可以自己实现语言项，但这只应该是最后的手段：目前来看，语言项是高度不稳定的语言细节实现，它们不会经过编译期类型检查（所以编译器甚至不确保它们的参数类型是否正确）。幸运的是，我们有更稳定的方式，来修复上面的语言项错误。

`eh_personality`语言项标记的函数，将被用于实现**栈展开**（[stack unwinding](http://www.bogotobogo.com/cplusplus/stackunwinding.php)）。在使用标准库的情况下，当panic发生时，Rust将使用栈展开，来运行在栈上活跃的所有变量的**析构函数**（destructor）——这确保了所有使用的内存都被释放，允许调用程序的**父进程**（parent thread）捕获panic，处理并继续运行。但是，栈展开是一个复杂的过程，如Linux的[libunwind](http://www.nongnu.org/libunwind/)或Windows的**结构化异常处理**（[structured exception handling, SEH](https://msdn.microsoft.com/en-us/library/windows/desktop/ms680657(v=vs.85).aspx)），通常需要依赖于操作系统的库；所以我们不在自己编写的操作系统中使用它。

### 禁用栈展开

在其它一些情况下，栈展开不是迫切需求的功能；因此，Rust提供了**在panic时中止**（[abort on panic](https://github.com/rust-lang/rust/pull/32900)）的选项。这个选项能禁用栈展开相关的标志信息生成，也因此能缩小生成的二进制程序的长度。有许多方式能打开这个选项，最简单的方式是把下面的几行设置代码加入我们的`Cargo.toml`：

```toml
[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

这些选项能将**`dev`配置**（dev profile）和**`release`配置**（release profile）的panic策略设为`abort`。`dev`配置适用于`cargo build`，而`release`配置适用于`cargo build --release`。现在编译器应该不再要求我们提供`eh_personality`语言项实现。

现在我们已经修复了出现的两个错误，可以信心满满地开始编译了。然而，尝试编译运行后，一个新的错误出现了：

```bash
> cargo build
error: requires `start` lang_item
```

## start语言项

这里，我们的程序遗失了`start`语言项，它将定义一个程序的**入口点**（entry point）。

我们通常会认为，当运行一个程序时，首先被调用的是`main`函数。但是，大多数语言都拥有一个**运行时系统**（[runtime system](https://en.wikipedia.org/wiki/Runtime_system)），它通常为**垃圾回收**（garbage collection）或**绿色线程**（software threads，或green threads）服务，如Java的GC或Go语言的协程（goroutine）；这个运行时系统需要在main函数前启动，因为它需要让程序初始化。

在一个典型的使用标准库的Rust程序中，程序运行是从一个名为`crt0`的运行时库开始的。`crt0`意为C runtime zero，它能建立一个适合运行C语言程序的环境，这包含了栈的创建和可执行程序参数的传入。这之后，这个运行时库会调用[Rust的运行时入口点](https://github.com/rust-lang/rust/blob/bb4d1491466d8239a7a5fd68bd605e3276e97afb/src/libstd/rt.rs#L32-L73)，这个入口点被称作**start语言项**（"start" language item）。Rust只拥有一个极小的运行时，它被设计为拥有较少的功能，如爆栈检测和打印**堆栈轨迹**（stack trace）。这之后，这个运行时将会调用main函数。

我们的独立式可执行程序并不能访问Rust运行时或`crt0`库，所以我们需要定义自己的入口点。实现一个`start`语言项并不能帮助我们，因为这之后程序依然要求`crt0`库。所以，我们要做的是，直接重写整个`crt0`库和它定义的入口点。

### 重写入口点

要告诉Rust编译器我们不使用预定义的入口点，我们可以添加`#![no_main]`属性。

```rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;

/// 这个函数将在panic时被调用
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

读者也许会注意到，我们移除了`main`函数。原因很显然，既然没有底层已有的运行时调用它，`main`函数也失去了存在的必要性。为了重写操作系统的入口点，我们转而编写一个`_start`函数：

```rust
#[no_mangle]
pub extern "C" fn _start() -> ! {
    loop {}
}
```

我们使用`no_mangle`标记这个函数，来对它禁用**名称重整**（[name mangling](https://en.wikipedia.org/wiki/Name_mangling)）——这确保Rust编译器输出一个名为`_start`的函数；否则，编译器可能最终生成名为`_ZN3blog_os4_start7hb173fedf945531caE`的函数，无法让链接器正确辨别。

我们还将函数标记为`extern "C"`，告诉编译器这个函数应当使用[C语言的调用约定](https://en.wikipedia.org/wiki/Calling_convention)，而不是Rust语言的调用约定。函数名为`_start`，是因为大多数系统默认使用这个名字作为入口点名称。

与前文的`panic`函数类似，这个函数的返回值类型为`!`——它定义了一个发散函数，或者说一个不允许返回的函数。这一点是必要的，因为这个入口点不将被任何函数调用，但将直接被操作系统或**引导程序**（bootloader）调用。所以作为函数返回的替换，这个入口点应该调用，比如操作系统提供的**exit系统调用**（["exit" system call](https://en.wikipedia.org/wiki/Exit_(system_call))）函数。在我们编写操作系统的情况下，关机应该是一个合适的选择，因为**当一个独立式可执行程序返回时，不会留下任何需要做的事情**（there is nothing to do if a freestanding binary returns）。现在来看，我们可以添加一个无限循环，来满足对返回值类型的需求。

如果我们现在编译这段程序，会出来一大段不太好看的**链接器错误**（linker error）。

## 链接器错误

**链接器**（linker）是一个程序，它将生成的目标文件组合为一个可执行文件。不同的操作系统如Windows、macOS、Linux，规定了不同的可执行文件格式，因此也各有自己的链接器，抛出不同的错误；但这些错误的根本原因还是相同的：链接器的默认配置假定程序依赖于C语言的运行时环境，但我们的程序并不依赖于它。

为了解决这个错误，我们需要告诉链接器，它不应该包含（include）C语言运行环境。我们可以选择提供特定的**链接器参数**（linker argument），也可以选择编译为**裸机目标**（bare metal target）。

### 编译为裸机目标

在默认情况下，Rust尝试适配当前的系统环境，编译可执行程序。举个栗子，如果你使用`x86_64`平台的Windows系统，Rust将尝试编译一个扩展名为`.exe`的Windows可执行程序，并使用`x86_64`指令集。这个环境又被称作你的**宿主系统**（"host" system）。

为了描述不同的环境，Rust使用一个称为**目标三元组**（target triple）的字符串。要查看当前系统的目标三元组，我们可以运行`rustc --version --verbose`：

```
rustc 1.35.0-nightly (474e7a648 2019-04-07)
binary: rustc
commit-hash: 474e7a6486758ea6fc761893b1a49cd9076fb0ab
commit-date: 2019-04-07
host: x86_64-unknown-linux-gnu
release: 1.35.0-nightly
LLVM version: 8.0
```

上面这段输出来自一个`x86_64`平台下的Linux系统。我们能看到，`host`字段的值为三元组`x86_64-unknown-linux-gnu`，它包含了CPU架构`x86_64`、供应商`unknown`、操作系统`linux`和[二进制接口](https://en.wikipedia.org/wiki/Application_binary_interface)`gnu`。

Rust编译器尝试为当前系统的三元组编译，并假定底层有一个类似于Windows或Linux的操作系统提供C语言运行环境——这将导致链接器错误。所以，为了避免这个错误，我们可以另选一个底层没有操作系统的运行环境。

这样的运行环境被称作裸机环境，例如目标三元组`thumbv7em-none-eabihf`描述了一个ARM**嵌入式系统**（[embedded system](https://en.wikipedia.org/wiki/Embedded_system)）。我们暂时不需要了解它的细节，只需要知道这个环境底层没有操作系统——这是由三元组中的`none`描述的。要编译为这个目标，我们需要使用rustup添加它：

```
rustup target add thumbv7em-none-eabihf
```

这行命令将为目标下载一个标准库和core库。这之后，我们就能为这个目标构建独立式可执行程序了：

```
cargo build --target thumbv7em-none-eabihf
```

我们传递了`--target`参数，来为裸机目标系统**交叉编译**（[cross compile](https://en.wikipedia.org/wiki/Cross_compiler)）我们的程序。我们的目标并不包括操作系统，所以链接器不会试着链接C语言运行环境，因此构建过程成功完成，不会产生链接器错误。

我们将使用这个方法编写自己的操作系统内核。我们不将编译到`thumbv7em-none-eabihf`，而是使用描述`x86_64`环境的**自定义目标**（[custom target](https://doc.rust-lang.org/rustc/targets/custom.html)）。在下篇文章中，我们将详细描述一些相关的细节。

### 链接器参数

我们也可以选择不编译到裸机系统，因为传递特定的参数也能解决链接器错误问题。虽然我们不将在后文中使用这个方法，为了教程的完整性，我们也撰写了专门的短文，来提供这个途径的解决方案。

[链接器参数](./appendix-a-linker-arguments.md)

## 小结

一个用Rust编写的最小化的独立式可执行程序应该长这样：

`src/main.rs`：

```rust
#![no_std] // 不链接Rust标准库
#![no_main] // 禁用所有Rust层级的入口点

use core::panic::PanicInfo;

// Linux系统:
#[no_mangle] // 不重整函数名
pub extern "C" fn _start() -> ! {
    // 因为编译器会寻找一个名为`_start`的函数，所以这个函数就是入口点
    // 默认命名为`_start`
    loop {}
}

/// 这个函数将在panic时被调用
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

`Cargo.toml`：

```toml
[package]
name = "crate_name"
version = "0.1.0"
authors = ["Author Name <author@example.com>"]

# 使用`cargo build`编译时需要的配置
[profile.dev]
panic = "abort" # 禁用panic时栈展开

# 使用`cargo build --release`编译时需要的配置
[profile.release]
panic = "abort" # 禁用panic时栈展开
```

选用任意一个裸机目标来编译。比如对`thumbv7em-none-eabihf`，我们使用以下命令：

```bash
cargo build --target thumbv7em-none-eabihf
```

要注意的是，现在我们的代码只是一个Rust编写的独立式可执行程序的一个例子。运行这个二进制程序还需要很多准备，比如在`_start`函数之前需要一个已经预加载完毕的栈。所以为了真正运行这样的程序，我们还有很多事情需要做。

## 下篇预告

下一篇文章要做的事情基于我们这篇文章的成果，它将详细讲述编写一个最小的操作系统内核需要的步骤：如何配置特定的编译目标，如何将可执行程序与引导程序拼接，以及如何把一些特定的字符串打印到屏幕上。
