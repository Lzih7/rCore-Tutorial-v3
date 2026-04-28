# RISC-V 64 汇编语法详解：以 rCore `trap.S` 为例

本文档将结合 `rCore-Tutorial-v3` 中的 [`trap.S`](os/src/trap/trap.S) 文件，详细讲解 RISC-V 64 汇编语言的核心语法、伪指令（Pseudo-instructions）以及宏（Macros）的高级用法。

## 1. 汇编宏与代码生成 (Macros & Code Generation)

在编写底层代码时，如果需要对 32 个通用寄存器进行相同的压栈/弹栈操作，手写会非常繁琐且容易出错。RISC-V 汇编器（如 GNU `as`）提供了一套强大的宏系统。

### 1.1 `.altmacro`
```assembly
.altmacro
```
* **作用**：开启备用宏模式（Alternate Macro Mode）。
* **意义**：在这种模式下，允许使用 `%` 前缀来计算表达式的值，或者使用 `\` 前缀来展开宏参数（例如 `\n`）。这是实现循环生成代码的基础。

### 1.2 定义宏：`.macro` 与 `.endm`
```assembly
.macro SAVE_GP n
    sd x\n, \n*8(sp)
.endm
```
* **`.macro [名字] [参数]`**：定义一个名为 `SAVE_GP` 的宏，接收一个参数 `n`。
* **`\n`**：在宏体内部，`\n` 会被替换为传入的参数值。
* **`sd x\n, \n*8(sp)`**：如果传入 `n=5`，这行代码在展开时就会变成 `sd x5, 40(sp)`（因为 5*8=40）。
* **`.endm`**：标记宏定义的结束。

### 1.3 循环展开：`.rept` 与 `.endr`
```assembly
    .set n, 5
    .rept 27
        SAVE_GP %n
        .set n, n+1
    .endr
```
* **`.set [变量], [值]`**：定义一个汇编时的局部变量 `n`，初始值为 5。
* **`.rept [次数]`**：开始一个循环（Repeat），括号内的代码会被重复生成指定的次数（这里是 27 次）。
* **`%n`**：这是 `.altmacro` 模式下的特有语法，表示“计算变量 `n` 的当前值，并作为字符串传递给宏”。
* **`.endr`**：标记循环结束。
* **效果**：这段代码等价于连续写下了 `SAVE_GP 5`, `SAVE_GP 6` ... 一直到 `SAVE_GP 31`，极大地简化了代码。

## 2. 内存与寄存器操作指令

RISC-V 是一种精简指令集（RISC），它的访存指令非常纯粹，只有 Load（读）和 Store（写）。

### 2.1 存储与加载：`sd` 和 `ld`
```assembly
    sd x1, 1*8(sp)
    ld x1, 1*8(sp)
```
* **`sd` (Store Double-word)**：将 64 位（8 字节）的寄存器数据写入内存。
    * 语法：`sd 源寄存器, 偏移量(基址寄存器)`
    * 例：将 `x1` 寄存器的值，存入地址为 `sp + 8` 的内存空间。
* **`ld` (Load Double-word)**：从内存读取 64 位数据到寄存器。
    * 语法：`ld 目标寄存器, 偏移量(基址寄存器)`

### 2.2 算术与数据传输：`addi` 和 `mv`
```assembly
    addi sp, sp, -34*8
    mv a0, sp
```
* **`addi` (Add Immediate)**：立即数加法。
    * 语法：`addi 目标寄存器, 源寄存器, 立即数`
    * 例：`sp = sp + (-272)`，常用于分配或释放栈空间。
* **`mv` (Move)**：数据移动（伪指令）。
    * 语法：`mv 目标寄存器, 源寄存器`
    * 例：把 `sp` 的值复制给 `a0`。底层实际上被翻译为 `addi a0, sp, 0`。

## 3. 特权级与 CSR (控制状态寄存器) 指令

在操作系统内核中，最关键的操作是读写 CSR（Control and Status Register）。

### 3.1 读写 CSR：`csrr` 和 `csrw`
```assembly
    csrr t0, sstatus
    csrw sstatus, t0
```
* **`csrr` (CSR Read)**：读取 CSR 的值到通用寄存器。
    * 例：读取 `sstatus`（特权级状态寄存器）的值到 `t0`。底层等价于 `csrrw t0, sstatus, zero`。
* **`csrw` (CSR Write)**：将通用寄存器的值写入 CSR。
    * 例：把 `t0` 的值写入 `sstatus`。底层等价于 `csrrw zero, sstatus, t0`。

### 3.2 神奇的原子交换：`csrrw`
```assembly
    csrrw sp, sscratch, sp
```
* **`csrrw` (CSR Read and Write)**：原子性地读取并写入 CSR。
    * 语法：`csrrw 目标寄存器, CSR名字, 源寄存器`
    * 过程：
        1. 先把 CSR 的旧值读出来，准备存入目标寄存器。
        2. 把源寄存器的新值写入 CSR。
        3. 把第一步读出的旧值真正存入目标寄存器。
    * **意义**：在 `trap.S` 中，这行代码通过一条指令，无缝且安全地交换了 `sp`（用户栈）和 `sscratch`（内核栈），是陷入内核的最关键一步。

### 3.3 特权跳转：`sret`
```assembly
    sret
```
* **`sret` (Supervisor Return)**：从 S-Mode 异常处理程序返回。
    * 行为：
        1. 根据 `sstatus` 寄存器中的 `SPP` 位，决定是将特权级保持在 S-Mode 还是降级到 U-Mode。
        2. 将 `sepc` 寄存器中的值拷贝到 PC（程序计数器），实现跳转。
    * **意义**：这是操作系统完成异常处理后，将控制权还给用户程序的终极指令。

## 4. 汇编伪指令与段声明

```assembly
    .section .text
    .globl __alltraps
    .align 2
```
* **`.section .text`**：告诉链接器，接下来的代码属于代码段（`.text`），通常具有可执行权限。
* **`.globl` (或 `.global`)**：声明一个全局符号，使其对其他文件（如 Rust 代码）可见。这样 Rust 代码才能通过 `extern "C"` 调用它。
* **`.align 2`**：内存对齐指令。在 RISC-V 中，参数是 2 的幂次。`2` 表示按照 $2^2 = 4$ 字节对齐。这对于确保指令地址合法性非常重要。

## 5. RISC-V 64 寄存器全景解析

在 `trap.S` 中，我们需要保存和恢复 CPU 的状态。理解 RISC-V 的寄存器分布，是读懂这段代码的核心。

### 5.1 32 个通用寄存器 (General-Purpose Registers)

RISC-V 64 有 32 个 64 位的通用寄存器（`x0` 到 `x31`）。在汇编代码中，我们既可以使用它们的编号（如 `x1`），也可以使用它们的 ABI 别名（如 `ra`）。

| 寄存器 | ABI 名称 | 描述 (Description) | 谁来保存 (Saver) |
| :--- | :--- | :--- | :--- |
| **`x0`** | `zero` | 硬件连线为 0，任何对它的写入都会被丢弃。因此 `trap.S` 中不需要保存它。 | - |
| **`x1`** | `ra` | 返回地址 (Return Address)。保存函数调用结束后的返回位置。 | Caller |
| **`x2`** | `sp` | 栈指针 (Stack Pointer)。指向当前栈顶。在 `trap.S` 中，它是被最后单独处理的特殊存在。 | Callee |
| **`x3`** | `gp` | 全局指针 (Global Pointer)。 | - |
| **`x4`** | `tp` | 线程指针 (Thread Pointer)。应用通常不用它，`trap.S` 中特意跳过了对它的保存。 | - |
| **`x5-x7`** | `t0-t2` | 临时寄存器 (Temporaries)。在 `trap.S` 中被用作数据搬运的临时中转站。 | Caller |
| **`x8-x9`** | `s0/fp`, `s1` | 保存寄存器 (Saved Registers) / 帧指针。 | Callee |
| **`x10-x11`** | `a0-a1` | 函数参数 / 返回值 (Arguments / Return Values)。在 `trap.S` 中，`a0` 被用来向 Rust 函数传递 `TrapContext` 的指针。 | Caller |
| **`x12-x17`** | `a2-a7` | 函数参数 (Arguments)。 | Caller |
| **`x18-x27`** | `s2-s11`| 保存寄存器 (Saved Registers)。 | Callee |
| **`x28-x31`** | `t3-t6` | 临时寄存器 (Temporaries)。 | Caller |

> **💡 在 `trap.S` 中的体现：**
> - 代码中特意跳过了 `x0`（恒为 0）、`x2`（sp 单独处理）和 `x4`（tp 应用程序不用）。
> - 然后用一个 `.rept 27` 循环，暴风吸入式地保存了 `x5` 到 `x31` 的所有寄存器。

### 5.2 特权控制状态寄存器 (CSRs)

除了通用寄存器，RISC-V 还有用于管理系统特权状态的 CSR 寄存器。在陷入内核时，它们决定了操作系统的生死存亡：

1. **`sstatus` (Supervisor Status)**
    * **作用**：记录当前特权级的状态。
    * **关键位**：`SPP` 位记录了陷入内核之前，CPU 是在 U-Mode 还是 S-Mode。当执行 `sret` 时，硬件会读取这个位来决定返回哪个特权级。
2. **`sepc` (Supervisor Exception Program Counter)**
    * **作用**：保存触发异常的那条指令的下一条指令地址（或者引发异常指令本身的地址）。
    * **在 `trap.S` 中的作用**：必须把它保存到栈里，因为处理完异常后，`sret` 指令会把 `sepc` 的值塞回 PC 寄存器，让程序从断点处继续执行。
3. **`sscratch` (Supervisor Scratch)**
    * **作用**：一个为内核预留的“草稿本”寄存器，用来保存一个 64 位的值。
    * **在 `trap.S` 中的魔法**：rCore 用它来**暂存内核栈的栈顶指针**。当用户态程序运行且发生系统调用时，通过 `csrrw sp, sscratch, sp`，只需一条指令，就能把 `sp` 切换成内核栈，同时把原来的用户栈地址备份到 `sscratch` 中！
