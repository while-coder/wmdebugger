# WMDebugger

WMDebugger 是面向 Unity 游戏的远程运行时调试与诊断工具。它通过一个自托管 Server 连接 Unity 客户端和调试界面，可用于查看实时日志、场景层级、GameObject 与组件、运行时对象、性能数据和画面预览，也支持在确认后修改对象、调用方法、执行 GM 命令和模拟 UGUI 输入。

- [在线使用](https://while-coder.github.io/wmdebugger/)
- [下载最新版本](https://github.com/while-coder/wmdebugger/releases/latest)

## 工作方式

```text
Unity 游戏客户端 ── WebSocket ── WMDebugger Server ── WebSocket ── 调试界面
                                                     ├─ 桌面客户端
                                                     ├─ VS Code 扩展
                                                     └─ 网页端
```

Server 负责转发 Unity 客户端与调试界面之间的消息，并保存 Server 密钥、HTTPS 证书和项目配置。桌面客户端、VS Code 扩展和网页端只是不同的调试入口，可以按需要任选一种。

## 快速开始

首次部署建议按以下顺序操作：

1. 从 [Releases](https://github.com/while-coder/wmdebugger/releases/latest) 下载 Server、Unity Package 和需要的调试客户端。
2. 使用 Docker 启动并初始化 WMDebugger Server。
3. 从调试界面复制 Server 公钥，将 Unity Package 接入项目并调用 `Debugger.Start`。
4. 运行 Unity 游戏，在桌面客户端、VS Code 或网页端进入同一个 Server。

### Release 文件说明

| 文件 | 用途 |
| --- | --- |
| `com.wm.debugger.zip` | Unity Package，解压后放入 Unity 工程 |
| `wm-debugger-server.tar` | 已构建的 WMDebugger Server Docker 镜像 |
| `wmdebugger-<version>.vsix` | VS Code 扩展 |
| `WMDebugger_<version>_windows_x64.exe` | Windows x64 桌面客户端安装包 |
| `WMDebugger_<version>_windows_arm64.exe` | Windows ARM64 桌面客户端安装包 |
| `WMDebugger_<version>_macos_universal.dmg` | macOS 安装包 |
| `WMDebugger_<version>_linux_x64.deb` | Debian / Ubuntu 安装包 |
| `WMDebugger_<version>_linux_x64.AppImage` | Linux 免安装版本 |
| `*.sig` | 桌面客户端自动更新使用的签名文件，普通安装不需要下载 |

建议同一次部署使用同一个 Release 中的 Server、Unity Package 和调试客户端。

## 1. 部署 Server

### 1.1 加载镜像

先安装并启动 Docker，然后执行：

```bash
docker load -i wm-debugger-server.tar
```

Release 镜像名称为 `wm-debugger-server:<version>`。可用下面的命令确认已加载的版本：

```bash
docker image ls wm-debugger-server
```

### 1.2 启动容器

Linux / macOS：

```bash
docker run -d \
  --name wm-debugger-server \
  --restart unless-stopped \
  -p 5800:5800 \
  -p 5801:5801 \
  -e LOG_LEVEL=info \
  -v wm-debugger-data:/root/.wm-debugger \
  wm-debugger-server:<version>
```

Windows PowerShell：

```powershell
docker run -d `
  --name wm-debugger-server `
  --restart unless-stopped `
  -p 5800:5800 `
  -p 5801:5801 `
  -e LOG_LEVEL=info `
  -v wm-debugger-data:/root/.wm-debugger `
  wm-debugger-server:<version>
```

将 `<version>` 替换为下载的 Release 版本，例如 `0.2.2`。

### 1.3 参数说明

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| `LOG_LEVEL` | `info` | Server 日志级别，例如 `debug`、`info`、`warn`、`error` |
| `/root/.wm-debugger` | — | Server 数据目录，保存 RSA 私钥、HTTPS 证书和项目配置，必须挂载持久化卷 |

容器内监听端口固定为 HTTP `5800`、HTTPS `5801`，不支持环境变量修改。`-p` 左侧是宿主机端口，右侧必须是 `5800`/`5801`。例如想通过宿主机 `8080` 访问 Server，可以使用 `-p 8080:5800`；调试界面和 Unity 中填写宿主机的 `8080`。

启动后检查状态：

```bash
docker logs wm-debugger-server
curl http://127.0.0.1:5800/health
```

正常情况下 `/health` 会返回 `ok: true`。首次启动时 `initialized` 为 `false`，完成下一步初始化后会变成 `true`。

### 1.4 首次初始化 Server

Server 使用 RSA 密钥证明自己的身份，Unity 客户端只接受持有对应私钥的 Server。首次启动后需要初始化一次：

1. 生成一份未加密的 RSA 私钥（至少 2048 位）：

   ```bash
   openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out wm-debugger-private-key.pem
   ```

2. 打开桌面客户端、VS Code 扩展或 Server 自带页面，例如 `http://127.0.0.1:5800`。首次远程初始化不建议使用 GitHub Pages 网页端，因为 HTTPS 页面通常无法连接尚未配置 HTTPS 的远程 Server。
3. 输入 Server 地址，例如 `http://127.0.0.1:5800`，点击“新增”，再点击“进入”。
4. 页面提示初始化时，选择刚生成的 `wm-debugger-private-key.pem` 并上传。
5. 进入“设置 → 服务配置 → Server 身份认证”，点击“加载公钥”并复制公钥，下一节接入 Unity 时需要使用。

私钥只应在可信网络中上传。它会保存在 Docker 卷 `wm-debugger-data` 中；不要丢失该卷，也不要把私钥提交到代码仓库。更换私钥会使公钥变化，所有 Unity 客户端都必须同步更新。

### 1.5 可选：启用 HTTPS

远程或公网部署建议使用 HTTPS：

1. 将域名解析到 Server，并放通映射后的 HTTP/HTTPS 端口。
2. 先通过可信网络进入“设置 → 服务配置 → HTTPS 证书”。
3. 上传匹配的私钥和证书，Server 会立即启用 HTTPS 端口（`5801`）。
4. 后续调试界面使用 `https://<域名>:<HTTPS端口>`，Unity 调用 `Debugger.Start` 时将 `serverHttps` 设为 `true`，并传入 HTTPS 端口。

浏览器从 HTTPS 页面连接 Server 时，Server 也必须使用 HTTPS，否则浏览器会阻止不安全的 HTTP/WebSocket 混合内容。Server 当前不提供用户账号或访问令牌认证，不应直接暴露到公网；请通过防火墙、VPN 或带访问控制的反向代理限制访问范围。WMDebugger 的运行时修改、方法调用、GM 和输入模拟都属于高权限操作。

## 2. 接入 Unity

### 2.1 安装 Unity Package

解压 `com.wm.debugger.zip`，将其中的 `com.wm.debugger` 目录完整复制到 Unity 工程：

```text
YourUnityProject/
└─ Packages/
   └─ com.wm.debugger/
      ├─ package.json
      └─ Runtime/
```

等待 Unity 完成导入和编译。不要只复制 `Runtime` 目录，压缩包最外层的 `com.wm.debugger` 就是完整 Package。

当前 Package 的 `WMDebugger.asmdef` 直接引用 `Wx` 和 `TTWebGL`，用于微信与抖音小游戏适配。接入工程需要提供这两个程序集；如果项目不使用对应小游戏 SDK，请先根据项目的平台依赖调整 asmdef，否则 Unity 会报告程序集引用不存在。

构建抖音小游戏或微信小游戏平台时，还需要在 Player Settings（或对应的构建脚本）中定义宏：

| 平台 | 宏定义 |
| --- | --- |
| 抖音小游戏 | `WM_DYMINI` |
| 微信小游戏 | `WM_WXMINI` |

定义宏后，目录浏览、文件读写和偏好设置会改用对应小游戏 SDK（`TTSDK` / `WeChatWASM`）的文件系统与存储接口；未定义宏时使用默认实现。编辑器下始终使用默认实现，宏只在真机构建生效。两个宏不要同时定义。

### 2.2 启动调试连接

在游戏初始化位置调用：

```csharp
using WMDebugger;

public static class DebuggerBootstrap
{
    public static void Start()
    {
        Debugger.Start(
            serverHost: "debug.example.com",
            serverPort: 5800,
            serverPublicKey: "从 Server 身份认证页面复制的公钥",
            serverHttps: false,
            persistentUUID: true
        );
    }
}
```

参数说明：

| 参数 | 说明 |
| --- | --- |
| `serverHost` | Server 主机名或 IP，不要包含 `http://`、`https://` 或路径 |
| `serverPort` | Unity 实际访问的 Server 端口；HTTP 和 HTTPS 端口不要混用 |
| `serverPublicKey` | 从调试界面“设置 → 服务配置 → Server 身份认证”复制的 Base64 公钥 |
| `serverHttps` | HTTP/WS 使用 `false`，HTTPS/WSS 使用 `true` |
| `persistentUUID` | `true` 时 UUID 写入 PlayerPrefs，重启后仍识别为同一个客户端；默认建议保持 `true` |

本机测试可使用：

```csharp
Debugger.Start("127.0.0.1", 5800, "复制的公钥", false, true);
```

如果 Unity 运行在手机或其他设备上，`127.0.0.1` 指向设备自身，必须改为设备能够访问的 Server 局域网 IP 或域名。

### 2.3 可选的客户端信息和高权限功能

登录玩家后可以补充客户端列表中的玩家信息：

```csharp
Debugger.PlayerId = playerId;
Debugger.PlayerName = playerName;
Debugger.SetInfo(
    ("environment", "test"),
    ("version", Application.version)
);
```

如果项目提供 GM 执行入口，可显式接入：

```csharp
Debugger.ExecuteCommand = async command =>
{
    return await YourGmSystem.ExecuteAsync(command);
};
```

UGUI 输入模拟默认开启，并会真实改变客户端状态。正式环境不允许远程输入时，应在 `Debugger.Start` 前关闭：

```csharp
Debugger.EnableInputSimulation = false;
```

### 2.4 接收 Server 运行时配置

调试界面的“Unity配置”按 `Application.identifier` 保存 JSON 对象。Unity 验证 Server 并登录后会自动收到当前配置；前端再次保存或清空时，在线客户端也会立即更新。回调在 Unity 主线程执行：

WMDebugger 当前内置 `SnapshotGzipEnabled` 快照配置，控制是否使用 GZIP，默认为 `true`：

```json
{
  "SnapshotGzipEnabled": true
}
```

```csharp
void ApplyDebuggerConfig(RuntimeConfig config)
{
    Debug.Log($"快照 GZIP：{config.SnapshotGzipEnabled}");
}

Debugger.RuntimeConfigChanged += ApplyDebuggerConfig;
Debugger.Start("127.0.0.1", 5800, "复制的公钥");
```

当前配置对象始终可以通过 `Debugger.RuntimeConfig` 读取。建议先订阅事件再调用 `Debugger.Start`。

建议只在开发、测试或明确授权的诊断构建中启用 WMDebugger，并按项目需要限制可执行的 GM 命令和可修改成员。

## 3. 打开调试界面

三种入口功能使用同一套 Server 地址，选择一种即可。

### 桌面客户端

从 [Releases](https://github.com/while-coder/wmdebugger/releases/latest) 下载当前系统对应的安装包：

- Windows：`WMDebugger_<version>_windows_x64.exe` 或 ARM64 版本
- macOS：`WMDebugger_<version>_macos_universal.dmg`
- Debian / Ubuntu：`WMDebugger_<version>_linux_x64.deb`
- 其他 Linux：`WMDebugger_<version>_linux_x64.AppImage`

安装并打开后，输入 Server 的完整 HTTP 或 HTTPS 地址，点击“新增”，然后从列表中点击“进入”。新增只保存地址，不会立即进入 Server。

### VS Code 扩展

1. 下载 `wmdebugger-<version>.vsix`。
2. 在 VS Code 扩展页面右上角菜单中选择“从 VSIX 安装...”。
3. 安装完成后执行命令“WMDebugger: 打开调试器”，或按 `Ctrl+Shift+D`；macOS 使用 `Cmd+Shift+D`。
4. 新增并进入 Server。

### 网页端

直接打开：

<https://while-coder.github.io/wmdebugger/>

网页端不需要安装，但浏览器安全策略更严格：HTTPS 网页不能连接纯 HTTP Server。远程使用网页端时，请先为 Server 配置 HTTPS；本机开发也可以直接打开 Server 自带页面 `http://127.0.0.1:5800`。

## 4. 开始调试

Unity 游戏运行并完成连接后，调试界面的客户端列表会显示对应实例。进入客户端后可按当前版本提供的功能进行：

- 查看实时日志、警告与错误，并按关键词过滤
- 查看实时画面、Scene 层级、GameObject 和组件信息
- 浏览和修改运行时对象、字段与属性，调用可用方法
- 查看帧率、内存、GC、帧耗时和卡顿标记
- 浏览 PlayerPrefs、目录和应用信息
- 执行项目接入的 GM 命令
- 在明确启用后模拟 UGUI 点击与拖拽
- 配置后使用 AI 调试和离线日志分析

项目显示名、GM 命令、常用对象、打点别名和日志分析规则等配置按 Unity `Application.identifier` 保存在 Server 中，同一 Server 的团队成员可以共享。桌面行为、表格显示和 AI 连接信息等个人设置只保存在当前设备。

## 常见问题

### Server 已启动，但页面显示“尚未初始化”

这是首次启动的正常状态。按照“首次初始化 Server”上传 RSA 私钥即可。若容器重建后再次要求初始化，通常是没有挂载或保留 `/root/.wm-debugger` 数据卷。

### Unity 客户端没有出现

依次检查：

1. `/health` 是否返回 `initialized: true`。
2. Unity 设备是否能访问填写的主机和端口。
3. `serverHttps` 是否与端口协议一致。
4. Unity 中的 `serverPublicKey` 是否与当前 Server 公钥一致。
5. 容器端口、防火墙、安全组或反向代理是否允许 HTTP(S) 和 WebSocket。

### Unity 提示 Server 身份验证失败

当前 Server 私钥与 Unity 中配置的公钥不匹配。到“设置 → 服务配置 → Server 身份认证”重新加载公钥，并更新 Unity 配置后重新构建或启动。

### 网页端无法连接 HTTP Server

GitHub Pages 使用 HTTPS，浏览器会阻止它访问不安全的 HTTP/WebSocket 地址。请为 Server 启用 HTTPS，或改用桌面客户端、VS Code 扩展、Server 自带的 HTTP 页面。

### 如何升级

下载新 Release 后：

1. 使用新的 `com.wm.debugger.zip` 替换 Unity Package 并重新构建游戏。
2. `docker load` 新的 Server 镜像，删除旧容器并用相同参数、相同数据卷重新创建。
3. 安装新版桌面客户端或 VSIX；网页端会随 Release 自动更新。

只要继续挂载原来的 `wm-debugger-data`，Server 私钥、HTTPS 证书和项目配置都会保留。
