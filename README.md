# WS2812B SPI 灯带软件包

本软件包通过 RT-Thread SPI 设备发送 WS2812B 的 GRB 串行码，支持多灯带、静态或动态内存、可选 Gamma、可注册灯效及 Tick 轮询接口。

## 文档导航

- [软件包架构与数据流](doc/architecture.md)：源文件职责、生命周期、编码/效果调用链与 RAM 预算。
- [Kconfig 配置与移植](doc/configuration.md)：静态/动态内存、直接时序、编译条件及板级约束。
- [WS2812B 单线协议与 SPI 波形编码原理](doc/ws2812b-spi-theory.md)：独立原理文章，解释波形、GRB、nibble LUT 和 RESET 计算，并提供原始资料链接。

## 安装和初始化

通过 `menuconfig` 启用 `PKG_USING_WS2812B`。三个可选设备槽由 `WS2812B_KCONFIG_SLOT_COUNT`（1～3）控制可见数量，逐路提供 SPI bus/device、灯珠数、SCK、协议版本、RESET 和编码时序参数。`WS2812B_MAX_DEVICES` 是注册表总容量，不是 Kconfig 槽数；Kconfig 会将其下限限制为**实际启用的设备槽数量**（支持跳过中间槽），与 `WS2812B_CHECK_PARAMETERS` 是否开启无关。手动通过 C API 创建设备时仍需额外预留注册表容量；绕过 Kconfig 直接提供不一致的编译宏也会触发编译期 `#error`。

初始化顺序：先安装每槽独立的 SPI attach 回调（如果设备已由 BSP 注册则无需回调）；启用灯效时先注册已编译的方法，然后创建设备并选择 Kconfig 默认灯效。软件包不创建线程，轮询时机由应用决定。

以下示例使用 `WS2812B_USING_KCONFIG_DEVICES=y`、`WS2812B_KCONFIG_SLOT_COUNT=3` 和 `WS2812B_USING_EFFECT=y`；按实际可见槽位设置 attach 回调。**`WS2812B_USING_EFFECT` 默认为 `n`**；关闭时 `SConscript` 不编译灯效实现，不能调用 `ws2812_effect_*` API。只使用基础像素控制时，保留设备 attach 和 `ws2812_init_all()`，省略示例中的灯效注册、初始化及轮询调用，改由应用调用像素写入与发送 API。

```c
ws2812_set_kconfig_attach(0, board_spi_attach, RT_NULL);
ws2812_set_kconfig_attach(1, board_spi_attach, RT_NULL);
ws2812_set_kconfig_attach(2, board_spi_attach, RT_NULL);
ws2812_effect_register_builtins();
ws2812_init_all();
ws2812_effect_init_kconfig();
/* Inside an application-owned LED task or event loop: */
ws2812_effect_poll_all(rt_tick_get());
```

`ws2812_create(char *, uint16_t)`、`ws2812_clear_buff()`、`ws2812_write_rgb_to_node()`、`ws2812_write_rgb_to_all()`、`ws2812_send()` 的名称、返回类型和参数类型保持原有定义。旧 `create` 使用 SPI mode 3、6 MHz、8 时钟/LED 位、0/1 分别为 C0/F0。旧设备结构体前三个成员保持原位置，但不得对扩展结构体使用旧版本的手工 `rt_free` 方式。

设备及灯效方法都应只在初始化阶段注册；软件包不提供设备注销/删除接口。运行时只改颜色、Gamma、亮度、灯效选择和状态。**不允许多个线程同时修改同一灯带缓冲区**；SPI 总线互斥不保护应用层像素与灯效数据。

## 运行时检查

`WS2812B_CHECK_PARAMETERS=y` 检查公开参数、设备和数组容量、帧大小等软件边界；关闭后调用方必须保证所有名称、数量、时序和存储容量正确，否则存在越界或非法访问风险。启用槽位数与 `WS2812B_MAX_DEVICES` 的配置约束始终有效，但手动 C API 创建的总数量仍由调用方保证不超容量。`WS2812B_CHECK_EXTENDED=y` 在初始化时进一步核实经典或 V5 的时序脉宽范围。SPI 初始化/发送与动态分配的实际失败仍会返回错误。

## 内存布局

`WS2812B_MEMORY_STATIC` 与 `WS2812B_MEMORY_DYNAMIC` 二选一，仅编译选中的内存实现。

静态模式使用固定 `struct ws2812 [WS2812B_MAX_DEVICES]` 以及两个分别按设备预留量合计的全局 RGB 和 SPI 帧池，**不会在 create 时调用 Heap**。三个 Kconfig 设备的帧容量可以填写每路 `WS2812B_DEVn_FRAME_BYTES`，0 表示按 `WS2812B_STATIC_MAX_CLOCKS_PER_BIT` 自动计算；可手工缩小，但必须保证实际所需容量。新增 C API 设备通过 `WS2812B_STATIC_EXTRA_PIXELS` 与 `WS2812B_STATIC_EXTRA_FRAME_BYTES` 配置额外预算，未预留时不能在静态模式下继续创建。

设 N 为灯珠数、C 为每 LED 数据位的 SPI 时钟数、F 为实际 SCK（Hz）、R 为 RESET 时间（μs）：

- 原始像素占 `4*N` 字节；
- 单个 SPI 编码像素占 `3*C` 字节；
- RESET 前后填充占 `2*ceil(F*R/8000000)` 字节；
- SPI 编码帧实际需求为 `3*N*C + 2*ceil(F*R/8000000)` 字节。

在 9 MHz、C=12、N=40、R=300 μs 下：一颗灯珠的编码占 36 字节，整个帧为 2116 字节，原始像素缓存占 160 字节。还应计入设备结构体内的 16 组四位模板、可选 256 字节/设备的自定义 Gamma 表以及灯效管理状态。应用自行决定线程栈的大小与分配方式。

## SPI 编码与时序

每个 LED 按 G7..G0、R7..R0、B7..B0 发送。默认直接采用 Kconfig 提供的 `(clocks_per_bit, T0H, T1H)`，不编译预设表和搜索逻辑；默认三路均为 9 MHz / V5 / (12,3,6)。`WS2812B_TIMING_TABLE` 提供匹配频率和版本的预设表；`WS2812B_TIMING_SEARCH` 仅在预设不匹配时使用整数搜索。

`WS2812B_ENCODER_FAST_ONLY` 默认使用 16 项四位 nibble 查表，每个 GRB 颜色字节复制两个编码模板；要求 SPI 位宽为偶数且不超过 `WS2812B_FAST_MAX_CLOCKS_PER_BIT`。`AUTO` 可为不适合快速查表的时序使用通用逐位编码。是否启用初始化时编码等价自检由 `WS2812B_ENCODER_SELF_CHECK` 控制，默认关闭。

SCK 必须是板级实际 SPI 时钟而不是仅请求值。必须核实 MOSI 空闲低电平、SPI 片选及 DMA 发送完成语义，并用示波器确认实际 T0H、T1H、T0L、T1L 与 reset-low。

## Gamma

Gamma 用于对亮度非线性感知进行映射。初始为 `WS2812_GAMMA_NONE`；可选择静态 `WS2812_GAMMA_22`、`WS2812_GAMMA_28`，或启用 `WS2812B_GAMMA_COMPUTE`，在运行时调用：

```c
ws2812_set_gamma_custom(dev, 180U);   /* Gamma 1.80 */
ws2812_set_gamma(dev, WS2812_GAMMA_28);
ws2812_set_gamma(dev, WS2812_GAMMA_NONE);
```

计算公式：`LUT[x] = round(255 * powf(x/255, exponent))`，指数为 `gamma_x100/100`，允许 50～500。计算只发生在调用时，每个设备保存自己的 256 字节表，Gamma 编码时只查表。采用浮点 `powf` 的配置需要 `libm`；没有选用计算功能则不编译该模块。亮度先以精确的 `(channel*brightness+127)/255` 等价整数式缩放，再选用 Gamma 表。

## 灯效

通过 `WS2812B_USING_EFFECT` 启用注册表，按需编译 `SOLID`、`BLINK`、`BREATHE`、`GRADIENT`、`GRADATION`、`CHASE`、`SEQUENCE` 独立方法。`GRADATION` 是随时间改变 RGB 起点并沿灯珠方向插值的空间渐变，不等于整条灯带同色过渡。

方法 `id` 直接作为 `methods[id]` 数组下标；启用的内置方法由条件编译生成连续 ID（顺序为 SOLID、BLINK、BREATHE、GRADIENT、CHASE、SEQUENCE、GRADATION），并非跨不同构建配置稳定的外部协议值。`WS2812B_EFFECT_METHOD_CAPACITY = WS2812_EFFECT_BUILTIN_COUNT + WS2812B_EFFECT_CUSTOM_SLOTS`，默认不预留自定义槽；如有自定义方法，先在 Kconfig 设置额外槽数，其 ID 从 `WS2812_EFFECT_BUILTIN_COUNT` 开始。`ws2812_effect_register_at(slot, method)` 要求 `slot == method->id`，注册必须在应用开始轮询之前完成。

每条灯带仅有一个当前效果，低电警告等临时效果由业务保存普通效果参数并手动切换/恢复。软件包不实现多层自动覆盖或任何应用特定控制策略。`ws2812_effect_start()` 只选择方法，真正渲染和 SPI 发送发生在应用调用 `ws2812_effect_poll()`/`poll_all()` 时。 对 SEQUENCE 可直接使用 `ws2812_effect_start_sequence(dev, steps, step_count)`，免去调用方手工构造通用 effect 配置；步骤数组在函数返回前被复制，持续时间仍使用 RT-Thread Tick。

软件包没有内部 worker、线程启动接口或应用 hook。应用在自己的一个任务或事件循环中调用 `ws2812_effect_poll_all(rt_tick_get())`，不得同时从多个上下文轮询或修改同一灯效。每个设备槽内部的 `WS2812B_DEVn_INITIAL_*` 单选菜单用于选择已启用的初始方法（槽 1 优先 GRADATION，其余优先 SOLID；SEQUENCE 不能作为默认方法），也可以选择 `No initial effect`，由应用在运行时启动灯效。若仅编入 SEQUENCE 或不编入其他默认可用方法，自动选择 `No initial effect`；此时 `ws2812_effect_init_kconfig()` 跳过该槽，且 `ws2812_effect_start_kconfig()` 不会停止其正在运行的灯效。初始效果由 Kconfig 的 `WS2812B_DEVn_INITIAL_*` 菜单自动生成隐藏的 `WS2812B_DEVn_INITIAL_KIND` 选择码，并通过 C99 预处理宏直接映射到当前构建的枚举 ID；不进行运行时转换，也不再生成固定数值 `WS2812B_DEVn_INITIAL_EFFECT`；`No initial effect` 的内部保留值为 65535，不属于可注册方法 ID。初始周期的配置下限为 2 Tick；**BLINK 必须使用偶数周期**（2、4、6…… Tick），因为其两个半周期通过整数除以 2 计算。选择 BLINK 时，初始周期若为奇数将触发 C99 编译期数组长度断言错误（无法编译），其他方法仍可使用奇数周期。运行时 `ws2812_effect_start()` 选择 BLINK 时，启用 `WS2812B_CHECK_PARAMETERS` 则奇数周期返回 `-RT_EINVAL`；MSH 命令会打印错误并返回 `-1`（成功为 `0`），不直接透传 API 错误码。关闭参数检查后，调用方必须保证偶数周期，否则实际闪烁周期会缩短 1 Tick。旧固定 ID 配置升级后应重新运行 `menuconfig`，保存 `.config/rtconfig.h`。可选 `ws2812b_msh.c` 独立编译为调试命令入口。

## 配置说明

- 关闭普通运行时检查会取消部分容量防护，产品固件建议在开发阶段开启；即使关闭，内存申请与 SPI 操作失败仍需处理。
- 静态模式不需要 Heap；使用动态模式时 RT-Thread 必须启用 Heap。
- 不进行设备名去重、重复初始化标记或运行时增删操作；这些使用边界由调用者负责。
- 使用 `rtdbg.h` 分级日志；按 RT-Thread 当前配置选择日志后端，编码热路径不打印逐像素日志。
- 当前设计不包括 BSP 的 GPIO/DMA/CS 配置，实际设备必须由 RT-Thread 注册或由调用者的 attach 回调注册。

## MSH 调试

启用 `WS2812B_USING_MSH`（需要 FinSH、灯效与 Kconfig 设备）后，统一使用 `ws2812b [slot=0] <command> [args...]`。省略槽位时默认操作第 0 路，三路硬件槽位可分别指定 `0/1/2`。无需额外 MSH 刷新线程：应用必须继续从自己的 LED 线程调用 `ws2812_effect_poll()`/`poll_all()`；任何 MSH 状态变更须与该轮询线程串行化，软件包本身不提供锁。

```text
ws2812b list                # 查看各槽灯珠数、SPI 配置频率、亮度、Gamma
ws2812b 2 info              # 查看第 2 路设备状态
ws2812b bright 64           # 先降低第 0 路亮度，避免全白时供电过载
ws2812b s FF0000            # 红色通道测试 Gamma：在较低全局亮度下更容易观察
ws2812b gamma off           # 不启用 Gamma
ws2812b gamma 28            # 启用 Gamma 2.8（需要编译对应查找表）
ws2812b s red               # 固定红色
ws2812b bl                  # 红/灭交替，默认完整周期 600 Tick
ws2812b br                  # 绿色呼吸，默认 1000 Tick
ws2812b gr                  # 红蓝时间渐变，默认 1000 Tick
ws2812b gd                  # 空间渐变，默认 20 Tick
ws2812b ch                  # 绿色追逐，默认单步 100 Tick
ws2812b seq                 # 红/绿/蓝各 500 Tick 依次循环
ws2812b seq r:200 g:300 b:400  # 自定义每步时间
ws2812b colors              # 15 种不同的预定义颜色，每色 1000 Tick
ws2812b 1 colors 500        # 第 1 路，每色 500 Tick
ws2812b off                 # 以 SOLID 方式关闭显示
ws2812b reset               # 恢复该槽 Kconfig 初始效果
ws2812b help                # 查看完整命令和默认值
```

常用颜色可以直接写 `red/green/blue/off/...`，`r/g/b` 为简写，也兼容六位十六进制 RGB888，例如 `ws2812b 0 s 404040`。`seq` 每个颜色参数可选 `:ticks` 后缀，不必重复输入每组持续时间；省略时使用 500 Tick。`colors` 自动循环 `ws2812_color.h` 的 15 个不同 RGB 值，包括以兼容宏表示的 `Dark_green_90`、`Light_green_150`、`Medium_blue_210`、`lilac_330`，并打印颜色名称和数值。受 `FINSH_ARG_MAX` 和 `FINSH_CMD_SIZE` 限制，长序列应改用 `colors` 或拆分命令。

`gamma off` 与 `gamma 28` 可用来对比中等强度的输出（例如先设 `bright 128`，再设 `s FF0000`）；`gamma 22` 只在 `WS2812B_GAMMA_22` 编入时可用，`gamma 50..500` 是可选 `WS2812B_GAMMA_COMPUTE` 下的指数×100，当前只启用 Gamma 2.8 的固件不支持 `gamma 22` 或自定义指数。亮度范围 0～255；颜色和亮度/Gamma 变更在下一次应用轮询后可见。MSH 成功反馈表示参数已受理，不代表 SPI 波形或真实灯珠显示已经通过实测。若 CAN 下发新的灯效或低电量告警切换，业务逻辑仍可能覆盖当前 MSH 效果。`BLINK` 完整周期须为不小于 2 的偶数 Tick。
