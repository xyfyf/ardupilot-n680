# N680 / EFT_EDU 硬件适配

本目录提供 N680 飞控的 ArduPilot 板级定义，编译目标为 `EFT_EDU`。
固定使用生产分支最初派生的公开 ArduPilot 基线
[`aabbe500aa4b682305b3062116edebc3fd0528c6`](https://github.com/ArduPilot/ardupilot/commit/aabbe500aa4b682305b3062116edebc3fd0528c6)，
该提交是 EFT 定制分支的原始上游父提交；本仓库固定此基线，未升级到后续 master 或稳定版。
基线的 Copter 版本标识为 `ArduCopter V4.7.0-dev`。

## 硬件

| 项目 | 配置 |
| --- | --- |
| MCU / 晶振 | STM32H743，2MB Flash，16MHz |
| Bootloader / 应用区 | 厂商本地板型 ID 9903；应用从 `0x08020000` 开始 |
| IMU | ICM42670，SPI2，CS=PE12，芯片相对板坐标旋转 `YAW_180` |
| 板载罗盘 | QMC5883P，I2C1，地址 `0x2C` |
| 气压计 | SPL06，I2C1，地址 `0x77`，以实际装配为准 |
| 外置罗盘 | IST8310，I2C4，地址 `0x0F`，可选 |
| 电流电压模块 | INA238，I2C4，地址 `0x40`；默认分流电阻 `0.0005Ω`，以实际模块为准 |
| CAN | CAN1，PD0/PD1 |
| SD | SDMMC1，PC8～PC12、PD2 |
| 输出 | PWM1～PWM12；PWM13/PC6供WS2812使用 |
| 状态LED | 红PE6、绿PE5、蓝PC5，高电平点亮 |

板型ID `9903` 沿用现有N680 bootloader，属于厂商本地ID，尚非上游正式分配的ID。
官方新增板型规范检查会提示该ID超出分配范围；本仓库保留兼容性，本地编译已通过。
如需向上游提交板型，应先申请正式ID并处理bootloader兼容。

### 串口

| 参数端口 | 硬件 | 丝印 / 用途 | 默认协议及波特率 |
| --- | --- | --- | --- |
| SERIAL0 | USB OTG1 | Type-C | 标准 MAVLink2 |
| SERIAL1 | USART1，PA9/PA10 | COM1 | MAVLink2，57600 |
| SERIAL2 | UART4，PA0/PA1 | COM2 | MAVLink2，115200 |
| SERIAL3 | USART2，PA2/PA3 | GPS | GPS，460800 |
| SERIAL4 | USART3，PD8/PD9 | LINK | MAVLink2，115200 |
| SERIAL5 | UART5，PB13/PB12 | ELRS | RCIN，CRSF/ELRS |
| SERIAL6 | UART8，PE1/PE0 | 扩展 | 关闭 |
| SERIAL7 | UART7，PE8/PE7 | 扩展 | 关闭 |

USB使用基线 ArduPilot 的标准 MAVLink 帧格式。默认参数只选择硬件接口，
不包含某架飞机的机架、电机顺序、PID、传感器校准或机型参数迁移。

### 输出引脚及定时器分组

| 输出 | 引脚 | 定时器 |
| --- | --- | --- |
| PWM1 / PWM4 / PWM5 / PWM6 | PA8 / PE14 / PE13 / PE11 | TIM1 |
| PWM2 / PWM3 / PWM10 | PB11 / PB10 / PB3 | TIM2 |
| PWM7 / PWM8 / PWM9 | PB1 / PB0 / PB4 | TIM3 |
| PWM11 / PWM12 | PD15 / PD14 | TIM4 |
| PWM13，WS2812 | PC6 | TIM8 |

同一定时器组的协议设置相互关联，配置DShot时应按组核对。
本适配使用基线的标准输出驱动，不包含定制启动PWM或电调直通补丁。

## 从零编译（Ubuntu 24.04 / WSL2）

使用 Linux 文件系统，例如 `~/ardupilot-n680`，保留 Git 可执行位。

```bash
git clone --recurse-submodules https://github.com/xyfyf/ardupilot-n680.git
cd ardupilot-n680
Tools/environment_install/install-prereqs-ubuntu.sh -y
. ~/.profile

# 首次构建需要先生成本板 bootloader。
python3 Tools/scripts/build_bootloaders.py EFT_EDU

# 配置并使用所有逻辑CPU编译 Copter。
./waf configure --board EFT_EDU
./waf copter -j"$(nproc)"
```

本仓库基线的依赖安装脚本支持 ARM GCC `10-2020-q4-major`。
如果工具链已安装，确保其 `bin` 目录在 `PATH` 中；已有完整依赖时可跳过安装步骤。
`build_bootloaders.py` 内部的Waf使用默认CPU并行度。

输出文件：

- `build/EFT_EDU/bin/arducopter.apj`：地面站刷机文件，板型ID为9903。
- `build/EFT_EDU/bin/arducopter_with_bl.hex`：包含bootloader的HEX。
- `Tools/bootloaders/EFT_EDU_bl.bin`：本地生成的bootloader。

更新已有克隆后同步依赖：

```bash
git pull --ff-only
git submodule update --init --recursive
```

所有子模块均引用基线固定的公开仓库及提交，不需要访问私有 MAVLink 仓库。

## 开发与验证范围

本仓库发布硬件映射、引导程序配置、接口默认值和公开的 ArduPilot 基线，供二次开发。
新增板级定义与文档由AI辅助整理，硬件映射来源于板卡已有配置；所有新提交由仓库维护者负责。
编译检查覆盖bootloader及Copter，发布前检查APJ板型和HEX应用区一致性。
尚未在公开版固件上完成实机传感器、电机输出、飞行及升级验证。

请按实际硬件版本核对传感器装配、分流电阻、引脚和安装方向，完成校准及机架配置后再测试。
已有飞控参数在刷机后可能继续保留；新固件的 `defaults.parm` 不会自动覆盖所有已保存参数。

许可证沿用仓库根目录的 [GPLv3](../../../../COPYING.txt)。
