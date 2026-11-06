
# STM32F4 YMODEM IAP Bootloader

> 基于 STM32F4 实现标准 YMODEM 协议 IAP（In-Application Programming）Bootloader。
> 串口接收 `.bin` 固件写入片内 Flash，校验通过后自动跳转用户 APP。
> 非阻塞状态机 + DMA 环形队列架构，兼顾稳定性与可移植性。

## 📑 Table of Contents
- [✨ 核心特性](#-核心特性)
- [📁 项目结构](#-项目结构)
- [🔄 YMODEM 传输时序](#-ymodem-传输时序)
- [🧠 状态机设计](#-状态机设计)
- [⚙️ 配置说明](#️-配置说明)
- [🚀 快速上手](#-快速上手)
- [❓ FAQ 常见问题](#-faq-常见问题)
- [📌 待开发计划](#-待开发计划)

## ✨ 核心特性
- ✅ **标准 YMODEM 协议**：支持 128B / 1024B 双帧格式，CRC16 校验
- ✅ **非阻塞状态机**：主循环轮询调用，无阻塞等待，不占用长中断
- ✅ **DMA + 环形缓冲区**：串口 DMA 接收，协议解析与硬件接收解耦
- ✅ **Flash 安全写入**：按扇区擦除、32bit 字对齐写入、写入后回读校验
- ✅ **超时重传机制**：控制帧超时自动重发，最大重试次数可配置
- ✅ **APP 安全跳转**：跳转前关闭外设/DMA/中断、复位 RCC、重定向向量表
- ✅ **上位机兼容**：SecureCRT、Xshell、TeraTerm 等支持 YMODEM 的终端工具

## 📁 项目结构
```
.
├── Ymodem/
│   ├── Src/
│   │   ├── ymodem.c            # YMODEM 接收状态机主逻辑
│   │   ├── ymodem_port.c       # 移植层：串口收发、超时底层接口
│   │   ├── ymodem_crc.c        # CRC16 校验算法
│   │   └── ymodem_flash.c      # Flash 擦写、APP 跳转逻辑
│   └── Inc/
│       ├── ymodem.h
│       ├── ymodem_port.h
│       ├── ymodem_crc.h
│       └── ymodem_flash.h
└── ...
```

## 🔄 YMODEM 传输时序
```
PC 发送方                    STM32 接收方
     |                             |
     | <------------- C -----------|  就绪，等待文件信息帧
     |                             |
     | -------- 文件信息帧 -------> |
     | <------------- ACK ---------|
     | <-------------- C ----------|  准备接收数据帧
     |                             |
     | -------- 数据帧 1 ---------> |
     | <------------ ACK ----------|
     |                             |
     | -------- 数据帧 N ---------> |
     | <------------ ACK ----------|
     |                             |
     | ----------- EOT -----------> |  第一次 EOT
     | <----------- NAK -----------|
     |                             |
     | ----------- EOT -----------> |  第二次 EOT
     | <----------- ACK -----------|
     | <------------- C -----------|
     |                             |
     | ------ 空文件信息帧 -------> |  传输结束标记
     | <----------- ACK -----------|
```

## 🧠 状态机设计
`Ymodem_Receive()` 为主入口函数，在主循环中周期性非阻塞调用，根据 `rx_state` 分流处理。

| 状态枚举 | 说明 |
| ---- | ---- |
| YMODEM_RX_WAIT_HEADER | 等待文件信息帧 |
| YMODEM_RX_WAIT_DATA | 等待固件数据帧 |
| YMODEM_RX_WAIT_EOT | 等待第一次 EOT |
| YMODEM_RX_WAIT_EOT_SECOND | 等待第二次 EOT |
| YMODEM_RX_WAIT_END | 等待空文件信息结束帧 |
| YMODEM_RX_FINISH | 固件接收完成 |
| YMODEM_RX_ERROR | 接收异常（校验失败、超时等） |
| YMODEM_RX_CACEL | 上位机主动取消传输 |

### 调用链路
```
main()
  ├── Ymodem_ReceiveInit()
  └── while (1)
        └── Ymodem_Receive()
              ├── Ymodem_HandleWaitHeader()
              ├── Ymodem_HandleWaitData()
              ├── Ymodem_HandleWaitEot()
              ├── Ymodem_HandleWaitEotSecond()
              └── Ymodem_HandleWaitEndPacket()
                    │
                    ├── Ymodem_ExtractPacket()   // 从环形队列提取完整帧
                    ├── Ymodem_CheckDataPacket() // 帧序号、CRC校验
                    ├── Ymodem_ParseFileInfo()   // 解析文件名、固件长度
                    ├── Ymodem_SendByte()        // 下发 ACK / NAK / C
                    └── FLASH_If_Write()         // Flash 写入
```

## ⚙️ 配置说明
宏定义集中在 `ymodem_port.h`、`ymodem_flash.h`，按需修改：

| 宏名 | 默认值 | 说明 |
| ---- | ---- | ---- |
| YMODEM_COM | huart1 | 绑定使用的串口句柄 |
| YMODEM_RETRY_MAX | 10 | 单帧最大重试次数 |
| YMODEM_TIMEOUT_MS | 1000 | 帧接收超时时间(ms) |
| FILE_APP_SIZE_MAX | 0x10000 | APP 固件最大长度（64KB） |
| APPLICATION_ADDRESS | 0x08010000 | APP 运行起始地址，Bootloader 占用前 64KB |
| USER_FLASH_END_ADDRESS | 0x08100000 | 片内 Flash 结束地址（1MB） |

> ⚠️ **重要提醒**：修改 APP 分区大小时，需要同步更新：`FILE_APP_SIZE_MAX`、APP 工程链接脚本地址、`VECT_TAB_OFFSET`。

## 🚀 快速上手
1. 编译 Bootloader，烧录至 `0x08000000`
2. 配置 APP 工程：链接起始地址 `0x08010000`，`VECT_TAB_OFFSET = 0x10000`
3. APP 编译输出 `.bin` 固件（不要使用 .elf / .hex）
4. 串口连接开发板与 PC，打开 SecureCRT / Xshell
5. 复位开发板进入 Bootloader，终端选择 YMODEM 发送，选中 bin 文件
6. 传输校验完成后，Bootloader 自动跳转运行 APP

## ❓ FAQ 常见问题
### Q1：传输一开始上位机不发送文件，没有任何反应
A：确认 Bootloader 持续发送 `C`；检查串口波特率、奇偶校验、停止位，上下必须一致；
不要开启串口流控（RTS/CTS）。

### Q2：可以收到文件帧，但是频繁报 CRC 错误、丢包
A：
1. DMA 环形队列长度不足，适当加大环 buffer；
2. 串口波特率过高，优先测试 115200；
3. 检查接线，TX/RX 交叉，共地；
4. 确认上位机 YMODEM 模式（标准 YMODEM，不是 Y-Modem-G）。

### Q3：固件接收成功，但是跳转后 APP 死机、无响应
A：高频踩坑点：
1. APP 没有设置 `SCB->VTOR` 向量表偏移；
2. APP 工程链接脚本地址、偏移量和 Bootloader 配置不一致；
3. APP 里没有重新初始化时钟、SysTick；
4. Bootloader 跳转前没有关闭全局中断、DMA、串口外设。

### Q4：为什么必须传输结束后发空文件帧？
A：YMODEM 协议规范，空文件信息包作为传输结束标识，用于通知接收方本次传输会话结束。

### Q5：支持大于 1MB Flash 的芯片吗？
A：本项目默认配置 STM32F4 1MB Flash，修改 `USER_FLASH_END_ADDRESS` 即可适配更大 Flash，注意扇区编号和擦除函数。

### Q6：可以移植到 STM32F1 / F7 / H7 吗？
A：协议层（ymodem.c、ymodem_crc.c）无平台依赖；只需要重新实现 `ymodem_port.c`（串口收发、超时）和 `ymodem_flash.c`（Flash 操作）。

## 📌 待开发计划
- [ ] YMODEM 上行发送（设备 → PC 上传固件）
- [ ] 多文件传输支持
- [ ] 固件安全校验：CRC32 / SHA256 签名验证
- [ ] 双 Bank 分区升级 + 固件回滚机制
- [ ] 实时升级进度上报
