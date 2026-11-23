# 实现笔记：VisionMaster 4.3 接入

本笔记记录 Perfect Vision 接入 VisionMaster 4.3 的实现方式与关键取舍，供后续维护和版本升级参考。

## 运行环境

- Windows x64，C# 7.3，.NET Framework 4.7.2，VS2022。
- 海康机器人 VisionMaster 4.3.0，官方程序集位于 `Development\V4.x\ComControls\Assembly`。
- 海康 MVS SDK：`MvCameraControl.Net`、`MVDCore.Net`、`MVDImage.Net` 及原生 `MvCameraControl.dll`。

未升级到 .NET 6/8 的原因在于，VM 4.3 的托管程序集与 WinForms 控件均面向 .NET Framework 编译。迁移到更高版本的运行时需要增加 COM 宿主层，或等待海康发布对应版本。工控机环境对依赖数量较为敏感，因此未引入额外的运行时适配层。

仓库不包含海康 DLL 与安装包。编译前需自行安装 VM 与 MVS，并通过 `VisionMasterRoot`、`MvsSdkDir` 将路径传给 `Build.ps1`。

## VM 接入

### 线程模型

VM 的方案、流程与模块参数不具备并发安全性，其底层还涉及 COM 与原生对象。实现方式是创建一条 STA 线程，在该线程上建立隐藏消息窗口并运行 WinForms 消息泵；所有 VM 调用通过 `Control.BeginInvoke` 排队执行。方案加载、流程运行、参数读写、保存与关闭均在该线程上串行完成。

代码位于 `VisionInspect.VisionMaster/Interop/VisionMasterDispatcher.cs` 与 `VisionMasterEngine.cs`。

第一版实现按照官方示例使用了 `procedure.SyncRun()`。该方式在测试方案上运行正常，但在模块数量较多的方案中，启动后界面无响应，日志停留在 `Run()` 调用处。原因是 `SyncRun()` 占用了需要处理消息的 STA 线程，VM 内部消息泵无法运转，形成对自身的等待。

当前实现拆分为两个阶段：

```csharp
// STA 线程：完成图像注入并启动流程，随后返回消息泵
source.ModuParams.ImageSourceType = ImageSourceParam.ImageSourceTypeEnum.SDK;
source.SetImageData(image, true);
procedure.Run();

// 普通线程：等待流程完成回调
var completed = new TaskCompletionSource<bool>();
procedure.OnWorkEndStatusCallBack += ...;

// 回调到达后，回到 STA 线程读取结果并释放 ImageBaseData
```

等待过程使用 `TaskCompletionSource`，不使用 `.Wait()` 或 `.Result`，否则会再次阻塞 STA 线程。

`RunTimeoutMs` 默认 30 秒。超时仅标记状态，并禁止重跑或切换方案，不强制释放方案。原因是迟到回调仍会读取 `ModuResult`，此时方案已被释放会导致进程崩溃。相比直接释放，保留上下文并报告超时更为可控。

### 方案与模块

```csharp
VmSolution.Load(path, password, false);
var procedures = VmSolution.Instance.GetAllProcedureList();
VmProcedure procedure = VmSolution.Instance["流程1"] as VmProcedure;
var modules = procedure.GetProcedureModuleList();
object module = VmSolution.Instance["流程1.图像源1"];
```

模块需使用“流程名.模块名”的全名定位。方案中修改模块名称后，参数预设与默认值会失效，这是现场常见问题之一。

### 图像注入

相机帧与本地图片最终都转换为 `RawImage`，支持 Mono8、RGB24 与 BGR24。BGR 在适配层转换为 RGB24，再构造 `ImageBaseData`：

```csharp
new ImageBaseData(buffer, length, width, height, VMPixelFormat.VM_PIXEL_MONO_08)
```

本地图片同样通过 SDK 注入，不使用 VM 的“本地图像”模式。因此本地图片与相机帧共用同一条检测链路，日志中记录的文件名与实际执行的图像能够对应。

### 结果读取

优先读取流程输出：

```csharp
foreach (var info in procedure.ModuResult.GetAllOutputNameInfo()) {
    switch (info.TypeName) {
        case IMVS_MODULE_BASE_DATA_TYPE.IMVS_GRAP_TYPE_INT: ...
        case IMVS_MODULE_BASE_DATA_TYPE.IMVS_GRAP_TYPE_FLOAT: ...
        case IMVS_MODULE_BASE_DATA_TYPE.IMVS_GRAP_TYPE_STRING: ...
    }
}
```

现场方案经常未配置流程输出，因此保留了两条备用路径：从模块的 `ModuResult` 反射读取叶子节点（圆查找的 `CenterX`、`CenterY`、`Radius`、`FitError` 即由此获得），以及读取格式化模块的 `format_out`。

判定顺序位于 `ResultJudge`：

1. 查找名为“检测结果”的流程输出，比较 OK/NG；
2. 若不存在，则扫描全部文本，任一 NG 优先判定为 NG；
3. 回退到 `Procedure.IsRunOK`；
4. 最后回退到所有未禁用模块的 `IsRunOK` 汇总，界面标注“模块状态”。

第 4 条用于处理“模块全部通过但流程未配置输出”的方案，不能作为正常配置使用。

### 参数编辑

参数页使用 VM 官方 `VmParamsConfigControl`，通过 `ModuleSource = (IVmModule)VmSolution.Instance[fullName]` 绑定。ROI、模板、位置修正和输入绑定均由 VM 处理，本软件不重新实现。

参数修改仅允许管理员在空闲状态下进行。检测期间修改参数曾导致原生层崩溃并退出进程，因此该限制不可放宽。

标量参数另存为 JSON 预设，仅包含 int、float、bool、string、enum 等可读写的简单属性。ROI 与模板不写入 JSON，完整配方仍通过 `.sol` 保存。

“恢复默认”调用 `VmModule.ResetParam()`，恢复的是算子算法默认值，而非“上次加载方案时的值”。界面会提示先保存 `.sol` 备份。

### 运行环境定位

`VisionMasterEnvironment` 负责查找安装目录、切换工作目录、将 `Application(s)`、`Assembly`、`Module(sp)\x64` 加入 `PATH`，并注册 `AppDomain.AssemblyResolve`。工程中的 VM 引用设置为 `Private=false`，输出目录不包含完整 VM DLL，需要由运行时从安装目录解析。

## 相机

MVS 的调用流程较为直接，主要复杂度集中在生命周期管理。

回调内仅执行三项操作：拷贝当前帧、唤醒等待中的检测线程、按 `PreviewFrameSkip` 决定是否发布预览。算法与界面逻辑均不进入回调；MVS 回调对耗时较为敏感，阻塞会导致丢帧。

曝光与增益在写入后必须回读：

```csharp
MV_CC_SetFloatValue_NET(handle, "ExposureTime", value);
MV_CC_GetFloatValue_NET(handle, "ExposureTime", out actual);
```

早期版本曾将写入失败静默忽略，界面显示 5000 us，相机仍处于自动曝光，图像亮度反复变化。当前实现在数值不一致时抛出 `CameraSdkException`，通过状态栏提示现场人员。

掉线重连由独立线程负责，按 `ReconnectIntervalMs` 调用 `MV_CC_IsDeviceConnected_NET`。网线与 USB 断开的情况均已测试，连接可自动恢复；正在执行的那一轮检测会失败，属于预期行为。

## 通讯

### Modbus TCP

Modbus TCP 未引入 NModbus，采用裸 TCP 与手写 MBAP/PDU 实现。主要目的是减少第三方依赖：工控机部署环境对 DLL 版本冲突较为敏感，且协议问题出现时，自行实现的报文更便于排查。

停止时存在一个问题：读线程阻塞在 socket 上，`CancellationToken` 无法取消。当前实现是在停止时关闭本次连接的 socket，使阻塞读抛出 `IOException`，再等待线程退出。连接不可复用，重连时重新建立。

结果寄存器布局为：偏移 0 表示 OK/NG（默认 1/2），偏移 1 为序号低 16 位，偏移 2 为耗时，后续按 `Mappings` 映射测量值。字节序与字序需按 PLC 手册核对，不能按经验推断。

### TCP

TCP 支持两种模式：主动连接下位机，或监听产线触发。触发服务最多支持 32 个客户端，每行 4096 字节，按行解析。ACK 仅表示收到命令，检测是否被接受仍需检查登录状态、空闲状态与方案状态。

### 串口

串口使用 `System.IO.SerialPort`，提供 ASCII 与 HEX 两套编解码。ASCII 支持 `\r\n` 转义，HEX 使用空格分隔。接收触发与发送结果可分别启用。

三个通道各自捕获异常并独立重连。PLC 故障不影响本地判定与记录，仅影响产线联锁。该边界需在项目初期与自动化团队确认。

配置热更新统一按“停止检测 → 应用配置 → 失败回滚 → 保存配置”的顺序执行。若顺序颠倒，检测结果可能写入旧连接或丢失。

## 数据

- 检测日志按天写入 CSV，字段固定，可直接用文本编辑器打开。
- 图片保存为 PNG，按“年/月/日/OK|NG”目录组织；写入采用有界队列与单写线程，队列满时丢弃图片但保留记录。
- 报表使用手写 OpenXML 实现，通过 `ZipArchive` 打包为 `.xlsx`，不依赖 Office，也不引入 ClosedXML。
- 配置使用 JSON，`JavaScriptSerializer` 已满足需求。

未使用数据库。当前数据量用文件系统即可满足，备份只需复制目录，现场无需额外维护数据库服务。

## 界面

主界面、设置和登录窗口均基于 WinForms，控件以自绘为主，日夜主题实现在 `Theme.cs` 与 `Theme.Animation.cs` 中。VM 的参数编辑器为 WPF 实现，因此通过 `ElementHost` 嵌入，工程引用 `PresentationCore`、`PresentationFramework` 和 `WindowsBase`。

透明叠加层经历了多次调整。最初使用半透明 WinForms 子控件覆盖 `VmRenderControl`，实际显示为整块黑色，遮挡了 VM 的原生渲染。当前改为独立顶层窗口，通过 `TransparencyKey`、`WS_EX_TRANSPARENT` 与 `HTTRANSPARENT` 实现视觉透明与鼠标透传；缩放和平移仍由底层 VM 控件接收。

主题切换时，覆盖层需重新同步位置与透明键，具体复盘见 `THEME_OVERLAY_FIX.md`。

## 交付

- 构建：`Build.ps1`，MSBuild Release/x64。
- 回归：`Test-Core.ps1` 与通讯的 `Run-Regression.ps1`，不需要 VM 授权。
- CI：GitHub Actions 执行仓库检查与上述两组回归，运行环境为 `windows-latest`。
- 安装包：WiX Burn，依次安装 .NET Framework 4.8、VC++ 运行库、MVS 驱动和客户端 MSI。
- 手册：Python ReportLab 生成 PDF。

安装包、设备授权、加密方案包、混淆与签名工具不在公开仓库中。相关模块或包含私钥，或属于商业交付代码，不适合与源码一同公开。

## 未实现的部分

- 未接入 HALCON 与 OpenCV。检测仅由 VM 执行；源码保留 `IAlgorithmEngine` 抽象，但目前只有 `VmAlgorithmEngine` 一个实现。
- 未使用 MySQL 或 SQL Server，运行时数据均为本地文件。
- 未实现 Modbus RTU。串口通道不等同于 RTU。
- 未提供 MES 与 HTTP 上传。
- 真实产线的 168 小时连续运行测试尚未完成。现有验证仅代表测试环境下的结果。

## 版本

| 程序集 | FileVersion |
| --- | --- |
| `VM.Core.dll` | 4.3.0.1373494 |
| `VM.PlatformSDKCS.dll` | 4.3.0.1335825 |
| `VMControls.Winform.Release.dll` | 4.3.0.1262187 |
| `ImageSourceModuleCs.dll` / `FormatModuleCs.dll` | 1.0.0.0 |

更换 VM 版本后，应使用 VS 对象浏览器重新核对 `VmSolution`、`VmProcedure`、`ImageSourceModuleTool` 与 `VmParamsConfigControl` 的签名。海康不同小版本的接口可能变化，官方示例不一定适用于目标版本。
