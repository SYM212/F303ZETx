# F303ZETx — STM32F303ZET6 裸机样板工程

基于 **STM32F303ZET6**（Cortex-M4F，512 KB Flash / 64 KB SRAM + 16 KB CCM）的裸机样板工程，使用 **ST 标准外设库 SPL + CMSIS**（`Library/` 为 2021 年版驱动，版本 V1.2.4；`Start/` 为 CMSIS 内核头、启动文件与 DSP 库），无 RTOS、无 HAL/CubeMX。

这个仓库的定位是**起点**而不是成品固件：启动文件、系统时钟（72 MHz）、内核异常处理框架、两套可编译可烧录的工程（Keil MDK 与 VS Code + Embedded IDE）以及目录划分约定都已就绪，应用代码从 `User/main.c` 的空主循环开始写。样板本身不含任何外设初始化代码 —— `Hardware/`、`Periph/`、`System/` 三个目录已建好且已进入两个工程的搜索路径，等待按需填充。

## 芯片与资源

| 项目 | 说明 |
| --- | --- |
| 目标器件 | STM32F303ZETx（STM32F303xE，LQFP144），Keil 器件包 `Keil.STM32F3xx_DFP.3.1.0` |
| 内核 | Cortex-M4，单精度 FPU + DSP 指令（`--cpu Cortex-M4.fp`） |
| 存储 | 512 KB Flash（`0x08000000`）；64 KB SRAM（`0x20000000`）+ 16 KB CCM（`0x10000000`） |
| 系统时钟 | HSE 8 MHz × PLL9 = **72 MHz**；AHB /1、APB1 /2（36 MHz）、APB2 /1（72 MHz）、Flash 2 等待周期 |
| 外设库 | STM32F30x_StdPeriph_Driver V1.2.4，`Library/` 下 24 个驱动源文件（adc/can/comp/crc/dac/dbgmcu/dma/exti/flash/fmc/gpio/hrtim/i2c/iwdg/misc/opamp/pwr/rcc/rtc/spi/syscfg/tim/usart/wwdg） |
| CMSIS | `core_cm4.h` V4.00（含 core_cmFunc / core_cmInstr / core_cmSimd）+ `system_stm32f30x.c` |
| DSP 数学库 | `Start/arm_cortexM4lf_math.lib`（约 12.7 MB）与 `arm_math.h`，宏 `ARM_MATH_CM4` 已定义，已参与链接 |
| 构建规模 | 27 个 C 文件 + 1 个汇编文件；Code=692 / RO-data=420 / RW-data=0 / ZI-data=1632 字节 |
| 尚未包含 | RTOS、HAL/CubeMX、SysTick 延时、串口 printf 重定向、看门狗喂狗、任何外设示例 |

## 目录结构

仓库根目录是源码与版本控制根目录，`EIDE/` 是 VS Code / Embedded IDE 的工作目录：

```text
F303ZETx/
├── EIDE/                       # VS Code + Embedded IDE 工程
│   ├── .eide/eide.yml          # 源文件目录、宏、包含路径、工具链、下载配置
│   ├── .eide/files.options.yml # 单文件级追加编译选项（当前为空）
│   ├── .vscode/tasks.json      # build / rebuild / clean / flash / build and flash
│   ├── F303ZETx.code-workspace # VS Code 工作区（同时打开 EIDE/ 与仓库根目录）
│   ├── .clang-format           # Microsoft 基准，4 空格缩进
│   └── build/Debug/            # 构建产物（被 .gitignore 忽略）
├── Hardware/                   # 板级器件驱动（空目录，预留）
├── Library/                    # ST SPL 驱动源码（上游代码，通常不改）
├── Periph/                     # 外设抽象层（空目录，预留）
├── Start/                      # CMSIS 内核头 + system_stm32f30x.c + 启动文件 + DSP 库
│   └── ARM/startup_stm32f303xe.s   # 另有 4 个其它型号的启动文件（构建时排除）
├── System/                     # 算法与系统工具（空目录，预留）
├── User/                       # 应用层入口与中断实现
│   ├── main.c / main.h         # 空主循环；main.h 为应用层总包含头
│   ├── stm32f30x_conf.h        # SPL 外设头文件开关、USE_FULL_ASSERT 开关
│   └── stm32f30x_it.c / .h     # Cortex-M4 内核异常处理模板
├── Keil/                       # Keil MDK 工程（F303ZETx.uvprojx）
├── .gitignore
└── README.md
```

约定：`User/main.h` 是应用层总包含头，各层模块的公共头文件都由它汇总（当前只有 `stm32f30x.h` 与 `arm_math.h`）；上层依赖下层——`Hardware/` 是板级器件驱动，`Periph/` 是外设初始化抽象，`System/` 放算法与通用工具，`User/` 放主流程与中断。

## 启动与时钟

- **启动文件** `Start/ARM/startup_stm32f303xe.s`：栈 `Stack_Size = 0x400`、堆 `Heap_Size = 0x200`，向量表 → `Reset_Handler` → `SystemInit()` → `__main` → `main()`。同目录另有 `startup_stm32f302x8.s`、`startup_stm32f303xc.s`、`startup_stm32f30x.s`、`startup_stm32f334x8.s` 四个非本型号的文件，两个工程都已排除，不要误加。
- **系统时钟** `Start/system_stm32f30x.c`：`VECT_TAB_OFFSET = 0x0`（向量表留在 Flash 起始，`VECT_TAB_SRAM` 未启用），HSE 经 PLL 倍频到 72 MHz。
- **中断框架** `User/stm32f30x_it.c`：提供 NMI / HardFault / MemManage / BusFault / UsageFault / SVC / DebugMon / PendSV / SysTick 的空实现或死循环实现，文件末尾留有 `PPP_IRQHandler` 注释模板；外设中断服务函数需要自己新增（函数名必须与启动文件的向量名一致）。
- **晶振假设**：代码假定 HSE = 8 MHz（`Start/stm32f30x.h` 的 `HSE_VALUE`）。板子上晶振不是 8 MHz 时，需要同时修改 `HSE_VALUE` 与 `system_stm32f30x.c` 里的 PREDIV / PLLMUL、Flash 等待周期。

## 构建与烧录

工程没有 Makefile 或 CMake，采用两套并行的构建系统，两边配置口径基本一致（实测产出的 Program Size 完全相同，差异见文末“已知问题与待办”）：

### EIDE（主要入口）

用 VS Code 打开 `EIDE/F303ZETx.code-workspace`，安装扩展 `cl.eide`（Embedded IDE）后即可构建，工程定义在 `EIDE/.eide/eide.yml`：

- **工具链**：AC5（ARM Compiler 5，实测 5.06），Cortex-M4，单精度硬件浮点，C99，优化等级 `-O0`，函数级分段，`--diag_suppress=1 --diag_suppress=1295`，输出格式 elf。
- **宏定义**：`STM32F303xE`、`USE_STDPERIPH_DRIVER`、`ARM_MATH_CM4`。
- **包含路径**：`../Hardware`、`../Library`、`../Periph`、`../Start`、`../System`、`../User`。
- **源文件来源**：`srcDirs` 目录扫描上述 6 个目录，**新增 `.c` 文件放进这些目录即自动参与构建**，无需改工程文件；`excludeList` 排除 4 个无关启动文件。
- **存储布局**：ROM `0x08000000` 长度 `0x80000`；RAM 分两段——SRAM `0x20000000` 长度 `0x10000`、CCM `0x10000000` 长度 `0x4000`。未使用自定义 scatter，链接脚本由 EIDE 自动生成到 `EIDE/build/Debug/F303ZETx.sct`（对应 `RW_IRAM1` / `RW_IRAM2`）。
- **烧录**：`uploader: OpenOCD`，`interface: stlink`，`target: stm32f3x`，基址 `0x08000000`；调试器配置为 `cortex-debug`。

VS Code 任务（`终端 > 运行任务`）：

| 任务 | EIDE 命令 |
| --- | --- |
| `build` | `eide.project.build` |
| `rebuild` | `eide.project.rebuild` |
| `clean` | `eide.project.clean` |
| `flash` | `eide.project.uploadToDevice` |
| `build and flash` | `eide.project.buildAndFlash` |

构建产物在 `EIDE/build/Debug/`：`F303ZETx.axf`、`.hex`、`.bin`、`.s19`、`.map`、`.htm`（静态调用图）、`.sct`，以及 IDE 侧用的 `compile_commands.json`、`.obj/` 中间文件；该目录整体被 `.gitignore` 忽略。

### Keil MDK

打开 `Keil/F303ZETx.uvprojx`（实测 μVision 5.24 / `__UVISION_VERSION=524`，AC5，器件包 `Keil.STM32F3xx_DFP.3.1.0`）直接编译即可：

- 编译选项：`--c99 -c --cpu Cortex-M4.fp -g -O0 --apcs=interwork --split_sections`，**未启用 MicroLIB**，`CreateHexFile` 当前为 0（需要 hex 时在 `Options for Target > Output` 勾选）。
- 宏定义：`STM32F303xE`（由器件包按所选器件自动带上）`USE_STDPERIPH_DRIVER`、`ARM_MATH_CM4`；包含路径 `..\Start;..\Library;..\Periph;..\Hardware;..\System;..\User` + 器件包提供的 CMSIS 路径。
- 文件分组：`Start`（CMSIS 头、`system_stm32f30x.c`、`startup_stm32f303xe.s`、`arm_cortexM4lf_math.lib`）、`Library`（24 个驱动）、`Periph` / `Hardware` / `System`（空分组）、`User`。
- 产物输出到 `Keil/Objects/`、列表到 `Keil/Listings/`（均被 `.gitignore` 忽略）；`Keil/keilkill.bat` 用于清理这些中间文件（脚本内保留 `*.opt` 以免丢掉 J-Link 设置）。
- **注意**：Keil 侧文件是显式列在工程里的，**新增 `.c` 文件必须在 μVision 中手工加入对应分组**，否则不会被编译。