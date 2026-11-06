STM32 YMODEM Bootloader
基于 STM32F4 的 YMODEM 协议 IAP（In-Application Programming）Bootloader，支持通过串口接收 .bin 固件、写入内部 Flash，并跳转执行用户 APP。
✨ 特性
标准 YMODEM 协议：支持 128B / 1024B 数据帧，CRC16 校验

状态机驱动：非阻塞轮询式接收，主循环中调用，不占用中断

环形队列 + DMA：串口 DMA 接收，数据入环形队列，解耦接收与协议处理

Flash 安全写入：按扇区擦除、32 位字对齐写入、回读校验

超时重传：控制字符超时自动重发，最大重试次数可配

跳转前环境清理：关闭外设/DMA/中断、复位 RCC、切换向量表，保证 APP 干净启动

上位机兼容：适配 SecureCRT 等常见 YMODEM 发送工具
