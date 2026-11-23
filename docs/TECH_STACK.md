# 技术栈与实现分析

> 分析对象：Perfect Vision 5.0.4 交付包、使用说明书与工程源码。  
> 分析日期：2026-11-23。  
> 证据来源：`VisionInspect.sln`、各工程 `.csproj`、`Directory.Build.props/targets`、源码调用点、交付包 PE 元数据及随包 PDF 手册。仓库不包含海康 SDK 二进制，版本结论以工程引用与本机 SDK 元数据为准。

## 1. 结论摘要

Perfect Vision 是一套 **C# / .NET Framework 4.7.2 / WinForms** 的 64 位工业视觉上位机，核心技术栈由四部分组成：

1. **视觉**：海康机器人 VisionMaster 4.3.0 SDK，负责 `.sol` 方案加载、流程执行、结果读取和参数编辑；通过专用 STA 线程串行调用，规避 COM/原生组件的线程亲和性问题。
2. **采集**：海康机器人 MVS SDK，负责工业相机枚举、连接、回调取图、软硬触发、曝光/增益写入与掉线重连。
3. **通讯**：自研 Modbus TCP 主站（FC03/FC16）、TCP 触发服务与串口通道，不依赖 NModbus 等第三方协议库。
4. **数据**：JSON 配置、按日 CSV、PNG 图片归档、手写 OpenXML 的 XLSX 报表，全部本地落盘，不依赖 Office、MySQL 或日志框架。

工程按 Core / Infrastructure / VisionMaster / Cameras / Communication / App / SDK 七层拆分，检测链路与 SDK、UI、存储、通讯解耦；独立 SDK 复用同一套 VM 适配层，不启动界面。

## 2. 应用构成

| 交付物 | 说明 |
| --- | --- |
| `Perfect Vision.exe` | 约 181 MB 的 WiX Burn 引导程序（内部名 `burn`，文件版本 5.0.4.0）。链式安装 .NET Framework 4.8、VC++ x64 运行库、MVS 相机驱动与客户端 MSI |
| `使用说明书.pdf` | 7 页，ReportLab 生成，覆盖安装、授权、检测、参数、相机/通讯/报表、卸载与排障 |
| 工程源码 | 7 个工程、约 1.6 万行 C#（含测试宿主），x64 / .NET Framework 4.7.2 / C# 7.3 |

安装包不是单文件绿色程序：它先做运行环境检测，再按顺序安装依赖，最后安装到 `[ProgramFiles64Folder]Perfect Vision`，并以 `XVision.exe` 作为启动目标。程序集、注册表项（`SOFTWARE\XVision\Customer`）与数据目录（`XVision.Customer`）沿用 `XVision` 命名，是产品改名后为兼容现场部署保留的内部标识。

## 3. 分层技术栈

### 3.1 语言与运行时

| 项 | 采用值 | 约束方式 |
| --- | --- | --- |
| 语言 | C# 7.3 | 所有工程 `LangVersion=7.3` |
| 运行时 | .NET Framework 4.7.2 | `TargetFrameworkVersion=v4.7.2`；安装包另带 .NET Framework 4.8 |
| 平台 | Windows x64 | `PlatformTarget=x64`、`Prefer32Bit=false`；`Directory.Build.targets` 在构建前校验，AnyCPU/x86 直接报错 |
| 工具链 | Visual Studio 2022 / MSBuild | 解决方案格式 12.00，VS 17 |
| 构建特性 | Deterministic、AutoGenerateBindingRedirects | 便于复现构建与程序集绑定 |

选择 .NET Framework 而非 .NET 6/8，是因为 VM 4.3 的托管程序集与 WinForms 控件面向 .NET Framework；选择 x64 是因为 VM 的算法模块为 64 位原生库。

### 3.2 桌面界面

- **WinForms 为主体**：全部窗体、控件与消息循环基于 `System.Windows.Forms`，无第三方 UI 框架。
- **自绘控件体系**：`RoundedButton`、`SegmentedControl`、`StudioTabControl`、`StudioComboBox`、`RoleBadge` 等自绘控件，配合 `Theme.cs` / `Theme.Animation.cs` 实现日夜主题、配色过渡与动画。
- **WPF 互操作**：通过 `WindowsFormsIntegration` 的 `ElementHost` 承载 WPF `FrameworkElement`，用于 VM 官方参数编辑器的宿主与布局（交付版 `ParameterRenderBridge`）。工程同时引用 `PresentationCore`、`PresentationFramework`、`WindowsBase`。
- **原生控件嵌入**：VM 的 `VmRenderControl` 负责图像与 ROI/结果渲染，`VmParamsConfigControl` 负责官方参数编辑器；本项目只做宿主、布局与权限控制，不重写算法交互。
- **透明叠加层**：十字线、坐标与大号 OK/NG 由独立顶层窗口 `CrosshairOverlayWindow` 绘制，使用 `TransparencyKey` + `WS_EX_TRANSPARENT` + `HTTRANSPARENT`，实现视觉透明与鼠标透传；缩放/平移仍交给 VM 控件。
- **高 DPI**：`app.manifest` 声明 `PerMonitorV2`，应用配置同时启用 WinForms 的 `DpiAwareness`。

### 3.3 视觉引擎（VisionMaster 4.3.0）

引用的官方程序集与用途：

| 程序集 | 用途 | 版本证据 |
| --- | --- | --- |
| `VM.Core.dll` | `VmSolution`、`VmProcedure`、`VmModule`、`VmIO`、结果结构 | FileVersion 4.3.0.1373494 |
| `VM.PlatformSDKCS.dll` | `ImageBaseData`、`VMPixelFormat`、`VmException` | 4.3.0.1335825 |
| `VMControls.Winform.Release.dll` | `VmRenderControl`、`VmParamsConfigControl` | 4.3.0.1262187 |
| `VMControls.Interface.dll` | `IVmModule` 绑定 | 随 SDK |
| `ImageSourceModuleCs.dll` | `ImageSourceModuleTool`、`ImageSourceParam` | 1.0.0.0 |
| `FormatModuleCs.dll` | `FormatModuleTool`、`FormatResult` | 1.0.0.0 |

关键调用链：

```text
VmSolution.Load(.sol) → GetAllProcedureList() → VmSolution.Instance["Flow.Module"]
→ ImageSourceType = SDK → SetImageData(ImageBaseData, true) → Procedure.Run()
→ OnWorkEndStatusCallBack → ModuResult.GetOutputInt/Float/String → ResultJudge
```

- **图像注入**：相机帧或本地图片统一转为 `RawImage`（Mono8 / RGB24 / BGR24），BGR 在适配层转 RGB24，再构造 `ImageBaseData` 交给图像源模块，保证本地与相机走同一条检测链路。
- **异步执行**：不使用会阻塞 SDK 消息泵的 `SyncRun()`，而是 `Run()` 启动、普通线程等待 `TaskCompletionSource`、回调后回 STA 读取结果并释放图像；`RunTimeoutMs` 防止永久等待，超时保留上下文禁止重跑。
- **结果解析**：优先读流程输出 `GetAllOutputNameInfo()`，兼容模块 `ModuResult` 反射读取（如圆查找 `CenterX/CenterY/Radius/FitError`）与 `FormatModuleTool.format_out`；无流程输出时回退模块汇总状态。
- **参数编辑**：管理员且空闲时启用官方 `VmParamsConfigControl`，直接绑定 `IVmModule`，支持 ROI、模板、位置修正与输入绑定；标量参数另有 JSON 预设；`ResetParam()` 恢复算子默认值。
- **运行环境定位**：`VisionMasterEnvironment` 按配置 → `VISIONMASTER_ROOT` → 常见路径查找安装目录，校验 `VM.Core.dll`，设置工作目录、`PATH` 与 `AppDomain.AssemblyResolve`，解决 `Private=false` 时的原生依赖解析。

### 3.4 工业相机（MVS SDK）

使用 `MvCameraControl.Net`、`MVDCore.Net`、`MVDImage.Net` 与原生 `MvCameraControl.dll`。实际调用的 API 包括：

```text
MV_CC_EnumDevices_NET / MV_CC_CreateDevice_NET / MV_CC_OpenDevice_NET
MV_CC_RegisterImageCallBackEx_NET / MV_CC_StartGrabbing_NET / MV_CC_StopGrabbing_NET
MV_CC_SetEnumValue_NET / MV_CC_SetFloatValue_NET / MV_CC_GetFloatValue_NET
MV_CC_SetIntValueEx_NET / MV_CC_GetIntValueEx_NET / MV_CC_SetBoolValue_NET / MV_CC_GetBoolValue_NET
MV_CC_SetPixelFormat_NET / MV_CC_SetImageNodeNum_NET / MV_CC_GetOptimalPacketSize_NET
MV_CC_TriggerSoftwareExecute_NET / MV_CC_IsDeviceConnected_NET
```

实现要点：回调内只拷贝帧并唤醒等待者，不在 SDK 回调里做算法或 UI；曝光/增益写入后回读校验，失败抛出 `CameraSdkException` 而不是静默成功；监控线程检测掉线并按配置间隔重连；预览抽帧与检测取图分离，避免 UI 拖慢采集。

### 3.5 设备通讯

| 通道 | 实现 | 能力 |
| --- | --- | --- |
| Modbus TCP | 自研主站，裸 TCP + 手写 MBAP/PDU | FC03 读保持寄存器、FC16 写多寄存器；结果/序号/耗时/自定义输出映射；超时与自动重连 |
| TCP | 主动连接与监听两种模式 | 结果模板发送；触发服务最多 32 客户端、每行 4096 字节，ACK 与命令解析 |
| 串口 | `System.IO.SerialPort` | ASCII / HEX 命令编解码，OK/NG/触发命令，接收触发检测 |

通讯与检测解耦：PLC 不可用不影响 OK/NG 判定与本地记录；每个通道独立捕获异常与重连；配置在检测停止边界热应用，失败回滚旧配置。未引入 NModbus、log4net 等第三方库，减少现场部署依赖。

### 3.6 数据与报表

- **配置**：JSON（`JavaScriptSerializer`），`appsettings.json` + `users.json`；带 SchemaVersion，配置对象集中校验。
- **检测日志**：按日 `detection-YYYY-MM-DD.csv`，固定字段（序号、时间、结果、方案、流程、耗时、输入源、图片路径、判定文本、输出摘要、错误）。
- **图片归档**：PNG，按 `年/月/日/OK|NG` 分目录；有界队列（默认 500）+ 单写线程 + 内存预算，满队列丢图但保留日志与结论。
- **报表**：手写 OpenXML（`[Content_Types].xml`、`workbook.xml`、`styles.xml`、`sheet1.xml`…）后用 `ZipArchive` 打包为真正的 `.xlsx`，含汇总与明细两张表；不依赖 Office/COM/ClosedXML。
- **日志**：自研文件日志，系统日志按天切分，审计日志独立；无 log4net。
- **目录**：数据根可配置；配置与系统日志默认在 `%ProgramData%\VisionInspect` 或 `%LocalAppData%\VisionInspect`。

### 3.7 安全与授权（交付版）

公开仓库只保留应用核心；交付版还包含以下产品化能力（实现细节不在本仓库）：

- 操作员/管理员两级权限，操作员口令使用 PBKDF2 加盐派生，管理员口令本地存储。
- 设备授权：机器指纹绑定，`.xreq` 申请 / `.xlic` 授权文件导入，RSA 签名校验。
- 方案包保护：加密方案包（`.xvp`）与完整性认证，支持客户参数策略。
- 发布保护：程序集混淆（Obfuscar 2.2.50）、发布签名与文件哈希清单、单实例互斥。
- 安装/卸载清理：仅清理应用自有授权与缓存目录，不删除系统运行库与驱动。

这些能力不改变“`.sol` 仍需官方 VM 运行许可”的事实。

### 3.8 构建、测试与发布

| 环节 | 工具/脚本 | 说明 |
| --- | --- | --- |
| 构建 | `scripts/Build.ps1`、`scripts/Check-Environment.ps1` | 校验 VS2022、.NET 4.7.2、VM/MVS SDK 路径后 MSBuild 构建 |
| 回归 | `Test-Core.ps1`、`Run-Regression.ps1`、`Test-*.ps1` | 自研控制台回归宿主，覆盖状态机、归档、CSV、通讯回环、主题叠加层 |
| CI | `.github/workflows/core-regression.yml` | `windows-latest` 上执行仓库检查 + 无 SDK 的核心/通讯回归 |
| 发布 | `Publish.ps1`、`Pack-XVisionSdk.ps1` | 生成应用目录与 SDK 包（`lib/net472`） |
| 安装包 | Python 生成 WiX 源 + WiX v6 + Burn | 按发布目录生成组件，链式安装并应用自定义 BA 主题（构建工程不在本仓库） |
| 手册 | Python ReportLab | 生成 7 页 PDF（构建脚本不在本仓库） |

## 4. 关键实现决策

1. **为什么必须单 STA 线程**：VM 方案、流程、模块参数不是并发安全的，底层还涉及 COM/原生组件。所有加载、运行、参数读写、关闭都投递到同一个 STA 线程 + WinForms 消息泵，通过 `Control.BeginInvoke` 排队，保证线程亲和性并避免并发崩溃。
2. **为什么用 `Run()` + 回调**：`SyncRun()` 会占住需要处理消息的 STA，真实 4.3.0 流程中可能长时间不返回。改为回调后 UI、相机预览与停止按钮在检测期间保持响应。
3. **为什么结果处理不阻塞检测**：UI 通知先于落盘/通讯；落盘与通讯使用 `CancellationToken.None`，停止时已产生的结果不丢；每步独立异常边界，后处理失败不改写 OK/NG。
4. **为什么自研 Modbus / XLSX**：工控机部署环境不可控，减少第三方 DLL 与 Office 依赖；协议与报表格式均由本仓库掌控，便于现场排障。
5. **为什么保留 `XVision` 内部命名**：现场注册表、数据目录、授权与脚本已按此部署；产品展示名改为 Perfect Vision，内部标识不变以避免升级破坏兼容性。

## 5. 运行环境与依赖

| 依赖 | 要求 |
| --- | --- |
| 操作系统 | Windows 10/11 x64（manifest 兼容 Win7/8.1 声明，实际以 VM/MVS 支持为准） |
| .NET | .NET Framework 4.7.2（安装包附带 4.8） |
| 视觉运行时 | VisionMaster 4.3.0 x64 及覆盖目标算法/版本的有效许可 |
| 相机 | MVS 驱动与 Runtime（`MVS\Runtime\Win64_x64`） |
| 原生依赖 | VC++ x64 运行库；VM 的 `Application(s)`、`Assembly`、`Module(sp)\x64` 需在 `PATH` 中 |
| 显示 | 支持 PerMonitorV2 的多显示器环境；覆盖层依赖分层窗口能力 |

## 6. 明确未使用的技术

为避免技术栈表述失真，以下技术在本次分析的 5.0.4 交付包中**没有**使用：

- **HALCON / OpenCV**：检测全部由 VM 执行；源码保留 `IAlgorithmEngine` 抽象与 `VmAlgorithmEngine` 实现，但未集成 HALCON HDevEngine 或 OpenCV。内部设计文档中的双引擎方案属于扩展规划，不是当前交付能力。
- **MySQL / SQL Server**：运行时数据为本地 JSON/CSV/PNG/XLSX，无数据库服务；仅开发者签发工具使用 Windows 自带 `winsqlite3.dll` 保存签名密钥，该工具不在本仓库。
- **Office / COM**：报表自行生成 OpenXML，不调用 Excel。
- **NModbus / log4net**：协议与日志均为自研实现。
- **Web / 云服务**：无 HTTP 上传、MES 或远程接口；授权与数据均在本地。

## 7. 证据索引

| 结论 | 证据文件 |
| --- | --- |
| 七工程分层与依赖方向 | `VisionInspect.sln`、各 `*.csproj` |
| x64 / 4.7.2 / C# 7.3 强制约束 | `Directory.Build.props`、`Directory.Build.targets` |
| VM SDK 引用与版本 | `src/VisionInspect.VisionMaster/VisionInspect.VisionMaster.csproj`、`src/VisionInspect.App/VisionInspect.App.csproj`；版本号取自随包官方 DLL 元数据 |
| STA 调度与 `Run()` 回调 | `src/VisionInspect.VisionMaster/Interop/VisionMasterDispatcher.cs`、`VisionMasterEngine.cs` |
| MVS API 调用 | `src/VisionInspect.Cameras/HikCameraService.cs` |
| Modbus FC03/FC16 | `src/VisionInspect.Communication/Modbus/ModbusTcpClient.cs` |
| TCP 触发与串口 | `src/VisionInspect.Communication/Triggers/TcpTriggerSource.cs`、`src/VisionInspect.Communication/Serial/SerialIoService.cs` |
| OpenXML 报表 | `src/VisionInspect.Infrastructure/Storage/ExcelReportService.cs` |
| 安装包链 | 交付包 `Perfect Vision.exe` 的 PE 元数据（内部名 `burn`、版本 5.0.4.0）及安装行为 |
| 权限与配置 | `SECURITY.md`、`config/appsettings.example.json` |

## 8. 分析边界

- 未运行真实产线方案、相机、PLC 与 168 小时连续性验收；结论来自代码、配置、安装包结构与随包手册。
- 未反编译混淆后的交付 DLL；行为描述以源码与手册为准。
- 海康 SDK 的具体算法能力不在本仓库范围内，本文只分析集成方式与调用边界。
