+++
title = 'ARM 逆向工程入门：从交叉编译到 GDB 调试'
date = 2026-07-09T22:30:00+08:00
tags = ['ARM', '逆向分析', 'NDK', 'GDB', '移动安全']
categories = ['移动安全']
draft = false
description = '从 NDK 工具链交叉编译、ARM32/ARM64 寄存器体系、GDB 远程调试到 ARM 指令集全览——一篇笔记覆盖 ARM 逆向核心基础。'
+++

## 为什么学 ARM

做 Android 逆向，绕不开 ARM。绝大多数手机 SoC 都是 ARM 架构，native so 库跑在 ARM 指令集上。想看懂反汇编、做动态调试、写 Frida hook 理解寄存器传参，ARM 基础是第一道门槛。

这篇笔记把交叉编译、寄存器、调试、指令集四个核心模块整理在一起，作为 ARM 逆向的知识骨架。

---

## NDK 工具链：交叉编译 ARM 二进制

NDK 提供的 `clang` 可以在 Windows / macOS / Linux 上交叉编译 ARM 二进制。

### 指定目标架构

用 `-target` 参数指定目标平台：

```bash
$NDK/toolchains/llvm/prebuilt/$HOST_TAG/bin/clang++ \
    -target aarch64-linux-android21 foo.cpp
```

主机标记对照表：

| 操作系统 | 主机标记 |
|---------|---------|
| macOS | `darwin-x86_64` |
| Linux | `linux-x86_64` |
| 64 位 Windows | `windows-x86_64` |

架构对照表：

| ABI | target 前缀 |
|-----|------------|
| armeabi-v7a | `armv7a-linux-androideabi` |
| arm64-v8a | `aarch64-linux-android` |
| x86 | `i686-linux-android` |
| x86-64 | `x86_64-linux-android` |

也可以直接用带前缀的编译器，省去 `-target` 参数：

```bash
$NDK/toolchains/llvm/prebuilt/$HOST_TAG/bin/aarch64-linux-android21-clang++ foo.cpp
```

> **查找本机 clang 路径**：PowerShell 用 `gcm clang++ -all` 或 `where.exe clang++`；CMD 用 `where clang++`。

官方文档：[NDK 构建系统指南](https://developer.android.google.cn/ndk/guides/other_build_systems?hl=zh-cn)

---

## 编译四阶段

从源码到可执行文件，经历四个阶段：

| 阶段 | 参数 | 产物 | 说明 |
|------|------|------|------|
| 预处理 | `-E` | `.i` 文件 | 展开宏和头文件 |
| 编译 | `-S` | `.s` 汇编 | 预处理结果编译成汇编代码 |
| 汇编 | `-c` | `.o` 二进制 | 汇编代码编译成目标文件 |
| 链接 | 无参数 | 可执行文件 | 链接目标文件生成可执行程序 |

```bash
# 预处理
clang -target arm-linux-android21 -E hello.c -o hello.i

# 编译为汇编
clang -target arm-linux-android21 -S hello.i -o hello.s

# 汇编为目标文件
clang -target arm-linux-android21 -c hello.s -o hello.o

# 链接为可执行文件
clang -target arm-linux-android21 hello.o -o hello
```

### 推到手机执行

```bash
adb push hello /data/local/tmp/
adb shell chmod +x /data/local/tmp/hello
adb shell /data/local/tmp/hello
```

### 汇编代码结构

编译生成的 `.s` 文件长这样：

```asm
.text                           @ 代码段
.cpu    arm7tdmi                @ CPU 类型
.file   "hello.c"               @ 源文件名
.globl  main                    @ 全局符号 main（入口）
.type   main,%function          @ 类型：函数
.code   32                      @ 32 位 ARM 指令
main:                           @ 函数标号
    .fnstart                    @ 伪指令：函数开始
    push    {r11, lr}           @ 保存帧指针和返回地址
    mov     r11, sp             @ 设置帧指针
    sub     sp, sp, #24         @ 分配栈空间
    ...
    mov     r0, r1              @ 设置返回值
    mov     sp, r11             @ 栈平衡
    pop     {r11, lr}           @ 恢复寄存器
    bx      lr                  @ 返回调用者
    .fnend                      @ 伪指令：函数结束

.section .rodata.str1.1,"aMS",%progbits,1
.L.str:
    .asciz  "Hello %s\r\n"      @ 字符串常量
.L.str.1:
    .asciz  "armv7a"
```

几个要点：
- 以 `.` 开头的是**伪指令**（给汇编器看的，不生成机器码）
- `@` 是 ARM 汇编的注释符
- `.text` 是代码段，`.rodata` 是只读数据段
- `main` 是标号，后面跟冒号

---

## 寄存器体系

### ARM32 寄存器

ARM32 共 16 个核心寄存器（R0-R15）+ 状态寄存器。

#### 通用寄存器

| 寄存器 | 别名 | 用途 |
|--------|------|------|
| R0-R3 | - | 函数前 4 个参数；R0 存返回值 |
| R4-R11 | - | 通用数据存储（callee-saved） |
| R12 | IP | intra-procedure scratch |
| R13 | SP | 栈指针，指向栈顶 |
| R14 | LR | 链接寄存器，存函数返回地址 |
| R15 | PC | 程序计数器，当前执行地址 |

**栈操作方向**：`push` 时 SP 减小（向低地址），`pop` 时 SP 增大（向高地址）。

**PC 流水线特性**：ARM 状态下 PC 指向当前指令 +8（三级流水线，正在执行的地址后面两条），Thumb 状态下 +4。这在调试时读 PC 值要注意。

#### 状态寄存器

**CPSR**（当前程序状态寄存器）：

| 标志位 | 含义 |
|--------|------|
| N | 结果为负数（最高位为 1） |
| Z | 结果为零 |
| C | 无符号运算进位/借位 |
| V | 有符号运算溢出 |
| I | IRQ 中断禁止 |
| T | ARM/Thumb 状态切换 |

**SPSR**：异常处理时保存 CPSR 状态，异常返回后恢复。

### ARM64 寄存器

ARM64 在 32 位基础上扩展，寄存器从 32 位变成 64 位，数量也增加了。

| 寄存器 | 用途 |
|--------|------|
| X0-X28 | 64 位通用寄存器；X0-X7 传递前 8 个参数 |
| X29 (FP) | 帧指针，用于栈回溯和调试 |
| X30 (LR) | 返回地址 |
| SP | 64 位栈指针 |

状态寄存器从 CPSR 升级为 **PSTATE**，新增了 `D`（调试屏蔽）和 `A`（SERROR 异常屏蔽）等控制位。

---

## GDB 远程调试 ARM

### 安装多架构 GDB

普通 GDB 可能不支持 ARM 架构。用 `set architecture` 命令可以检查。

推荐安装 `gdb-multiarch`，基本支持所有平台：

```bash
sudo apt-get install gdb-multiarch
gdb-multiarch --version
```

> 推荐在 Linux 或 macOS 下使用，Windows 用 WSL。配合 [GEF](https://github.com/hugsy/gef) 插件效果更佳。

### 手机端：gdbserver

将 ARM 版 `gdbserver` 推到手机上运行：

```bash
# 推送 gdbserver
adb push gdbserver /data/local/tmp/
adb shell chmod +x /data/local/tmp/gdbserver

# 启动 gdbserver 调试目标程序
adb shell /data/local/tmp/gdbserver :12345 /data/local/tmp/hello
```

### PC 端：连接调试

```bash
# 端口映射
adb forward tcp:12345 tcp:12345

# 进入 GDB
gdb-multiarch

# 连接远程
(gdb) target remote 127.0.0.1:12345
```

### 常用调试命令

| 命令 | 作用 |
|------|------|
| `b main` | 在 main 函数下断点 |
| `b *0x004006d0` | 在指定地址下断点（用 `*`） |
| `info b` | 查看断点信息 |
| `disassemble main` | 反汇编函数 |
| `c` | 继续执行到断点 |
| `ni` | 单步步过（汇编级） |
| `n` | 单步步过（源码级） |
| `si` | 单步步入（进入函数） |
| `finish` | 执行到函数返回 |
| `p/x $r0` | 十六进制打印寄存器值 |

> 调试汇编用 `ni` 逐条执行。注意 ARM 的 PC 值在当前指令下方两条指令处（三级流水线）。

### 调试实例：逐条分析

下面是一个 `hello` 程序 main 函数的逐指令调试记录，展示了每条指令对寄存器和栈的影响：

```asm
push {r11, lr}     @ 保存 r11 和 lr 到栈
```

执行后，栈中多出了 r11 和 lr 的值，SP 向低地址移动。

```asm
mov r11, sp        @ 将 sp 保存到 r11（帧指针）
sub sp, sp, #24    @ 栈分配 24 字节空间
```

```asm
mov r2, #0         @ r2 = 0
str r2, [r11, #-4] @ 将 r2 写入 [r11-4] 内存
str r0, [r11, #-8] @ 将 r0 写入 [r11-8] 内存
str r1, [sp, #12]  @ 将 r1 写入 [sp+12] 内存
```

```asm
ldr r0, [pc, #40]  @ 从 PC+40 读数据到 r0（PC-relative 取地址）
add r0, pc, r0     @ r0 = pc + r0，得到字符串实际地址
                   @ r0 -> "hello %s!\r\n"
```

```asm
bl printf@plt      @ 调用 printf
                   @ 参数：r0 = 格式串, r1 = "arm"
```

```asm
mov r0, r1         @ 设置返回值
mov sp, r11        @ 栈平衡：恢复 SP
pop {r11, lr}      @ 恢复寄存器
bx lr              @ 返回调用者
```

整个流程完整展示了：**函数序言（push/mov r11,sp/sub sp）→ 函数体 → 函数尾声（mov sp,r11/pop/bx lr）** 的标准结构。

---

## ARM 指令集

### 寻址方式

**1. 立即数寻址** — 操作数直接编码在指令中

```asm
MOV R0, #0x12      @ R0 = 0x12
```

**2. 寄存器寻址** — 操作数在寄存器中

```asm
ADD R1, R2, R3     @ R1 = R2 + R3
```

**3. 寄存器间接寻址** — 寄存器存的是地址

```asm
LDR R0, [R1]       @ R0 = 内存[R1]
```

**4. 基址变址寻址** — 基址寄存器 + 偏移

```asm
LDR R0, [R1, #4]   @ R0 = 内存[R1 + 4]
```

**5. 多寄存器寻址** — 批量传输

```asm
LDMIA R0!, {R1-R3} @ 从 R0 地址批量加载到 R1/R2/R3，每次加载后 R0 递增
```

**6. 相对寻址** — PC + 偏移

```asm
B label            @ 跳转到 label（PC + 偏移量）
```

### 数据处理指令

| 指令 | 作用 | 示例 |
|------|------|------|
| `MOV` | 数据移动 | `MOV R0, #0x12` |
| `ADD` | 加法 | `ADD R1, R2, R3` → R1=R2+R3 |
| `SUB` | 减法 | `SUB R0, R1, #1` |
| `AND` | 逻辑与 | `AND R1, R2, R3` → R1=R2&R3 |
| `ORR` | 逻辑或 | `ORR R1, R2, #3` |
| `EOR` | 逻辑异或 | `EOR R1, R2, R0` → R1=R2^R0 |
| `CMP` | 比较（更新标志位） | `CMP R0, R1` 后接条件跳转 |
| `BIC` | 位清零 | `BIC R0, R0, #0x0F` 清低 4 位 |

`CMP` 配合条件后缀使用：

```asm
CMP R0, R1
BLE label          @ 如果 R0 <= R1，跳转到 label
```

### 加载和存储指令

| 指令 | 宽度 | 方向 |
|------|------|------|
| `LDR` / `STR` | 4 字节 | 加载 / 存储 |
| `LDRH` / `STRH` | 2 字节 | 加载 / 存储 |
| `LDRB` / `STRB` | 1 字节 | 加载 / 存储 |

```asm
LDR R0, [R1]       @ 从内存[R1]加载 4 字节到 R0
STR R2, [R3, #4]   @ 将 R2 存到内存[R3+4]
```

批量加载/存储：

| 指令 | 说明 |
|------|------|
| `LDM` | 从内存加载多个寄存器 |
| `STM` | 将多个寄存器存到内存 |
| `STMFD` | 满递减栈存储（等同 `push`） |
| `LDMFD` | 满递减栈加载（等同 `pop`） |

```asm
push {r4-r11, lr}      @ 等同于 STMFD sp!, {r4-r11, lr}
pop  {r4-r11, lr}      @ 等同于 LDMFD sp!, {r4-r11, lr}
```

### 跳转指令

| 指令 | 作用 |
|------|------|
| `B` | 无条件跳转 |
| `BL` | 跳转 + 保存返回地址到 LR（函数调用） |
| `BX` | 跳转到寄存器地址 + 切换 ARM/Thumb 状态 |
| `BLX` | 带返回 + 带状态切换的跳转 |

```asm
B    label          @ 无条件跳转
BL   function       @ 调用函数，返回地址存入 LR
BX   R0             @ 跳转到 R0 地址，最低位决定 ARM/Thumb
```

> `BX LR` 是函数返回的标配。当 R0 最低位为 0 时进入 ARM 状态，为 1 时进入 Thumb 状态。

### 乘法指令

```asm
MUL   R2, R0, R1         @ R2 = R0 * R1
MLA   R3, R0, R1, R2     @ R3 = R0 * R1 + R2
SMULL R0, R1, R2, R3     @ R0 = (R2*R3)低32位, R1 = 高32位（有符号）
SMLAL R0, R1, R2, R3     @ R0 += (R2*R3)低32位, R1 += 高32位
UMULL R0, R1, R2, R3     @ 无符号版 64 位乘法
```

### 移位指令

| 指令 | 作用 |
|------|------|
| `LSL` | 逻辑左移 |
| `LSR` | 逻辑右移 |
| `ASR` | 算术右移（保留符号位） |
| `ROR` | 循环右移 |
| `RRX` | 扩展循环右移（带进位位） |

```asm
LSL R0, R1, #2     @ R0 = R1 << 2
LSR R2, R3, #3     @ R2 = R3 >> 3
ASR R4, R5, #1     @ R4 = R5 算术右移 1 位
ROR R6, R7, #4     @ R6 = R7 循环右移 4 位
```

---

## 参考资源

- [NDK 构建系统指南](https://developer.android.google.cn/ndk/guides/other_build_systems?hl=zh-cn)
- [ARM 架构参考手册（ARMv7）](https://developer.arm.com/documentation/ddi0406/cd)
- [GEF - GDB 增强插件](https://github.com/hugsy/gef)
- [GDB 源码下载](http://ftp.gnu.org/gnu/gdb)

---

*这篇笔记持续更新，后续会补充 ARM Shellcode 编写、ROP 链构造等进阶内容。*
