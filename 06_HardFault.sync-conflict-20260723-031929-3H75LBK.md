

> 本笔记按两条架构线索展开：
> 
> - **Mainline**：Cortex-M3 / M4 / M7（Armv7-M）—— Fault体系完整，诊断信息丰富
> - **Baseline**：Cortex-M0 / M0+（Armv6-M）、M23（Armv8-M Baseline）—— 一切fault直进HardFault
> 
> 结合项目：GD32E230（M23，Formlabs测试治具）属于Baseline；RoboMaster常用的STM32F4属于Mainline。

---

# 1 共同基础（两种架构一致的部分）

## 1.1 异常 vs 中断

ARM术语体系中，**异常（Exception）是大概念，中断（Interrupt/IRQ）是异常的子集**：

- 异常编号 1–15：系统异常（Reset、NMI、HardFault、SVC、PendSV、SysTick…）
- 异常编号 ≥16：外设中断 IRQ，由NVIC管理
- 换算关系：`IRQn = 异常编号 - 16`，所以CMSIS里系统异常的IRQn是负数（SysTick_IRQn = -1，PendSV_IRQn = -2）
- 压栈、取向量、EXC_RETURN返回机制，两者完全相同

### 1.1.1 向量表布局（架构硬性规定，所有Cortex-M一致）

| 偏移    | 内容                       | 异常编号  | 优先级         |
| ----- | ------------------------ | ----- | ----------- |
| 0x00  | **MSP初始值**（不是异常入口！）      | —     | —           |
| 0x04  | Reset_Handler            | 1     | -3（固定，最高）   |
| 0x08  | NMI_Handler              | 2     | -2（固定，不可屏蔽） |
| 0x0C  | **HardFault_Handler**    | 3     | **-1（固定）**  |
| 0x10  | MemManage（仅Mainline有独立项） | 4     | 可配置         |
| 0x14  | BusFault（仅Mainline）      | 5     | 可配置         |
| 0x18  | UsageFault（仅Mainline）    | 6     | 可配置         |
| ...   | SVC / PendSV / SysTick 等 | 11–15 | 可配置         |
| 0x40起 | IRQ0, IRQ1, ...          | 16起   | 可配置         |

复位流程：硬件从0x00读MSP初始值装入MSP → 从0x04读Reset入口跳转执行。

注意区分**异常编号**和**优先级**：编号只是表内位置，优先级决定抢占关系。IAP跳APP前重定位SCB->VTOR，本质是告诉CPU新表的基址，表内布局不变。

## 1.2 硬件自动压栈的栈帧

异常发生时，硬件把8个word压入**当时正在使用的栈**（MSP或PSP）：

```
[SP+0]  R0
[SP+4]  R1
[SP+8]  R2
[SP+12] R3
[SP+16] R12
[SP+20] LR   ← 调用者返回地址
[SP+24] PC   ← 出错指令地址（核心信息！）
[SP+28] xPSR
```

用哪个栈看EXC_RETURN（进handler时LR的值）的**bit2**：0=MSP，1=PSP。

| EXC_RETURN      | 含义                                                           |
| --------------- | ------------------------------------------------------------ |
| 0xFFFFFFF1      | Handler模式发生fault（中断嵌套中出错）→ 栈帧必在MSP                           |
| 0xFFFFFFF9      | 线程模式，栈帧在MSP                                                  |
| 0xFFFFFFFD      | 线程模式，栈帧在PSP                                                  |
| 0xFFFFFFE9 / ED | （M4F/M7带FPU）lazy stacking扩展帧，多压S0-S15+FPSCR，但前8字顺序不变，PC仍在+24 |

---

# 2 M3 / M4 / M7 的完整Fault体系

## 2.1 三个可配置Fault + 升级机制

| Fault      | 触发场景          | 使能位（SHCSR, 0xE000ED24） |
| ---------- | ------------- | ---------------------- |
| MemManage  | 违反MPU权限       | MEMFAULTENA            |
| BusFault   | 总线访问错误        | BUSFAULTENA            |
| UsageFault | 未定义指令、除0、非对齐等 | USGFAULTENA            |

这就是HardFault固定-1优先级的原因：保证系统任何状态下都有兜底。

### 2.1.1 诊断寄存器

CFSR三段 = 三类fault：

- **MM**（内存权限）：违反MPU权限，如踩了栈guard region
- **BF**（总线）：访问不存在/不响应的地址，典型：外设时钟没开就读寄存器
- **UF**（用法）：指令本身有问题——未定义指令、除0、没FPU用浮点

---

# 3 Baseline：M0 / M0+ / M23 的"极简"Fault体系

## 3.1 核心差异：只有HardFault

- **没有**独立的MemManage / BusFault / UsageFault 异常
- **没有** CFSR / MMFAR / BFAR —— 无精细分类，无出错地址寄存器
- 空指针、非法指令、总线错误……**一切直接进HardFault**
- 诊断基本只剩一条路：**栈上取PC → 反汇编/.map 反查**

## 3.2 排查流程

1. 停在handler入口，看LR末位（9/D/1）判断栈
2. 读对应MSP/PSP，取栈帧+24处的PC
3. 反汇编/.map/addr2line反查源码


---

## 3.3 定位出错代码实操：Keil vs GCC

### 3.3.1 通用步骤

1. LR末位：`9`→MSP，`D`→PSP，`1`→Handler模式出错、必在MSP
2. 读对应栈指针指向的8个word：R0-R3, R12, LR, PC, xPSR

### 3.3.2 Keil (MDK)

1. 调试模式 → `Peripherals → Core Peripherals → Fault Reports`：自动解码CFSR/UFSR/MMSR/BFSR
2. Register窗口看LR → Memory窗口输入MSP/PSP值看栈帧
3. 拿到PC → 反汇编窗口右键 `Show Code at Address` 输入PC跳转
### 3.3.3 GCC + GDB

```gdb
# 停在HardFault_Handler后（断点，或死循环时Ctrl+C）
p/x $lr                 # EXC_RETURN末位：9=MSP，D=PSP
x/8xw $msp              # 或 x/8xw $psp，8字=R0-R3,R12,LR,PC,xPSR
                        # 第7个字（+24）= 出错PC

# 假设PC = 0x08001234：
list *0x08001234        # 直接显示源码行（最常用）
info line *0x08001234   # 函数名 + 文件:行号
disassemble 0x08001234  # 反汇编上下文
```

`$msp`/`$psp`可用性取决于GDB server（OpenOCD、J-Link均支持）；不支持时`p/x $sp`——刚进handler未压栈时SP即MSP。

### 3.3.4 无调试器的离线定位（产线/现场）

固件把栈帧PC串口打印或写Flash留痕，然后：

```bash
arm-none-eabi-addr2line -e firmware.elf -f 0x08001234
# 输出：函数名 + 源文件:行号
```

适合批量部署设备远程排障（Formlabs测试治具500+台的场景），思路与W25Q128飞行黑盒同源：**故障后留证据**。

---

## 3.4 常见触发场景

- 空指针解引用（访问0地址附近未映射区域）
- 数组越界 / 野指针写飞，破坏变量或返回地址
- **栈溢出**
- 函数指针跳到非法地址
- 执行未定义指令
- 无FPU用浮点指令（Mainline报NOCP；Baseline直接HardFault）
- 非对齐访问 / 除0