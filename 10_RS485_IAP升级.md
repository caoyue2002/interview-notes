# GD32E230 RS485 多机 IAP 升级设计（复习笔记）

芯片 GD32E230C8：Cortex-M23，片内 Flash 64 KB，SRAM 8 KB，Flash 页大小 1 KB。目标是在一条 RS485 总线上，一次升级最多 128 个同型号设备。

## 1. Flash 分区

```text
0x08000000 ~ 0x08003BFF    Bootloader，15 KB
0x08003C00 ~ 0x08003FFF    参数/信息区，1 KB
0x08004000 ~ 0x0800FFFF    App，48 KB
```

- Bootloader 上电先跑，负责通信、擦写 App、校验和跳转。
- 参数区单独占 1 页，用来保存 App 状态、版本、大小和 CRC。
- 写固件数据时只能擦写 App 区；更新状态时只能擦写参数页；不能碰 Bootloader 自身。

## 2. 参数区

参数区保存掉电后仍要保留的信息：

```c
typedef struct {
    uint32_t magic;
    uint32_t state;       // EMPTY / UPDATING / VALID / INVALID
    uint32_t version;
    uint32_t app_size;
    uint32_t app_crc32;
} dev_info_t;
```


## 3. App 工程修改

App 不能再从 `0x08000000` 链接，要改到 `0x08004000`：

```text
IROM1 Start: 0x08004000
IROM1 Size : 0x0000C000
```

同时把向量表偏移改成：

```c
#define VECT_TAB_OFFSET  ((uint32_t)0x4000)
```

## 4. 通信协议

RS485 是半双工总线，原则是：**广播只接收不回复，只有被点名的设备才允许回包**。

主机下发包：

```text
magic      2 字节    0x55AA
cmd        1 字节    PING / START / DATA / END / QUERY_ACK / QUERY_RESULT
target_id  1 字节    0=广播，1~128=单播
seq        2 字节    包序号
addr       4 字节    DATA 写入地址
len        2 字节    payload 长度
crc16      2 字节    覆盖 header 关键字段 + payload
payload    N 字节
```

命令含义：

```text
PING          单播探测设备是否在 Bootloader，回 PONG 后准备接收 DATA
START         广播开始升级，设备进入 UPDATING
DATA          广播固件数据，带 seq/addr/len/payload
END           广播 app_size + app_crc32，触发整包 CRC 校验
QUERY_ACK     单播查询某台设备对某个 seq 的处理结果
QUERY_RESULT  单播查询某台设备最终 VALID/INVALID
```

回复包统一表示 PONG、ACK 或最终结果：

```text
0xA5 0x5A  id  seq  status  crc16
```

常用 `status`：

```text
OK / CRC_ERR / ADDR_ERR / FLASH_ERR / SEQ_ERR / VALID / INVALID
```

## 5. 升级触发

设备平时运行 App。上位机要升级时，广播 20 次：

```text
升级触发命令 + 目标版本号
```

App 收到后比较版本：

- 目标版本更高：直接 `NVIC_SystemReset()`，进入 Bootloader 的升级等待窗口。
- 目标版本不高：忽略，不进入升级流程。

Bootloader 每次上电/复位后先等待一段时间：

```text
如果 App 有效：
    1 分钟内收到升级相关命令 → 继续升级
    1 分钟内没有后续命令   → 跳转 App
如果 App 无效或 state 不是 VALID：
    留在 Bootloader，等待重刷
```

## 6. 总流程

```text
1. Host 广播 20 次“升级触发命令 + 目标版本”
   App 收到后比较版本：
   - 目标版本更高：软复位进 Bootloader
   - 目标版本不高：忽略

   进 Bootloader 后，如果收到 START：
   - state = UPDATING
   - 准备接收 DATA

   如果超过 1 分钟没有后续升级命令，且原 App 有效，就跳回 App。

2. 设备发现
   Host 逐个 PING 1~128。
   设备被点名后回 PONG，Host 建 online_list，并且擦除 App Flash

3. 发送固件
   Host 广播 DATA(seq, addr, len, payload)。
   设备写 Flash，并记录该 seq 的处理状态。

   Host 再逐个发 QUERY_ACK：
   被点名设备回 ACK(seq, OK/ERR)。
   某设备超时或错误，则 Host 重发该 DATA 帧。

4. 结束校验
   Host 广播 END(app_size, app_crc32)。
   设备按 app_size 计算 App CRC：
   - CRC 一致：state=VALID，记录 app_size/app_crc32
   - CRC 不一致：state=INVALID

5. 查询结果
   Host 逐个 QUERY_RESULT。
   VALID 的设备跳 App。
   INVALID 或无响应的设备留在 Bootloader。

6. 重复第 1 步
   Host 再广播 20 次“升级触发命令 + 目标版本”，然后 PING 一轮。
   已升级成功的设备版本一致，不会进入 Bootloader，也不会 PONG。
   升级失败的设备仍留在 Bootloader，会 PONG。
   如果 PONG = 0，说明全部升级完成。
```

## 7. Bootloader 接收：DMA + IDLE

帧变长（START/PING 的 payload=0，DATA 一般 512，END 带 `app_size/app_crc32`），用 **DMA + 串口 IDLE 中断**收整包：DMA 搬进缓冲区，总线空闲触发 IDLE，一包收完置标志，主循环再处理。缓冲区要大于最大包长，设 1024。收到包先验 CRC，再看 `target_id` 是不是广播或本机 ID，不是就丢弃。

```c
#define RX_BUF_SIZE  1024
static uint8_t  rx_buf[RX_BUF_SIZE];
static volatile uint8_t  frame_ready = 0;
static volatile uint16_t frame_len   = 0;

void USART0_IRQHandler(void)
{
    if (usart_interrupt_flag_get(USART0, USART_INT_FLAG_IDLE)) {
        usart_data_read(USART0);                 // 读 DATA 清 IDLE 标志
        dma_channel_disable(DMA0, DMA_CH);
        frame_len = RX_BUF_SIZE - dma_transfer_number_get(DMA0, DMA_CH);
        frame_ready = 1;
    }
}

int main(void)
{
    systick_config();
    boot_uart_init();          // 含 DMA 接收 + 使能 USART IDLE 中断
    dma_recv_start();

    while (1) {
        if (frame_ready) {
            frame_ready = 0;

            if (parse_packet(rx_buf, frame_len, &pkt) &&
                (pkt.target_id == 0 || pkt.target_id == g_my_id)) {
                if (pkt.cmd == CMD_START) {
                    mark_updating();             // state = UPDATING
                    remember_status(pkt.seq, ST_OK);
                } else if (pkt.cmd == CMD_PING) {
                    send_pong();                 // 单播 PING 才回 PONG
                    mark_updating();
                    erase_app_area();            // 按当前流程：PONG 后准备接收 DATA
                } else if (pkt.cmd == CMD_DATA) {
                    check_seq();                 // 序号连续，重复包幂等处理
                    check_crc();                 // CRC 覆盖 header + payload
                    check_addr();                // 地址在 App 区
                    write_flash();
                    remember_status(pkt.seq, ST_OK);
                } else if (pkt.cmd == CMD_END) {
                    parse_end_params(&pkt, &app_size, &app_crc32);
                    if (whole_crc_ok(app_size, app_crc32)) mark_valid();
                    else                                  mark_invalid();
                    remember_status(pkt.seq, dev_info.state);
                } else if (pkt.cmd == CMD_QUERY_ACK) {
                    send_ack_status(pkt.seq);    // 点名后才回 ACK，避免总线冲突
                } else if (pkt.cmd == CMD_QUERY_RESULT) {
                    send_result();               // 单播回 VALID/INVALID
                }
            }

            dma_recv_start();                    // 处理完必须重启 DMA
        }

        // 加超时：距上次收包超阈值则退出升级等待或报错
    }
}
```

要点：IDLE 中断只置标志，校验和写 Flash 放主循环；每包处理完必须重启 DMA；重复包要能幂等，因为 ACK 丢了上位机会重发。

## 8. Flash 擦除与写入

擦除只按 App 区页地址循环，绝不整片擦；写入后回读校验；地址检查拒绝 App 区外和非 4 字节对齐。

```c
static uint8_t app_addr_valid(uint32_t addr, uint32_t len)
{
    if (addr < APP_BASE_ADDR)            return 0;
    if ((addr + len) > FLASH_END_ADDR)   return 0;
    if ((addr & 0x03U) != 0U)            return 0;
    return 1;
}

static uint8_t flash_write(uint32_t addr, uint8_t *data, uint32_t len)
{
    if (!app_addr_valid(addr, len)) return 0;

    fmc_unlock();
    for (uint32_t i = 0; i < len; i += 4) {
        uint32_t word = 0xFFFFFFFFU;
        for (uint8_t j = 0; j < 4; j++) {
            if ((i + j) < len) {
                word &= ~(0xFFUL << (j * 8));
                word |=  ((uint32_t)data[i + j] << (j * 8));
            }
        }

        if (fmc_word_program(addr + i, word) != FMC_READY ||
            *(volatile uint32_t *)(addr + i) != word) {
            fmc_lock();
            return 0;
        }
    }
    fmc_lock();
    return 1;
}
```

最后一包不足 512 字节时按实际长度写。

## 9. App 有效性判断 + 跳转 App

跳转前检查向量表前两个字：第一个字（初始 MSP）落在 SRAM，第二个字（Reset_Handler）落在 App Flash。跳转前必须先关掉 Bootloader 用的 DMA、USART、IDLE 中断，否则 App 会继承一个还在跑的 DMA。

```c
typedef void (*pFunction)(void);

static uint8_t app_is_valid(void)
{
    uint32_t sp    = *(volatile uint32_t *)APP_BASE_ADDR;
    uint32_t reset = *(volatile uint32_t *)(APP_BASE_ADDR + 4U);

    if (sp    < SRAM_BASE_ADDR || sp    > SRAM_END_ADDR)   return 0;
    if (reset < APP_BASE_ADDR  || reset >= FLASH_END_ADDR) return 0;
    return 1;
}

static void jump_to_app(void)
{
    uint32_t sp    = *(volatile uint32_t *)APP_BASE_ADDR;
    uint32_t reset = *(volatile uint32_t *)(APP_BASE_ADDR + 4U);

    __disable_irq();
    dma_channel_disable(DMA0, DMA_CH);
    usart_disable(USART0);
    SysTick->CTRL = 0;
    SysTick->LOAD = 0;
    SysTick->VAL  = 0;

    for (uint32_t i = 0; i < 8; i++) {
        NVIC->ICER[i] = 0xFFFFFFFFU;
        NVIC->ICPR[i] = 0xFFFFFFFFU;
    }

    SCB->VTOR = APP_BASE_ADDR;
    __DSB();
    __ISB();
    __set_MSP(sp);
    ((pFunction)reset)();
}
```

`((pFunction)reset)();` 就是把 App 的 Reset_Handler 地址强转成函数指针并调用，等价于跳到 App 运行。`rx_buf` 是 Bootloader 私有缓冲区，跳 App 前关键是停掉指向它的 DMA；进 App 后这块 RAM 会被 App 启动代码重新初始化。

## 10. 关键保护点

- **不能广播后同时回复**：DATA、START、END 都可以广播，但 ACK 必须通过 `QUERY_ACK` 点名收。
- **CRC 要覆盖 header + payload**：防止 `addr/len/seq` 被干扰后仍写入错误位置。
- **重复包要幂等**：同一 `seq/addr` 已写过且回读一致，就记录成功，不重复编程 Flash。
- **掉电保护靠状态机**：升级中掉电会停在 `UPDATING/INVALID`，下次不跳残缺 App。
- **END 做最终兜底**：漏包、错包、Flash 写失败，最后都会被整包 CRC 抓出来。
- **跳 App 前要清理外设**：关闭 DMA、USART、SysTick 和中断，设置 `SCB->VTOR=APP_BASE_ADDR`，再设置 MSP 并跳 Reset_Handler。

## 11. 面试怎么讲

> RS485 半双工，我让上位机做广播主控。升级前先广播 20 次升级触发命令和目标版本，App 比对版本，只有新版本才软复位进 Bootloader。进 Bootloader 后收到 START，就写 state=UPDATING；如果 1 分钟没后续且原 App 有效，就跳回 App。然后主机逐个 PING 1~128，能 PONG 的就是还在 Bootloader 的设备，加入 online_list，并擦 App 区准备收 DATA。之后广播 DATA 写固件，但 ACK 不让设备同时回，而是主机用 QUERY_ACK 逐个点名查询。最后广播 END(app_size, app_crc32)，设备算整包 CRC，写 VALID 或 INVALID；主机 QUERY_RESULT 查结果，VALID 跳 App，失败的留在 Bootloader。下一轮再广播 20 次触发命令并 PING，如果 PONG=0，就说明全部升级完成。
