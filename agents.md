# RTKLIB-B2b 项目认知笔记

生成时间：2026-05-14  
工作目录：`D:\Desktop\rtklib_b2b`  
主体项目目录：`RTKLIB-B2b`

## 1. 项目概览

`RTKLIB-B2b` 是基于 RTKLIB 扩展的 C/C++ GNSS 工具包，核心目标是解码 BDS-3 PPP-B2b 改正信息，并将解码后的 SSR 改正用于精密单点定位（PPP）。

从 README、源码和示例配置看，项目主要支持：

- PPP-B2b 改正流解码：支持实时解码和事后解码。
- PPP-B2b 定位：支持实时 PPP 和事后 PPP。
- 接收机格式：重点支持 SinoGNSS/司南（K803W/K803LITE/W/S）和 Unicore/和芯星通（UM980/UM982）。
- 卫星系统：配置示例主要使用 GPS+BDS，源码也保留 RTKLIB 的多系统能力。
- 输出：B2b SSR 信息可输出为 ASCII 格式，定位输出沿用 RTKLIB 的 `.pos` 等格式。

README 声明项目许可证为 GPLv3，版本宏在 `include/rtklib.h` 中为 `VER_RTKLIB "v1.0"`，补丁级别为 `PATCH_LEVEL "20250305"`。

## 2. 顶层结构

当前目录只有一个主体子目录：

- `RTKLIB-B2b/`：实际项目仓库，包含源码、构建脚本、示例和文档。

`RTKLIB-B2b/` 内主要目录：

- `app/`：命令行应用入口。
- `src/`：RTKLIB 主体源码和 B2b 扩展实现。
- `include/`：公共头文件，核心是 `rtklib.h` 和 `B2b.h`。
- `example/`：事后解码、实时解码、实时 PPP、事后 PPP 示例配置和示例输出。
- `manual/`：用户手册 PDF 和论文 PDF。
- `bin/`：已有可执行文件和清理脚本。
- `lib/`：已有静态库 `libB2bLib.a`。
- `out/`、`build/`、`.vs/`、`.vscode/`：本地构建/IDE 产物或配置。

当前可见源码规模约为 68 个 `.c` 文件、9 个 `.h` 文件、13 个 `.conf` 示例配置文件。

## 3. 构建方式

项目使用 CMake：

- 根配置：`RTKLIB-B2b/CMakeLists.txt`
- 库配置：`RTKLIB-B2b/src/CMakeLists.txt`
- 应用配置：`RTKLIB-B2b/app/*/CMakeLists.txt`

根 CMake 设置：

- C 标准：C99。
- 运行产物输出到 `bin/`。
- 静态库和动态库输出到 `lib/`。
- 构建 `B2bLib` 静态库，并链接 4 个应用：`postppp`、`rtppp`、`postdecoder`、`rtdecoder`。

`B2bLib` 包含 RTKLIB 主体源码、B2b 扩展源码、接收机解码器和 `f2c` 转换的潮汐/气象模型源码。编译定义默认启用：

- `ENAGLO`
- `ENAGAL`
- `ENACMP`
- `ENAQZS`

平台链接差异：

- Unix/Linux：链接 `m` 和 `pthread`。
- Windows：增加 `WIN32/_WIN32` 定义，CMake 中为 MinGW 路径配置了 `-mthreads`、`ws2_32`、`winmm`。

`CMakeSettings.json` 是 Visual Studio/Ninja 的 `x64-Debug` 配置，继承 `msvc_x64_x64` 环境。已有日志中出现过 MSVC 下 `winsock.h` 与 `winsock2.h` 重定义错误，以及中文编码 C4819 警告；README 也建议实时场景优先使用 Linux，Windows 更适合事后处理。

## 4. 主要命令行程序

### `postdecoder`

路径：`app/postdecoder/postdecoder.c`

用途：事后解码接收机私有 PPP-B2b 二进制数据，输出 B2b SSR ASCII 结果。

用法来自源码和 `example/postdecoder/note.txt`：

```text
./postdecoder [-U|-S] -in B2b_filepath [-out B2bSSR_resultpath]
```

参数：

- `-U`：Unicore 格式。
- `-S`：SinoGNSS 格式。
- `-in`：输入 PPP-B2b 二进制文件。
- `-out`：输出路径；未指定时默认为 `<input>.B2bSSR`。

示例数据：`example/postdecoder/Unicore_2024360.B2bBin`。

### `rtdecoder`

路径：`app/rtdecoder/rtdecoder.c`

用途：实时 PPP-B2b 解码服务器，只接受 SinoGNSS 或 Unicore 输入流。支持控制台、无控制台启动、NTRIP/file 等 RTKLIB 风格流配置。

主要命令行：

```text
B2bPPP [-s] [-nc] [-o file] [--version]
```

典型配置：`example/rtdecoder/conf/rtdecoder_mycaster_sino.conf`。

### `postppp`

路径：`app/postppp/postppp.c`

用途：事后 PPP 定位。它类似 RTKLIB 的 `rnx2rtkp`，但通过配置文件读取输入文件、输出文件和 PPP-B2b 选项。

常见启动方式：

```text
./postppp -o conf/postppp_25065_wuh2.conf
```

关键配置：

- `PPP_Glo.RT_flag = 0`
- `prcopt.sateph = 5`：使用广播星历 + PPP-B2b。
- `prcopt.B2b_format = 20`：SinoGNSS。
- `prcopt.B2b_format = 21`：Unicore。
- 输入文件中 `.b2b` 或 `.B2b` 会被 `postpos.c` 识别为 B2b 改正文件。

示例配置位于 `example/postppp/conf/`。

### `rtppp`

路径：`app/rtppp/rtapp.c`

用途：实时 PPP 定位入口。该程序很薄，直接调用 `app_rtkrcv(argc, argv)`，实际逻辑在 `src/rtkrcv.c`。

典型配置：`example/rtppp/conf/rtppp_25080_unicore.conf` 等。

## 5. B2b 核心实现位置

### 公共类型和宏

核心文件：`include/rtklib.h`

重要新增点：

- `EPHOPT_B2b = 5`：星历选项，表示广播星历 + B2b SSR APC。
- `STRFMT_SINO = 20`
- `STRFMT_UNICORE = 21`
- `prcopt_t.B2b_format`：配置 B2b 接收机格式。
- `B2bmask_t`：保存 B2b 掩码、IOD、有效卫星列表等。
- `B2bssr_t`：保存 B2b SSR 改正，包括轨道、钟差、码偏、URA、IOD、更新时间等。
- `nav_t.B2bssr[MAXSAT]`：导航数据结构中保存 B2b SSR 改正。

### B2b 工具函数

核心文件：`src/B2b.c`、`include/B2b.h`

主要职责：

- GPST/BDST/MJD 时间转换。
- 根据 B2b 消息 TOD 和接收头时间推导 RTKLIB `gtime_t`。
- B2b satellite slot 到 RTKLIB satno 的映射。
- B2b mask 合并并生成有效卫星列表。
- 初始化 `B2b_t`。
- 校验轨道、码偏、钟差改正一致性。
- 输出 MASK、ORBIT_URAI、DIFF_CODE_BIAS、CLOCK 到 B2b trace 文件。

### 接收机私有协议解码

重点文件：

- `src/rcv/sinan.c`：SinoGNSS/司南 PPP-B2b 私有格式解码。
- `src/rcv/unicore.c`：Unicore PPP-B2b 私有格式解码。
- `src/rcvraw.c`：将 `STRFMT_SINO` 和 `STRFMT_UNICORE` 接入 `input_raw/input_rawf`。

解码器将 B2b 消息写入 `raw->nav.B2bssr[...]`，并设置 `update` 标志。消息类型主要对应：

- Message 1：MASK
- Message 2：EPH + URA
- Message 3：Code Bias
- Message 4：Clock

### 定位链路中的 B2b 改正

重点文件：

- `src/ephemeris.c`：`satpos_B2b()` 和 `satpos_B2b_otp()` 计算 B2b 改正后的卫星位置和钟差；`satpos()` 中 `EPHOPT_B2b` 分支会调用该逻辑。
- `src/ppp.c`：`corr_meas()` 中 `EPHOPT_B2b` 分支会用 `nav->B2bssr[obs->sat].cbias[...]` 修正伪距码偏。
- `src/postpos.c`：事后处理时读取 `.b2b/.B2b` 文件，并按观测历元更新 `navs.B2bssr`。
- `src/rtksvr.c`、`src/rtkrcv.c`、`app/rtdecoder/rtdecoder.c`：实时流更新 B2b SSR，并维护服务器状态和日志。

注意：B2b SSR 数组在多处使用 `nav->B2bssr[sat]` 或 `nav->B2bssr[obs->sat]`，而传统 RTKLIB SSR 常见索引是 `sat-1`。修改相关代码时必须先确认当前项目的索引约定，不能机械套用原版 RTKLIB 写法。

## 6. 配置和示例数据

### 事后 PPP 示例

目录：`example/postppp/conf/`

示例配置包括 `postppp_25065_wuh2.conf`、`postppp_25066_gamg.conf` 等，通常包含：

- RINEX 观测文件。
- 广播星历文件。
- PPP-B2b 改正文件。
- 天线文件 `igs20_2350.atx`。
- 海潮负荷文件 `ocnload_241227.blq`。
- 输出 `.pos` 路径。

### 实时 PPP 示例

目录：`example/rtppp/conf/`

配置采用 RTKLIB 流配置风格：

- `strtype[0..2]`：输入流类型。
- `strfmt[0..2]`：输入格式，`20` 为 SinoGNSS，`21` 为 Unicore。
- `strpath[0..2]`：rover/base/correction 数据流或回放文件。
- `strtype[3..4]`、`strfmt[3..4]`、`strpath[3..4]`：输出流。

示例使用 `::T::x5` 这样的回放参数模拟实时输入。

### 实时解码示例

目录：`example/rtdecoder/`

`note.txt` 说明实时 PPP 数据集也可用于该模块，只需指定 PPP-B2b 改正和广播星历数据路径并开启回放模式。

## 7. 日志、输出和调试

常见输出/日志：

- B2b SSR trace：`B2bPPP_%Y_%m_%d.B2bssr`
- 常规 trace：`B2bPPP_%Y_%m_%d.trace`
- 实时状态：`B2bPPP_%Y_%m_%d.stat`
- 实时日志：`B2bPPP_%Y_%m_%d.log`
- 事后定位：`.pos`

`B2b_tracelevel(22)` 在多个入口中默认开启较高等级 B2b SSR 输出。`src/trace.c` 中实现了独立的 B2b trace 文件句柄和按时间替换路径逻辑。

当前仓库根目录有 `download_test.log` 和 `rtkcmn_test.log`，显示曾经用 MSVC 编译时出现大量安全函数警告；`download_test.log` 末尾还显示 Windows SDK `winsock.h/winsock2.h` 重定义错误。`build_download.log` 和 `build_postppp.log` 当前为空文件。

## 8. 开发注意事项

- 优先保留 RTKLIB 原有接口风格，B2b 扩展通常通过新增枚举、结构字段和分支接入原流程。
- 修改 B2b 定位链路时，需要同时检查 `ephemeris.c`、`ppp.c`、`postpos.c`、`rtksvr.c/rtkrcv.c` 的数据流是否一致。
- 修改接收机格式时，至少检查 `include/rtklib.h` 的格式编号、`src/rcvraw.c` 的 `input_raw/input_rawf` 分发、对应 `src/rcv/*.c` 解码器。
- 事后模式和实时模式共享很多底层逻辑，但入口和配置来源不同，不要只验证一个模式就认为另一个模式正常。
- Windows 下要特别关注头文件包含顺序、`winsock2.h` 冲突和编码警告；Linux/MinGW 路径更贴近当前 CMake 链接设置。
- README 中中文/emoji 显示存在编码乱码，源码中也有部分中文注释乱码；编辑这些文件时要谨慎处理编码，避免扩大乱码范围。

## 9. 本地操作约束

当前工作区明确禁止批量删除文件或目录。不要使用：

- `del /s`
- `rd /s`
- `rmdir /s`
- `Remove-Item -Recurse`
- `rm -rf`

如确需删除文件，只能一次删除一个明确路径的文件，例如：

```powershell
Remove-Item "C:\path\to\file.txt"
```

如果需要批量删除文件，应停止操作并询问用户，让用户手动删除。
