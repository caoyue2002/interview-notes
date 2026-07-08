## 1. 编译产物

### ELF —— 可执行文件容器

- Keil 里叫 `.axf`，本质就是 ELF。
- 内部分段（section）：
    - `.text`：机器码
    - `.data`：已初始化全局变量
    - `.bss`：未初始化全局变量
    - `.debug_*`：调试信息（即 [[#DWARF]]）
    - 符号表：函数/变量名 ↔ 地址

### DWARF —— ELF 里的「调试翻译表」

- 装在 ELF `.debug_*` 段里，gcc加 `-g` 才生成。
- 包含五类信息：
    1. **行号映射**：源码行 ↔ 机器地址
    2. **变量信息**：变量名 ↔ 地址/寄存器 ↔ 类型
    3. **类型定义**：int/struct 的布局
    4. **函数信息**：地址、参数、返回值
    5. **作用域**：变量在哪可见
- 没有 DWARF → GDB 只能看裸地址和汇编。

### 衍生文件

| 文件          | 谁生成        | 装什么             | 用途      |
| ----------- | ---------- | --------------- | ------- |
| `.elf/.axf` | 链接器        | 代码+数据+**调试信息**  | 调试加载    |
| `.hex`      | objcopy 提取 | 代码+地址，**无调试信息** | 烧录      |
| `.bin`      | objcopy 提取 | 纯二进制            | 烧录      |
| `.map`      | 链接器        | 内存地址清单（纯文本）     | 看占用/查地址 |

> [!important] 关键认知：调试信息永远不进芯片 烧进 Flash 的只有 hex/bin（纯代码）；**DWARF 调试信息一直在电脑的 ELF/axf 里，不占芯片一个字节**。所以不存在「Flash 满了塞不下 ELF」——ELF 根本不进芯片。

---

## 2.调试机制

### ptrace

- **process trace**，Linux **系统调用**。
- GDB 作为「控制者」进程，通过 ptrace 附身被调试进程：读写内存/寄存器、暂停/继续、收信号。
- 连接：`gdb ./program`（启动）、`gdb -p PID`（附加）。

### INT 3 / 0xCC —— 软件断点

- **INT 3** 是 x86 **陷阱指令**，机器码 **0xCC**（一字节）。同一指令两种表示。
- 实现流程：
    1. GDB 记下目标地址原指令首字节
    2. 替换成 `0xCC`
    3. 程序执行到此 → 触发 SIGTRAP → 内核接住 → 通知 GDB → 暂停
    4. GDB 换回原字节，`continue` 逻辑不变
- 因为改内存，**软件断点数量不限**。

### 单步执行

- 靠 CPU 的 **Trap Flag（TF）**：置 1 后每执行一条指令触发 SIGTRAP。
- `step`（源码行）vs `stepi`（机器指令），靠 DWARF 映射。

---

## 3. 单片机调试体系（硬件机制）

### 物理链路全景

```
电脑 → USB → 调试器 → SWD → 芯片[DAP → 内存/FPB/DWT/ITM]
```
- 电脑只有 **USB**，芯片只认 **SWD**，物理上接不上。
- ST-Link/J-Link/DAPLink 本质都是 **「USB 转 SWD」转接硬件**（翻译官）。

### SWD / JTAG —— 调试物理接口

- **SWD**：2根线（SWCLK + SWDIO），现代 MCU 首选，简洁。
- **JTAG**：4~5根线（TCK/TMS/TDI/TDO），引脚多，兼容老芯片/FPGA/多芯片链。
- 两者都能配合 GDB 使用，选哪种是 OpenOCD 的事，GDB 不感知。
- STM32 上 SWCLK/SWDIO 就是 TCK/TMS 复用，J-Link 可以选协议。

### DAP —— 芯片的「调试大门」

- **D**ebug **A**ccess **P**ort，Cortex-M 内核里的硬件模块，**外部进入芯片的唯一入口**。
- 内部分 **DP**（对外接 SWD）+ **AP**（对内访问总线/内存）。
- 「SWD 是路，DAP 是门」。

> [!warning] DAP ≠ DAPLink
> 
> - **DAP** 在芯片**内核里**（调试入口）
> - **DAPLink** 是芯片**外**的开源调试器硬件（去连目标的 DAP）

### FPB —— 硬件断点单元（Flash Patch and Breakpoint）

#### 功能1：硬件断点（核心）
- 把断点地址写进 FPB 的**比较寄存器**
- CPU 每次取指，FPB 硬件实时比较 **PC 值**与断点地址
- 地址匹配 → FPB 直接让 CPU 进入 **Debug Halt**（停止取指/执行，寄存器定格）
- **不改任何指令，纯硬件比较，数量有限：M0/M0+ 4个，M3/M4 6个，M7 8个**

#### 功能2：Flash Patch（突破数量限制）
- 硬件断点超出数量后，用这个实现软件断点
- 原理：把 Flash 某地址**重映射到 RAM**，在 RAM 里放 BKPT 指令
- CPU 取指时被 FPB 拦截，偷偷去 RAM 取 BKPT → 触发断点
- Flash 本身完全没被改动，CPU 被「骗」去执行 RAM 里的陷阱指令
#### Debug Halt （调试停止状态）时芯片状态
| 部件             | 状态              |
| -------------- | --------------- |
| CPU 内核         | 停止取指/执行，PC 定格   |
| 主频/时钟          | 还在运行            |
| 外设/定时器/PWM     | **默认还在跑！**      |
| **独立看门狗 IWDG** | 默认**不冻结**，会超时复位 |
| **窗口看门狗 WWDG** | 默认**不冻结**，会超时复位 |

### DWT —— 数据监视与跟踪（多功能）

- **D**ata **W**atchpoint and **T**race。功能不止计时：
    - **CYCCNT 周期计数器**：常被当「微秒级定时器」测函数耗时（周期数 ÷ 主频）
    - **数据观察点**：监视地址，值变就触发（数据断点）
    - **事件计数**、**PC 采样**（性能分析）
- DWT **采集**，数据经 ITM **输出**。

### ITM —— 高速信息输出通道

- **I**nstrumentation **T**race **M**acrocell，经 **SWO 引脚**高速输出 printf/trace。
- 比串口快、几乎不影响实时性。
- **独立于 DAP**：配置靠 DAP 写寄存器，数据输出走 SWO，不挤 SWD 带宽。

### 软件断点（突破 FPB 数量限制）

- ARM 陷阱指令是 **BKPT**（机器码 `0xBExx`），相当于 x86 的 0xCC。
- 思路：替换目标指令为 BKPT → 触发 **DebugMonitor 异常** → 芯片**自己**的 handler 接住 → 维护表存原指令。
- 数量只受内存限制 = 「无限断点」。
- 难点：**Flash 不好动态改写** → 让代码在 **RAM** 执行。
- 现实实现：**GDB stub**、Black Magic Probe、调试监控器。

---

## 4. 两套体系本质对比

> [!success] 核心一句话 GDB 是统一前端。Linux 借**操作系统**能力（ptrace+0xCC）；单片机借**芯片硬件**能力（SWD→DAP→FPB）。前端相同，后端是软件 vs 硬件两套。

> [!question] 没有 DAP 能在线看变量吗？ **不能。** Linux 靠 OS 读内存不需要额外硬件；单片机没有 OS，**DAP 是唯一的外部内存访问入口**，没有 DAP，GDB 命令传不进芯片，什么都看不到——除非芯片自跑 GDB stub 用软件实现。

---

## 5. 连接启动流程

### 单片机（两个角色：建立监听 + 连接）

```bash
# 终端1：启动 OpenOCD（GDB Server），建立监听
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg
# → Listening on port 3333 for gdb connections  ← 建立监听成功

# 终端2：启动 GDB，连接监听
arm-none-eabi-gdb firmware.elf
(gdb) target remote localhost:3333   # 连接3333端口
(gdb) load                           # 烧录到Flash
(gdb) break main
(gdb) continue
```

> [!note] 客户端-服务端架构
> 
> - **OpenOCD = 服务端**：建立监听，提供「访问芯片」服务（懂 SWD/DAP，不懂源码）
> - **GDB = 客户端**：连接服务端，发命令（懂 DWARF/源码，不懂硬件）
> - VS Code（Cortex-Debug）/ Keil 把这整套自动化成「一键调试」

### 为什么要 OpenOCD + GDB 两个

- **OpenOCD**：硬件层，操作 SWD/DAP/FPB，只懂「地址和字节」。
- **GDB**：源码层，靠 DWARF 把地址翻译成「变量名、类型、源码行」。
- 只烧录/看个地址 → OpenOCD 够；**源码级调试（看变量、按行断点、看调用栈）→ 必须 GDB**。

---
### 提取代码（汇编取栈指针 + C 处理）

```c
__attribute__((naked)) void HardFault_Handler(void) {
    __asm volatile (
        "tst lr, #4         \n"   // 测 EXC_RETURN 的 bit2
        "ite eq             \n"
        "mrseq r0, msp      \n"   // bit2=0：MSP 放进 r0
        "mrsne r0, psp      \n"   // bit2=1：PSP 放进 r0
        "b fault_handler_c  \n"   // 跳C函数，r0 即第一个参数
    );
}
void fault_handler_c(uint32_t *stack) {
    uint32_t stacked_pc = stack[6];  // 出错地址 → 拿去 addr2line
    uint32_t stacked_lr = stack[5];  // 从哪调用来
    while(1);
}
```

---

## 8. 配套工具链（围绕 ELF）

| 工具            | 干什么                         | 静/动 |
| ------------- | --------------------------- | --- |
| **objcopy**   | ELF → hex/bin               | 转换  |
| **objdump**   | 反汇编、看段（`-d` 汇编、`-S` 源码汇编对照） | 静态  |
| **addr2line** | 地址 → 代码行（靠 DWARF）           | 静态  |
| **readelf**   | 看 ELF 结构                    | 静态  |
| **nm**        | 看符号表                        | 静态  |
| **GDB**       | 运行时断点/看变量                   | 动态  |

```
.c → .o → 链接 ─┬─ objcopy → hex/bin（烧录）
                ├─ objdump  → 看汇编/段
                ├─ addr2line→ 地址定位行
                ├─ GDB      → 运行调试
                └─ .map     → 内存清单
```

---

## 11. 面试浓缩版（背这段）

> [!quote] 标准回答 GDB 是调试前端，底层分两套。Linux 上靠操作系统的 ptrace 系统调用读写进程内存，断点是把指令首字节换成 INT 3（0xCC）陷阱指令，内核接住 SIGTRAP 通知 GDB，是软件断点、数量不限。
> 
> 单片机没有操作系统，用不了 ptrace，靠 ARM 芯片内置的硬件调试单元：GDB 通过远程协议连 OpenOCD，经 SWD 两线接口访问芯片的 DAP 入口读内存，用 FPB 硬件单元设断点（M4 只有 6 个），DWT 做数据监视和周期计数，ITM 经 SWO 高速输出 trace。物理上电脑只有 USB、芯片只认 SWD，所以中间要 ST-Link/J-Link/DAPLink 做 USB 转 SWD。
> 
> 核心区别是 Linux 靠操作系统软件机制、单片机靠芯片硬件，GDB 是统一前端对接两套后端。这也是为什么单片机调试必须有 DAP——没有操作系统时，它是唯一的外部内存访问入口。
> 
> 编译产物上，ELF 是容器，里面的 DWARF（靠 -g 生成）让 GDB 认识变量名和行号；hex/bin 是瘦身的烧录文件，调试信息不进芯片。Hardfault 排查就是读 EXC_RETURN 判断 MSP/PSP，从栈偏移 0x18 取出 PC，再用 addr2line 靠 DWARF 翻译成代码行。
> 
> 理解到这一层，理论上我可以用一块单片机 bit-bang SWD 去调另一块——这就是 DAPLink 的原理。
