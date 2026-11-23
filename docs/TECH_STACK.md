# 实现笔记：VM 4.3 是怎么接进来的

这份笔记写给以后接手的人，也写给我自己。隔几个月回头看代码，最容易被忘掉的不是语法，是当时为什么这么写。

## 环境

- Windows x64，C# 7.3，.NET Framework 4.7.2，VS2022。
- 海康机器人 VisionMaster 4.3.0，官方程序集在 `Development\V4.x\ComControls\Assembly`。
- 海康 MVS SDK，`MvCameraControl.Net`、`MVDCore.Net`、`MVDImage.Net` 加原生 `MvCameraControl.dll`。

没升 .NET 6/8，原因很简单：VM 4.3 的托管 DLL 和 WinForms 控件都是 .NET Framework 的，硬搬要么加 COM 宿主，要么等海康升级。工控机上少一层是一层。

仓库里没有海康的 DLL，也没有安装包。编译前自己装 VM 和 MVS，路径通过 `VisionMasterRoot`、`MvsSdkDir` 传给 `Build.ps1`。

## VM：一根 STA 线程，所有调用排队

VM 的方案、流程、模块参数都不是线程安全的，底层还有 COM 和原生对象。我的做法是开一根 STA 线程，建隐藏消息窗口，跑 WinForms 消息泵。所有 VM 调用都用 `Control.BeginInvoke` 投进去，加载、运行、读写参数、保存、关闭，一律排队。

代码在 `VisionInspect.VisionMaster/Interop/VisionMasterDispatcher.cs` 和 `VisionMasterEngine.cs`。

### 为什么不用 SyncRun

第一版按官方示例用了 `procedure.SyncRun()`。测试方案没问题，换到模块多的方案，点启动界面就卡死，日志停在 `Run()`。原因是 `SyncRun()` 占住了需要处理消息的 STA，VM 内部消息泵转不起来，自己等自己。

现在拆成两段：

```csharp
// STA 线程：只做注入和启动，马上回消息泵
source.ModuParams.ImageSourceType = ImageSourceParam.ImageSourceTypeEnum.SDK;
source.SetImageData(image, true);
procedure.Run();

// 普通线程：等回调
var completed = new TaskCompletionSource<bool>();
procedure.OnWorkEndStatusCallBack += ...;

// 回调到了，再回 STA 读结果、释放 ImageBaseData
```

等待用 `TaskCompletionSource`，不用 `.Wait()` 或 `.Result`，否则又会把 STA 堵住。

`RunTimeoutMs` 默认 30 秒。超时只标记状态、禁止重跑和切方案，不强行释放。原因是迟到回调还会回来读 `ModuResult`，这时候方案已经被释放，进程就没了。宁可让操作员看见“流程超时”，也不能赌。

### 方案和模块怎么找

```csharp
VmSolution.Load(path, password, false);
var procedures = VmSolution.Instance.GetAllProcedureList();
VmProcedure procedure = VmSolution.Instance["流程1"] as VmProcedure;
var modules = procedure.GetProcedureModuleList();
object module = VmSolution.Instance["流程1.图像源1"];
```

模块要用“流程名.模块名”的全名定位。方案里模块改名，参数预设和默认值会对不上，这是现场最常见的问题之一。

### 图像注入

相机帧和本地图片最后都转成 `RawImage`，支持 Mono8、RGB24、BGR24。BGR 在适配层转成 RGB24，再构造 `ImageBaseData`：

```csharp
new ImageBaseData(buffer, length, width, height, VMPixelFormat.VM_PIXEL_MONO_08)
```

本地图片也走 SDK 注入，不切 VM 的“本地图像”模式。这样本地和相机是同一条链路，日志里记的文件名和实际跑的图能对上。

### 结果读取

优先读流程输出：

```csharp
foreach (var info in procedure.ModuResult.GetAllOutputNameInfo()) {
    switch (info.TypeName) {
        case IMVS_MODULE_BASE_DATA_TYPE.IMVS_GRAP_TYPE_INT: ...
        case IMVS_MODULE_BASE_DATA_TYPE.IMVS_GRAP_TYPE_FLOAT: ...
        case IMVS_MODULE_BASE_DATA_TYPE.IMVS_GRAP_TYPE_STRING: ...
    }
}
```

现场方案经常不配流程输出。所以还留了两条后路：从模块的 `ModuResult` 反射读叶子节点（圆查找的 `CenterX`、`CenterY`、`Radius`、`FitError` 就是这么来的），以及格式化模块的 `format_out`。

判定顺序在 `ResultJudge`：

1. 找名为“检测结果”的流程输出，比较 OK/NG；
2. 没有就扫全部文本，任意 NG 优先；
3. 再退到 `Procedure.IsRunOK`；
4. 最后退到所有未禁用模块的 `IsRunOK` 汇总，界面标“模块状态”。

第 4 条是给那些“模块全过但流程没配输出”的方案兜底的，不能当成正常配置。

### 参数

参数页用的是 VM 官方的 `VmParamsConfigControl`，直接 `ModuleSource = (IVmModule)VmSolution.Instance[fullName]`。ROI、模板、位置修正、输入绑定都由 VM 自己处理，我不碰。

只允许管理员在空闲状态改。检测中改参数，轻则下一次运行不生效，重则原生层直接崩，这个亏吃过一次。

标量参数另存 JSON 预设，只存 int/float/bool/string/enum 这些可读写的简单属性。ROI 和模板不进 JSON，完整配方还是靠 `.sol`。

“恢复默认”调用 `VmModule.ResetParam()`，恢复的是算子算法默认值，不是“上次加载方案时的值”。界面上会提示先保存 `.sol` 备份。

### 运行环境

`VisionMasterEnvironment` 负责找安装目录、切工作目录、把 `Application(s)`、`Assembly`、`Module(sp)\x64` 加进 `PATH`，再挂一个 `AppDomain.AssemblyResolve`。因为工程里 VM 引用是 `Private=false`，输出目录不带一整套 VM DLL，得让运行时自己去安装目录找。

## 相机

MVS 的调用不复杂，麻烦的是生命周期。

回调里只做三件事：拷贝当前帧、唤醒等帧的检测线程、按 `PreviewFrameSkip` 决定要不要发预览。算法和 UI 都不进回调，MVS 的回调线程不能堵，堵了就开始丢帧。

曝光和增益写入后一定回读：

```csharp
MV_CC_SetFloatValue_NET(handle, "ExposureTime", value);
MV_CC_GetFloatValue_NET(handle, "ExposureTime", out actual);
```

早期版本把写入失败吞了，界面显示 5000 us，相机还在自动曝光，图像一亮一暗。现在对不上就抛 `CameraSdkException`，让状态栏红着，比假装成功强。

掉线重连是单独线程，按 `ReconnectIntervalMs` 查 `MV_CC_IsDeviceConnected_NET`。拔网线、拔 USB 都试过，能回来，但正在跑的那轮会失败，这个要跟现场说清楚。

## 通讯

### Modbus TCP

自己写的，裸 TCP 加 MBAP/PDU。只实现 FC03 和 FC16，够接常见 PLC。

停止时有个坑：读线程阻塞在 socket 上，`CancellationToken` 取消不了。现在的做法是直接关掉这次连接的 socket，让阻塞读抛 `IOException`，再等线程退出。连接不能复用，重连时重新建。

结果寄存器布局：偏移 0 是 OK/NG（默认 1/2），偏移 1 是序号低 16 位，偏移 2 是耗时，后面按 `Mappings` 映射测量值。字节序和字序要按 PLC 手册核对，不能想当然。

### TCP

两种模式：主动连下位机，或者监听产线触发。触发服务最多 32 个客户端，每行 4096 字节，按行解析。ACK 只代表收到命令，检测有没有被接受还要看登录、空闲和方案状态。

### 串口

`System.IO.SerialPort`，ASCII 和 HEX 两套编解码。ASCII 支持 `\r\n` 转义，HEX 用空格分隔。接收触发和发送结果可以分开开关。

三个通道各自独立异常、独立重连。PLC 挂了不影响本地判定和记录，只影响产线联锁。这个边界要提前跟自动化的人对齐，不然出了事说不清。

配置热更新统一走：停检测 → 应用新配置 → 失败回滚 → 保存。反过来的顺序会丢结果。

## 数据

- 检测日志按天写 CSV，字段固定，记事本能打开。
- 图片存 PNG，按 `年/月/日/OK|NG`，有界队列加单写线程，队列满了丢图但保留记录。
- 报表是手写 OpenXML，再用 `ZipArchive` 打包成 `.xlsx`。不依赖 Office，也不引 ClosedXML。
- 配置是 JSON，`JavaScriptSerializer` 就够了。

没上数据库。这个量级用文件足够，备份就是拷目录，现场不用再多维护一个服务。

## 界面

WinForms 为主，自绘控件加日夜主题。VM 的参数编辑器是 WPF，用 `ElementHost` 嵌进来，所以工程引了 `PresentationCore`、`PresentationFramework`、`WindowsBase`，仅此而已。

透明叠加层绕了一圈。半透明 WinForms 子控件盖在 `VmRenderControl` 上会整块变黑，后来改成独立顶层窗口，`TransparencyKey` 加 `WS_EX_TRANSPARENT`、`HTTRANSPARENT`，视觉透明、鼠标透传。缩放和平移还是底下 VM 控件处理。

主题切换时覆盖层要重新同步位置和透明键，这个在 `THEME_OVERLAY_FIX.md` 里有复盘。

## 交付

- 构建：`Build.ps1`，MSBuild Release/x64。
- 回归：`Test-Core.ps1` 和通讯的 `Run-Regression.ps1`，不需要 VM 授权。
- CI：GitHub Actions 跑仓库检查加这两组回归，`windows-latest`。
- 安装包：WiX Burn，链式装 .NET Framework 4.8、VC++、MVS 驱动、客户端 MSI。
- 手册：Python ReportLab 生成 PDF。

安装包、设备授权、加密方案包、混淆和签名工具不在公开仓库。原因不复杂：里面要么有私钥，要么是商业交付代码，不能跟着源码一起公开。

## 没做的事

- 没接 HALCON、OpenCV。检测只有 VM，源码里留了 `IAlgorithmEngine` 抽象，但只有 `VmAlgorithmEngine` 一个实现。
- 没上 MySQL、SQL Server。运行时全是本地文件。
- 没做 Modbus RTU。串口通道不等于 RTU。
- 没做 MES、HTTP 上传。
- 没做完真实产线的 168 小时拷机。现有验证只能说明测试环境下跑过。

## 版本

| 程序集 | FileVersion |
| --- | --- |
| `VM.Core.dll` | 4.3.0.1373494 |
| `VM.PlatformSDKCS.dll` | 4.3.0.1335825 |
| `VMControls.Winform.Release.dll` | 4.3.0.1262187 |
| `ImageSourceModuleCs.dll` / `FormatModuleCs.dll` | 1.0.0.0 |

换 VM 版本之后，至少用 VS 的对象浏览器把 `VmSolution`、`VmProcedure`、`ImageSourceModuleTool`、`VmParamsConfigControl` 这几个类重新对一遍。海康不同小版本的签名会变，官方示例也不一定跟得上。
