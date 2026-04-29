# ch1

## 目录结构

```text
./os/src
Rust        4 Files   119 Lines
Assembly    1 Files    11 Lines

├── bootloader(内核依赖的运行在 M 特权级的 SBI 实现，本项目中我们使用 RustSBI)
│   └── rustsbi-qemu.bin(可运行在 qemu 虚拟机上的预编译二进制版本)
├── LICENSE
├── os(我们的内核实现放在 os 目录下)
│   ├── Cargo.toml(内核实现的一些配置文件)
│   ├── Makefile
│   └── src(所有内核的源代码放在 os/src 目录下)
│       ├── console.rs(将打印字符的 SBI 接口进一步封装实现更加强大的格式化输出)
│       ├── entry.asm(设置内核执行环境的的一段汇编代码)
│       ├── lang_items.rs(需要我们提供给 Rust 编译器的一些语义项，目前包含内核 panic 时的处理逻辑)
│       ├── linker-qemu.ld(控制内核内存布局的链接脚本以使内核运行在 qemu 虚拟机上)
│       ├── main.rs(内核主函数)
│       └── sbi.rs(调用底层 SBI 实现提供的 SBI 接口)
├── README.md
└── rust-toolchain(控制整个项目的工具链版本)
```

## 三叶虫 Lib\_OS

- 分为两层 --> S-MODE, M-MODE
  - S-MODE: 提供基本的系统调用接口(操作系统的模式，监管者模式)
  - M-MODE: 提供最高级的系统调用接口

## Start

1. `$ cargo new os --bin`: 使用cargo工具创建一个可执行程序项目
   ```bash
   $ tree os
   os
   ├── Cargo.toml
   └── src
       └── main.rs
   ```
   - 可使用`$ cargo run`来运行项目
   - Rust标准库与核心库
     在rscv环境下运行时:
     ```bash
     $ cargo run --target riscv64gc-unknown-none-elf
     Compiling os v0.1.0 (/home/shinbokuow/workspace/v3/rCore-Tutorial-v3/os)
     error[E0463]: can't find crate for `std`
       |
       = note: the `riscv64gc-unknown-none-elf` target may not be installed
     ```
2. 移出标准库依赖
   - 在.cargo目录下新建config.toml文件，内容如下:
   ```toml
   [build]
   target = "riscv64gc-unknown-none-elf"
   ```
   - 删除main函数，增加lang\_items.rs文件
   ```rust
   // os/src/main.rs
   #![no_main]
   #![no_std]
   mod lang_items;
   // ... other code

   // os/src/lang_items.rs
   use core::panic::PanicInfo;

   #[panic_handler]
   fn panic(_info: &PanicInfo) -> ! {
       loop {}
   }
   ```
3. QEMU启动
   - 预加载：
     - 物理起始地址为0x80000000
     - 把作为 bootloader 的 rustsbi-qemu.bin 加载到物理内存以物理地址 0x80000000 开头的区域上
     - 同时把内核镜像 os.bin 加载到以物理地址 0x80200000 开头的区域上
   - 步骤：
   - PC指向0x100，执行几条指令后，跳转至0x80000000
   - 运行 rustsbi-qemu.bin
   - 跳转至下一阶段的地址(0x80200000)，运行 os.bin
4. 编写汇编(entry.asm)
   - 进入内核后的第一条指令
   - 在main.rs中`use core::arch::global_asm;`, `global_asm!(include_str!("entry.asm"));`导入汇编
5. 使用链接脚本调整内存布局
   - 使用链接脚本
   ```toml
    // os/.cargo/config
    [build]
    target = "riscv64gc-unknown-none-elf"

    [target.riscv64gc-unknown-none-elf]
    rustflags = [
        "-Clink-arg=-Tsrc/linker-qemu.ld", "-Cforce-frame-pointers=yes"
    ]
   ```
6. 函数调用支持
   - 每次调用开辟栈帧，以下是结构：
   ```
       Father_StackFrame
             |
             ra            <-- fp
             |
           prev_fp
             |
         Callee_Saved
             |
       Local_Variables      <-- sp
   ```
   - 其中prev\_fp保存了父栈帧的结束地址，构成了一条调用链
   - 在entry.asm里分配并使用启动栈：
   ```rust
   # os/src/entry.asm
       .section .text.entry
       .globl _start
   _start:
       la sp, boot_stack_top
       call rust_main

       .section .bss.stack
       .globl boot_stack_lower_bound
   boot_stack_lower_bound:
       .space 4096 * 16
       .globl boot_stack_top
   boot_stack_top:
   ```
7. RustSBI服务
   - 它会在计算机启动时进行它所负责的环境初始化工作，并将计算机控制权移交给内核
   - 但 RustSBI 实际上作为**内核的执行环境**，还有另一项职责：即在内核运行时**响应内核的请求**为内核提供服务
   - 我们编写的 OS 内核位于 Supervisor 特权级，而 RustSBI 位于 Machine 特权级，也是**最高的特权级**
   - SBI: Supervisor Binary Interface，是内核与 RustSBI 之间的接口
   - 实现：
     1. 内核与 RustSBI 通信的相关功能实现在子模块 sbi 中， 在 sbi.rs 中直接调用接口实现输出字符
     2. 在 sbi.rs 中实现关机功能
     3. 在 console.rs 里实现格式化输出(println)

# ch2

## 目录结构

```
./os/src
Rust        13 Files   372 Lines
Assembly     2 Files    58 Lines

├── bootloader
│   └── rustsbi-qemu.bin
├── LICENSE
├── os
│   ├── build.rs(新增：生成 link_app.S 将应用作为一个数据段链接到内核)
│   ├── Cargo.toml
│   ├── Makefile(修改：构建内核之前先构建应用)
│   └── src
│       ├── batch.rs(新增：实现了一个简单的批处理系统)
│       ├── console.rs
│       ├── entry.asm
│       ├── lang_items.rs
│       ├── link_app.S(构建产物，由 os/build.rs 输出)
│       ├── linker-qemu.ld
│       ├── main.rs(修改：主函数中需要初始化 Trap 处理并加载和执行应用)
│       ├── sbi.rs
│       ├── sync(新增：同步子模块 sync ，目前唯一功能是提供 UPSafeCell)
│       │   ├── mod.rs
│       │   └── up.rs(包含 UPSafeCell，它可以帮助我们以更 Rust 的方式使用全局变量)
│       ├── syscall(新增：系统调用子模块 syscall)
│       │   ├── fs.rs(包含文件 I/O 相关的 syscall)
│       │   ├── mod.rs(提供 syscall 方法根据 syscall ID 进行分发处理)
│       │   └── process.rs(包含任务处理相关的 syscall)
│       └── trap(新增：Trap 相关子模块 trap)
│           ├── context.rs(包含 Trap 上下文 TrapContext)
│           ├── mod.rs(包含 Trap 处理入口 trap_handler)
│           └── trap.S(包含 Trap 上下文保存与恢复的汇编代码)
├── README.md
├── rust-toolchain
└── user(新增：应用测例保存在 user 目录下)
   ├── Cargo.toml
   ├── Makefile
   └── src
      ├── bin(基于用户库 user_lib 开发的应用，每个应用放在一个源文件中)
      │   ├── 00hello_world.rs
      │   ├── 01store_fault.rs
      │   ├── 02power.rs
      │   ├── 03priv_inst.rs
      │   └── 04priv_csr.rs
      ├── console.rs
      ├── lang_items.rs
      ├── lib.rs(用户库 user_lib)
      ├── linker.ld(应用的链接脚本)
      └── syscall.rs(包含 syscall 方法生成实际用于系统调用的汇编指令，
                     各个具体的 syscall 都是通过 syscall 来实现的)
```

## 简介

- 内核态 -> 操作系统
- 用户态 -> 应用程序，进行系统调用
### 调用关系
- Syscall: 用户态到内核态
  - app 里的 syscall 只负责发送请求
  - OS 里的 syscall 负责分发任务
- Sbicall: 内核态到RustSBI态

## Steps

### 特权级机制

- 限制应用程序：内存空间、指令
- 特权级架构：

| 级别 | 编码 | 名称       |
| -- | -- | -------- |
| 0  | 00 | 用户模式-U   |
| 1  | 01 | 监督模式-S   |
| 2  | 10 | 虚拟监督模式-H |
| 3  | 11 | 机器模式-M   |

- app -> ABI -> OS -> SBI -> SEE
- 传统的函数调用会绕过硬件的特权级保护检查，所以RSCV提供了机器指令`ecall, sret`
  - ecall:  具有**用户态到内核态**的执行环境切换能力的函数调用指令
  - sret: 具有内核态到用户态的执行环境切换能力的函数返回指令

### 实现应用程序
- 我们在`lib.rs`里定义了用户库的入口`_start`，并将汇编后的代码放入`.text.entry`段
```rust
#[no_mangle]
#[link_section = ".text.entry"]
pub extern "C" fn _start() -> ! {
    clear_bss();
    exit(main());
    panic!("unreachable after sys_exit!");
}
```
- `lib.rs`里的main函数(弱链接，链接器优先使用bin里的main函数)
```rust
fn main() -> i32 {
    println!("Hello, world!");
    0
}
```
- 内存布局
  - 将BASE_ADDR设置为0x80400000
  - _start放在开头
- 系统调用
  - 应用程序通过`ecall`指令调用系统提供的**接口**，`ecall`会触发异常，并Trap进入S-Mode
  - 这个接口可以成为ABI或者系统调用
```rust
fn syscall(id: usize, args: [usize; 3]) -> isize {
    let mut ret: isize;
    unsafe {
        asm!(
            "ecall",
            inlateout("x10") args[0] => ret,
            in("x11") args[1],
            in("x12") args[2],
            in("x17") id
        );
    }
    ret
}
```
  - 与ABI类似，a0保存返回值，a0-a6传递参数，a7传递syscallID
  - x10-x17对应a0-a7，x1对应ra

```rust
pub fn sys_write(fd: usize, buffer: &[u8]) -> isize {
    syscall(SYSCALL_WRITE, [fd, buffer.as_ptr() as usize, buffer.len()])
}
```
  - fd: 文件描述符，表示要向哪里写数据
```
  fd_table (per-process):
  0 -> stdin
  1 -> stdout
  2 -> stderr
  3 -> /tmp/a.txt
  4 -> (empty)
  5 -> socket(...)
```
- 进一步封装
  - 在lib.rs中
  ```rust
  pub fn write(fd: usize, buf: &[u8]) -> isize {
    sys_write(fd, buf)
  }
  ```
  - 在console.rs中
  ```rust
  impl Write for Stdout {
      fn write_str(&mut self, s: &str) -> fmt::Result {
          write(STDOUT, s.as_bytes());
          Ok(())
      }
  }
  ```
- 运行
```bash
$ cd user
$ make build
$ cd target/riscv64gc-unknown-none-elf/release/
# 确认待执行的应用为 ELF 格式
$ file 03priv_inst
03priv_inst: ELF 64-bit LSB executable, UCB RISC-V, version 1 (SYSV), statically linked, not stripped
# 执行特权指令出错
$ qemu-riscv64 ./03priv_inst
Try to execute privileged instruction in U Mode
Kernel should kill this application!
Illegal instruction (core dumped)
# 执行访问特权级 CSR 的指令出错
$ qemu-riscv64 ./04priv_csr
Try to access privileged CSR in U Mode
Kernel should kill this application!
```
- 注意链接器里内存要对齐
### 实现批处理的操作系统
- 应用加载机制 -> 在操作系统和应用程序需要放置在同一可执行程序时，需要让 os 更快地找到应用程序，以实现简洁的加载方式
- 将用户态文件(.bin)塞进内核镜像，使用了`link_app.S`，它由`build.rs`生成
#### 应用管理器
- 在 batch.rs 中，我们定义了一个结构体 `AppManager`，用于管理应用程序的加载和切换
	- 它包含三个字段：`num_app`、`current_app`、`app_start`
1. 内核启动，定义一个全局的APP_MANAGER
2. 调用 run_next_app()，准备跑下一个程序
3. load_app()，实现加载应用程序
	- 清空场地(0x80400000-0x80420000)
	- 根据 app_start 数组，找到当前 App 在内核数据段里的“压缩包”位置。然后，直接用内存拷贝(copy_from_slice)，把这段纯机器码复制到 0x80400000
	- 刷新内存( asm!("fence.i") )
- lazy_static!：用于定义全局变量
	- 全局变量必须在编译期确定值
	- lazy_static! 宏义的全局变量，会在运行时初始化，而不是在编译时初始化

### 实现特权级切换
- 特权级切换通过 Trap 实现
- 寄存器(CSR)
	- sstatus: 特权级状态寄存器
	- sepc: 异常程序计数器(记录Trap前执行的最后一条指令的地址)
	- scause: 异常原因寄存器
	- stval: 异常值寄存器
	- stvec: 异常向量寄存器(控制Trap处理代码的入口地址)
- 流程：
	1. sstatus 的 SPP 字段会被修改为 CPU 当前的特权级（U/S）。

	2. sepc 会被修改为被中断或触发异常的指令的地址。如 CPU 执行 ecall 指令会触发异常，则 sepc 会被设置为 ecall 指令的地址。

	3. scause/stval 分别会被修改成这次 Trap 的原因以及相关的附加信息。

	4. CPU 会跳转到 stvec 所设置的 Trap 处理入口地址，并将当前特权级设置为 S ，然后从Trap 处理入口地址处开始执行。
- 用户栈和内核栈
	- 在`batch.rs`里定义
#### Trap处理
1. Trap 的上下文切换
```rust
// os/src/trap/mod.rs

global_asm!(include_str!("trap.S"));

pub fn init() {
    extern "C" { fn __alltraps(); }
    unsafe {
        stvec::write(__alltraps as usize, TrapMode::Direct);
    }
}
```

- __alltrap 是上下文保护函数
- __restore 是上下文恢复函数
- 通过 __alltrap 将 Trap 上下文保存到内核栈上，并将 stvec 设置为 Direct 模式指向 __alltraps 的地址
2. Trap 的分发和处理
```rust
// os/src/trap/mod.rs

#[no_mangle]
pub fn trap_handler(cx: &mut TrapContext) -> &mut TrapContext {
    let scause = scause::read();
    let stval = stval::read();
    match scause.cause() {
        Trap::Exception(Exception::UserEnvCall) => {
            cx.sepc += 4;
            cx.x[10] = syscall(cx.x[17], [cx.x[10], cx.x[11], cx.x[12]]) as usize;
        }
        Trap::Exception(Exception::StoreFault) |
        Trap::Exception(Exception::StorePageFault) => {
            println!("[kernel] PageFault in application, kernel killed it.");
            run_next_app();
        }
        Trap::Exception(Exception::IllegalInstruction) => {
            println!("[kernel] IllegalInstruction in application, kernel killed it.");
            run_next_app();
        }
        _ => {
            panic!("Unsupported trap {:?}, stval = {:#x}!", scause.cause(), stval);
        }
    }
    cx
}
```
- 依赖外部 crate

### 实现系统调用
- syscall 函数并不实现处理请求，而是根据 syscall ID 分发到具体函数
```rust
// os/src/syscall/mod.rs

pub fn syscall(syscall_id: usize, args: [usize; 3]) -> isize {
    match syscall_id {
        SYSCALL_WRITE => sys_write(args[0], args[1] as *const u8, args[2]),
        SYSCALL_EXIT => sys_exit(args[0] as i32),
        _ => panic!("Unsupported syscall_id: {}", syscall_id),
    }
}
```
- 处理函数
```rust
// os/src/syscall/fs.rs

const FD_STDOUT: usize = 1;

pub fn sys_write(fd: usize, buf: *const u8, len: usize) -> isize {
    match fd {
        FD_STDOUT => {
            let slice = unsafe { core::slice::from_raw_parts(buf, len) };
            let str = core::str::from_utf8(slice).unwrap();
            print!("{}", str);
            len as isize
        },
        _ => {
            panic!("Unsupported fd in sys_write!");
        }
    }
}

// os/src/syscall/process.rs

pub fn sys_exit(xstate: i32) -> ! {
    println!("[kernel] Application exited with code {}", xstate);
    run_next_app()
}
```

# ch3
## 分时多任务操作系统
```
./os/src
Rust        18 Files   511 Lines
Assembly     3 Files    82 Lines

├── bootloader
│   └── rustsbi-qemu.bin
├── LICENSE
├── os
│   ├── build.rs
│   ├── Cargo.toml
│   ├── Makefile
│   └── src
│       ├── batch.rs(移除：功能分别拆分到 loader 和 task 两个子模块)
│       ├── config.rs(新增：保存内核的一些配置)
│       ├── console.rs
│       ├── entry.asm
│       ├── lang_items.rs
│       ├── link_app.S
│       ├── linker-qemu.ld
│       ├── loader.rs(新增：将应用加载到内存并进行管理)
│       ├── main.rs(修改：主函数进行了修改)
│       ├── sbi.rs(修改：引入新的 sbi call set_timer)
│       ├── sync
│       │   ├── mod.rs
│       │   └── up.rs
│       ├── syscall(修改：新增若干 syscall)
│       │   ├── fs.rs
│       │   ├── mod.rs
│       │   └── process.rs
│       ├── task(新增：task 子模块，主要负责任务管理)
│       │   ├── context.rs(引入 Task 上下文 TaskContext)
│       │   ├── mod.rs(全局任务管理器和提供给其他模块的接口)
│       │   ├── switch.rs(将任务切换的汇编代码解释为 Rust 接口 __switch)
│       │   ├── switch.S(任务切换的汇编代码)
│       │   └── task.rs(任务控制块 TaskControlBlock 和任务状态 TaskStatus 的定义)
│       ├── timer.rs(新增：计时器相关)
│       └── trap
│           ├── context.rs
│           ├── mod.rs(修改：时钟中断相应处理)
│           └── trap.S
├── README.md
├── rust-toolchain
└── user
    ├── build.py(新增：使用 build.py 构建应用使得它们占用的物理地址区间不相交)
    ├── Cargo.toml
    ├── Makefile(修改：使用 build.py 构建应用)
    └── src
        ├── bin(修改：换成第三章测例)
        │   ├── 00power_3.rs
        │   ├── 01power_5.rs
        │   ├── 02power_7.rs
        │   └── 03sleep.rs
        ├── console.rs
        ├── lang_items.rs
        ├── lib.rs
        ├── linker.ld
        └── syscall.rs
```
### 多道程序的放置和加载
- 在 ch2 中，我们将每个 app 都加载到相同的地址上，但在 ch3 中，由于有多个 app ，每个 app 都需要一个独立的物理地址区间
- 我们可以使用 build.py 构建应用，使得它们占用的物理地址区间不相交
- 我们可以在 loader.rs 中添加代码，将应用加载到内存中
#### 构建应用 build.rs
```python
import os

base_address = 0x80400000
step = 0x20000
linker = 'src/linker.ld'

app_id = 0
apps = os.listdir('src/bin')
apps.sort()
for app in apps:
    app = app[:app.find('.')]
    lines = []
    lines_before = []
    with open(linker, 'r') as f:
        for line in f.readlines():
            lines_before.append(line)
            line = line.replace(hex(base_address), hex(base_address+step*app_id))
            lines.append(line)
    with open(linker, 'w+') as f:
        f.writelines(lines)
    os.system('cargo build --bin %s --release' % app)
    print('[build.py] application %s start with address %s' %(app, hex(base_address+step*app_id)))
    with open(linker, 'w+') as f:
        f.writelines(lines_before)
    app_id = app_id + 1
```
1. 找到 src/linker.ld 中的 `BASE_ADDRESS = 0x80400000;` 这一行，并将后面的地址替换为和当前应用对应的一个地址
2. 使用 cargo build 构建当前的应用，注意我们可以使用 --bin 参数来只构建某一个应用
3. 将 src/linker.ld 还原

#### 加载应用
```rust
 // os/src/loader.rs

 pub fn load_apps() {
     extern "C" { fn _num_app(); }
     let num_app_ptr = _num_app as usize as *const usize;
     let num_app = get_num_app();
     let app_start = unsafe {
         core::slice::from_raw_parts(num_app_ptr.add(1), num_app + 1)
     };
     // load apps
     for i in 0..num_app {
         let base_i = get_base_i(i);
         // clear region
         (base_i..base_i + APP_SIZE_LIMIT).for_each(|addr| unsafe {
             (addr as *mut u8).write_volatile(0)
         });
         // load app from data section to memory
         let src = unsafe {
             core::slice::from_raw_parts(
                 app_start[i] as *const u8,
                 app_start[i + 1] - app_start[i]
             )
         };
         let dst = unsafe {
             core::slice::from_raw_parts_mut(base_i as *mut u8, src.len())
         };
         dst.copy_from_slice(src);
     }
     unsafe {
         asm!("fence.i");
     }
 }

  // os/src/loader.rs

 fn get_base_i(app_id: usize) -> usize {
     APP_BASE_ADDRESS + app_id * APP_SIZE_LIMIT
 }
```
