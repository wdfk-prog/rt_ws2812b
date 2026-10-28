# WS2812B SPI 灯带软件包

本软件包通过 RT-Thread SPI 设备输出 WS2812B 的 GRB 串行码，支持多路灯带、静态或动态内存管理、可配置 SPI 波形编码、可选 Gamma 校正及可注册灯效。软件包不负责板级 SPI/CS/GPIO 初始化，也不创建业务线程。

## 文档

- [WS2812B 单线协议与 SPI 波形编码原理](doc/ws2812b-spi-theory.md)：说明协议时序、GRB 位序、nibble LUT 与 RESET 帧计算。

## 安装和初始化

在 RT-Thread `menuconfig` 中开启 `PKG_USING_WS2812B`（依赖 SPI 设备框架）。通过 `WS2812B_KCONFIG_SLOT_COUNT` 选择可见的设备槽数量，逐路设置 SPI bus、device、灯珠数、SCK、RESET 与编码时序。`WS2812B_MAX_DEVICES` 表示设备注册表总容量，必须覆盖已启用槽位以及应用手工创建的设备。

板级 SPI 设备可以预先由 BSP 注册，也可以在调用 `ws2812_init_all()` 前通过 `ws2812_set_kconfig_attach(slot, callback, user_data)` 为槽位提供 attach 回调。创建后可使用 `ws2812_set_pixel()` / `ws2812_fill()` 修改颜色，使用 `ws2812_send_frame()` 发送完整编码帧；调用方负责检查所有 API 返回值。

原有五个入口 `ws2812_create()`、`ws2812_clear_buff()`、`ws2812_write_rgb_to_node()`、`ws2812_write_rgb_to_all()` 和 `ws2812_send()` 保留名称、参数与返回类型。原接口创建时仍使用 SPI mode 3、6 MHz、8 个 SPI clocks/LED bit 的默认时序；扩展设备结构体不得沿用旧示例中直接 `rt_free` 设备指针的写法。

设备注册和 SPI 发送必须在允许使用相关 RT-Thread API 的上下文执行。同一灯带的像素数据由应用自行串行化访问，SPI 总线互斥不能替代像素缓冲区保护。软件包不提供设备运行时注销和删除 API。

## 内存配置

`WS2812B_MEMORY_STATIC` 使用固定设备对象、RGB 像素池和 SPI 帧池，不依赖 RT-Thread Heap。手动创建静态设备时应通过 `WS2812B_STATIC_EXTRA_PIXELS` 与 `WS2812B_STATIC_EXTRA_FRAME_BYTES` 预留容量。`WS2812B_MEMORY_DYNAMIC` 为所有设备使用 Heap，依赖 `RT_USING_HEAP`。

设 N 为灯珠数、C 为一个 WS2812 逻辑 bit 的 SPI clock 数量、F 为实际 SCK（Hz）、R 为 RESET 时间（μs）。原始 RGB888 像素缓存占 `4*N` 字节，编码数据占 `3*N*C` 字节，包含前后低电平的整帧长度是 `3*N*C + 2*ceil(F*R/8000000)` 字节。

静态模式的 `WS2812B_DEVn_FRAME_BYTES=0` 会基于 `WS2812B_STATIC_MAX_CLOCKS_PER_BIT` 自动估算容量；自定义容量不能小于实际帧需求。`WS2812B_CHECK_PARAMETERS` 控制部分运行时边界检查；关闭后由调用方保证设备和内存容量正确。

## SPI 编码与时序

发送数据按每颗灯珠 G7..G0、R7..R0、B7..B0 排列。默认 `WS2812B_TIMING_DIRECT` 使用 Kconfig 时序参数；`WS2812B_TIMING_TABLE` 可以查预设配置，`WS2812B_TIMING_SEARCH` 在预设不匹配时尝试整数搜索。

默认 `WS2812B_ENCODER_FAST_ONLY` 使用 16 项四位 nibble LUT 编码（要求每逻辑 bit 的 SPI clock 数为偶数）；`WS2812B_ENCODER_AUTO` 允许通用逐位编码回退。`WS2812B_ENCODER_SELF_CHECK` 可选初始化等价性自检。实际 MOSI 时序必须与真实 SCK 匹配，并在目标板上确认复位低电平、SPI 片选及 DMA 传输完成语义。

## Gamma 与亮度

亮度先对 RGB 各通道进行整数缩放，然后应用可选 Gamma。`WS2812B_GAMMA_22` 和 `WS2812B_GAMMA_28` 使用静态查表；开启 `WS2812B_GAMMA_COMPUTE` 后可用 `ws2812_set_gamma_custom(dev, gamma_x100)` 根据 50～500 的指数×100 生成每设备 LUT，该配置需要链接 `libm`。默认 Gamma 为 `WS2812_GAMMA_NONE`。

## 灯效框架

在 `menuconfig` 开启 `WS2812B_USING_EFFECT` 后，`SConscript` 只编译对应启用的内置方法：`SOLID`、`BLINK`、`BREATHE`、`GRADIENT`、`GRADATION`、`CHASE` 和 `SEQUENCE`。每路灯带有独立的效果状态；只有一个当前效果，业务负责告警临时效果的切换与恢复。

初始化顺序为：板级 SPI 设备准备好后，调用 `ws2812_effect_register_builtins()` 注册方法，然后 `ws2812_init_all()` 创建设备，再 `ws2812_effect_init_kconfig()` 应用各槽默认效果。应用自己的任务或事件循环定期调用 `ws2812_effect_poll_all(rt_tick_get())`；软件包不自动启动任务或轮询。更改灯效与轮询必须由应用在同一个受控上下文中串行执行。

内置灯效的 ID 由当前编译启用的枚举项连续确定；Kconfig 的初始单选菜单借助隐藏 `WS2812B_DEVn_INITIAL_KIND` 编译期转换到当前 ID，无需运行时查找。每个设备槽可以选择 `No initial effect`，应用之后再通过 API 选择灯效。`SEQUENCE` 的步骤在 `ws2812_effect_start_sequence()` 返回前复制到设备内部存储，不支持将其作为 Kconfig 默认方法。

`BLINK` 的完整周期需为不小于 2 的偶数 Tick。Kconfig 默认 BLINK 使用奇数周期时会触发 C99 编译期数组长度断言；启用 `WS2812B_CHECK_PARAMETERS` 后，运行时奇数周期返回 `-RT_EINVAL`。按需通过 `WS2812B_EFFECT_CUSTOM_SLOTS` 预留自定义效果方法 ID 槽。

## 接入边界

`ws2812_init_all()` 和 SPI attach 回调由业务启动代码调用；软件包没有内部 worker 或定时任务。可选择开启 `WS2812B_CHECK_EXTENDED` 在初始化阶段检查 LED 版本对应的物理时序。真实 SPI 分频、MOSI 空闲电平、信号电平裕量及供电应在实际 BSP 和硬件上验证。
