# ecarx-bridge — ECARX 车机方向盘切歌桥接

> 一个开机自启、后台常驻的 Android APK：监听 ECARX 车机媒体中心打印的方向盘媒体键日志，以 `shell` 身份重新注入 `KEYCODE_MEDIA_NEXT / PREVIOUS`，让 CarMore / CarLife 在方向盘切歌时真正响应。**运行时不需要电脑、不需要 root。**

适用环境：Android 9 / SDK 28 的 ECARX 车机，SELinux Permissive，adbd 监听 TCP 回环 `127.0.0.1:5555`。如果你的车机不满足这些前置条件，请先读 [实施局限性](#三实施局限性) 和 [前置条件](#四前置条件)。

---

## 目录

- [一、必要性：为什么需要它](#一必要性为什么需要它)
- [二、原理：它是怎么工作的](#二原理它是怎么工作的)
- [三、实施局限性](#三实施局限性)
- [四、前置条件](#四前置条件)
- [五、构建与安装](#五构建与安装)
- [六、运行与验证](#六运行与验证)
- [七、目录结构](#七目录结构)
- [八、风险与免责](#八风险与免责)

---

## 一、必要性：为什么需要它

在这类 ECARX 车机上，方向盘「上一曲 / 下一曲」产生的**原始广播会被系统媒体中心 `ecarx.xsf.mediacenter` 拦截并消费**，CarMore / CarLife 等第三方车机应用收不到。结果是：用 CarMore 听歌时，按方向盘切歌键没有任何反应。

抓 `dumpsys media_session` 可以看到问题本质：

- `mediacenter` 的 MediaSession 处于 `active=true`，并且持有 `mediaButtonReceiver`；
- `Media button session is null`（没有任何应用调用 `registerMediaButtonEventReceiver` 抢占媒体按钮会话）；
- 真实按键时日志出现 `Mediacenter next` / `playCtrlNext execute true`，即媒体中心自己执行了切歌。

也就是说：**音频焦点虽然在 CarMore 手里，但媒体按钮（MEDIA_BUTTON）会话被媒体中心占据**，按键事件到不了 CarMore。

常规修复路径在这台车机上都走不通：

| 思路 | 为什么不可行 |
| --- | --- |
| 改系统广播分发 / 抢 MEDIA_BUTTON 会话 | 无系统签名、无 root，改不了系统应用 |
| 让 CarMore 注册 `mediaButtonReceiver` | 改不了第三方 APK |
| 替换 / 禁用媒体中心 | 风险高，且可能影响仪表、语音等其他功能 |

前期研究找到了一条**可行路径**：媒体中心处理按键时会打印一条日志，而该日志以 `shell` 身份可读；再用 `input keyevent 87/88` 重新注入一次媒体键，CarMore 前台时就能正确切歌。但这条链路原先依赖**电脑通过 ADB 实时监听并转发**——插着电脑能用，日常开车显然不现实。

**本方案做的事，就是把这条链路固化到车机本地**：一个 APK 开机自启、后台保活、检测日志、自动重注入，运行期完全脱离电脑。

---

## 二、原理：它是怎么工作的

```mermaid
flowchart TD
    A[方向盘按下 上一曲/下一曲] --> B[原始广播被媒体中心拦截]
    B --> C[媒体中心打印日志<br/>MDC-[31000064]-MediaKeyReceiver<br/>action:0,keyCode:87/88]
    C --> D[BootReceiver 开机自启<br/>BOOT_COMPLETED / QUICKBOOT_POWERON]
    D --> E[BridgeService 前台服务<br/>+ PARTIAL_WAKE_LOCK 保活]
    E --> F[连接本机 adbd<br/>127.0.0.1:5555<br/>内置 ADB key 完成 AUTH]
    F --> G[app_process 以 shell uid=2000<br/>启动 ShellKeyBridge]
    G --> H[logcat -b all 监听按键日志<br/>正则只认 action:0 + 300ms 去抖]
    H --> I{CarMore 是否前台?<br/>dumpsys activity activities}
    I -- 是 --> J[input keyevent 87/88<br/>重注入媒体键]
    I -- 否 --> K[跳过, 不误切其他音源]
    J --> L[CarMore 切歌成功]
```

### 六个关键阶段

1. **开机自启**：`BootReceiver` 监听 `BOOT_COMPLETED`、`QUICKBOOT_POWERON`（含 HTC 变体）和 `MY_PACKAGE_REPLACED`，用 `startForegroundService()` 拉起服务。标准广播已足够可靠。

2. **前台保活**：`BridgeService` 是前台服务（常驻通知）+ `PARTIAL_WAKE_LOCK`，熄屏充电时也能工作。`onCreate()` 刻意保持轻量——2048 位 RSA 密钥的加载被放到 worker 线程，避免拖慢开机。

3. **回环 ADB 认证**：应用通过 `127.0.0.1:5555` 连接车机本机 adbd，实现一个极简 ADB 协议客户端：CNXN → AUTH（先签名、失败后公钥回退）→ CNXN。认证成功后用 `exec` 拉起 shell 侧进程。

4. **shell 侧 worker**：执行
   ```
   CLASSPATH=<apk路径> app_process /system/bin com.example.ecarxbridge.ShellKeyBridge
   ```
   进程以 `uid=2000`（shell）运行，因此拥有读 `logcat` 和 `input keyevent` 的能力——这正是普通第三方应用做不到的。

5. **日志检测**：核心命令为
   ```
   logcat -b all -v brief MDC-[31000064]-MediaKeyReceiver:D *:S
   ```
   - 用 `-b all` 覆盖全部日志缓冲区，防止真实按键日志落在非 main buffer 被漏读；
   - **不使用 `-T 0`**（该 ROM 报 `-T 0 invalid, setting to 1`，行为不确定）；
   - 正则锚定 `action:\s*0\s*,\s*keyCode:87(?!\d)`：`87` = 下一曲，`88` = 上一曲；只认 `action:0`（按下），忽略媒体中心同时打印的 `action:1`（抬起）——否则按一次会切两次；`300ms` 去抖兜底多 buffer 重复行。

6. **定向重注入**：先 `dumpsys activity activities` 判断 `mResumedActivity` / `topResumedActivity` 是否为 `com.unifiedconnect.app`（CarMore）。仅当前台是 CarMore 时才执行 `input keyevent 87/88`，避免误切其他音源。重注入的事件不会产生被监听的 MDC 日志，因此没有反馈环，实测 1:1 转发。

### 几个决定成败的实现细节

| 细节 | 说明 |
| --- | --- |
| **ADB 公钥载荷 524 字节** | AUTH 公钥回退必须发送标准的 **524 字节 mincrypt RSAPublicKey 结构**（`4+4+64*4+64*4+4`，到 exponent 结束）。**comment 不属于二进制载荷**——它在 `adb_keys` 里是 base64 串后面的空格明文。早期版本把 comment 编进去变成 547 字节，导致公钥回退匹配失败、AUTH 卡死。同时兼容 PKCS#8 私钥（`adb` 生成的 key 只有 modulus，指数按 65537 还原）。 |
| **三层自愈** | ① shell 侧 `SELFTEST`：每 60s 注入一条 `keyCode:9999` 探针，15s 内读不回就判定链路断裂、自杀重建；② `HB` 心跳 20s + `BridgeService` 看门狗：70s 无输出则强制关 socket，中断阻塞中的 `stream.read()`；③ 流 EOF / 异常自动重连并重新拉起 logcat。 |
| **ADB exec 竞争** | ADB exec 子进程的生命周期绑定在 TCP 连接上，断连即被 adbd 杀死。残留进程只在 loop 开始前用**独立连接**清理一次；若在同一连接上先 `exec kill` 再 `exec` 桥接，第二个 exec 会被 adbd 以 `CLSE (0x45534c43)` 拒绝。 |
| **不要 persistent / directBootAware** | 曾为刷 system 预置 `persistent="true"`、`directBootAware`、`READ_LOGS`、`LOCKED_BOOT_COMPLETED`。`persistent` 会让系统在开机极早期 fork 本进程并等待，叠加 onCreate 的初始化开销，**实测显著拖慢开机**。现已全部移除，Manifest 保持最小权限集。 |

---

## 三、实施局限性

这是一套**日志层的 workaround，不是对广播拦截的根治**。请在部署前明确了解以下边界：

| # | 局限 | 影响 | 应对 |
| --- | --- | --- | --- |
| 1 | **依赖媒体中心的日志格式** | tag `MDC-[31000064]-MediaKeyReceiver` 和 `action:0,keyCode:87/88` 文本一旦因 ROM / 媒体中心版本升级而改变，检测即失效 | 重新抓日志并调整 `ShellKeyBridge` 中的 tag 与正则 |
| 2 | **依赖 adbd TCP 回环 5555** | 若 OTA 关闭 `persist.adb.tcp.port` 或 adbd 不再监听，链路断开 | 需在有权限的前提下重新开启该属性 |
| 3 | **依赖已授权的 ADB key** | 「撤销 USB 调试授权」、恢复出厂、OTA 清 data 都会使 `adb_keys` 变化，内置私钥失效 | 用电脑重新 adb 连接并授权，再用当前已授权的 key 重新构建 APK |
| 4 | **依赖 SELinux Permissive 与 shell 组权限** | 若改为 Enforcing，或 shell 不在 `log` / `input` 组，`logcat` / `input` 会被拒 | 需调整 SELinux 策略或改用 system/priv-app 提权 |
| 5 | **焦点问题未根治** | 使用的是全局 `input keyevent`；媒体中心仍持有 active MediaSession 与 mediaButtonReceiver。CarMore 前台时实测可切歌，但其他场景仍可能被媒体中心抢走 | 根治需让 CarMore 注册 `mediaButtonReceiver`，或显式广播到 CarMore 的 `WheelMediaKeyReceiver`，不在本方案范围内 |
| 6 | **前台判定靠字符串匹配** | `dumpsys` 输出格式因 ROM 而异；分屏、画中画、浮窗等场景可能误判 | 误判时**跳过转发**（安全侧），不会误切 |
| 7 | **仅 CarMore 前台生效** | 其他播放器不转发（有意设计，避免误切音源） | 如需支持，扩展 `CARMORE` 包名判定 |
| 8 | **300ms 去抖窗口** | 极快连按（<300ms）会被当成重复日志吞掉；实测 800ms 连按正常切两次 | 按实际手感调整 `DEDUP_WINDOW_MS` |
| 9 | **保活不是绝对的** | 用户手动清除通知、任务管理器杀进程、部分 ROM 的激进回收都可能中断服务 | `START_STICKY` + 看门狗可自愈大部分情况，但不保证所有 ROM |
| 10 | **开机自启受 ROM 策略约束** | 部分车机有「自启动管理」白名单，会拦截 `BOOT_COMPLETED` | 需在系统设置中手动放行；真实断电重启复测尚未完成（模拟广播被系统拒绝，但早期真实开机日志已证明 `BootReceiver` 能被触发） |
| 11 | **刷 system/priv-app 有风险** | push 到系统分区操作失误可能导致无法开机 | 仅在确认可恢复的设备上进行；且**不要**为自启添加 `persistent` / `directBootAware`（会拖慢开机） |
| 12 | **无反馈环是经验结论** | 实测重注入事件不产生被监听的 MDC 日志，但这依赖媒体中心实现，非协议保证 | 若未来出现自激，可调大去抖或给注入事件加标记 |

---

## 四、前置条件

车机侧（本方案已在满足以下条件的设备上验证）：

- Android 9 / SDK 28，SELinux 为 **Permissive**；
- `persist.adb.tcp.port=5555`，adbd 监听 `:::5555`（回环地址 `127.0.0.1:5555` 可达）；
- `/data/misc/adb/adb_keys` 中已授权 APK 内置私钥对应的公钥；
- `shell`（`uid=2000`）属于 `log` / `input` 组，可读日志、可注入按键。

电脑侧（仅构建时需要）：

- `ANDROID_HOME`，Android Studio 自带 jbr，Android SDK build-tools 35.0.0，7-Zip；
- 上一级目录的 `carmore.keystore`（构建脚本用它签名）。

**无需 root**：`su` 在该车机只能切到 shell context，无法写 `adb_keys`，这也是采用「内置已授权 adb key」方案的原因。

---

## 五、构建与安装

```bash
# 构建（Windows / PowerShell）
powershell -File build.ps1
# 输出：ecarx-bridge.apk

# 安装到车机
adb install -r ecarx-bridge.apk
```

安装后打开应用（或重启车机）即可运行。应用界面是控制/诊断面板，显示当前桥接状态，带「启动/重连」「停止」按钮。

可选：刷入 `/system/priv-app/` 或 `/system/app/` 提升稳定性，但**不要**为了自启添加 `persistent` / `directBootAware`——标准 `BOOT_COMPLETED` 已足够，那两个属性反而拖慢开机。刷系统分区有变砖风险，操作前请确认车机可恢复。

---

## 六、运行与验证

```bash
# 1. 桥接进程是否以 shell (app_process) 运行
adb shell ps -A | grep app_process

# 2. 实时观察桥接日志（状态、心跳、自检、按键转发）
adb logcat | grep -E 'BridgeService|ShellBridge|BootReceiver'

# 3. 模拟一次下一曲日志，验证整条链路（CarMore 需在前台）
adb shell 'log -p d -t "MDC-[31000064]-MediaKeyReceiver" "action: 0,keyCode:87"'
# 期望：ShellBridge: KEY NEXT → REINJECTED NEXT（且恰好一次）

# 4. 验证自愈：手动杀掉 logcat 子进程或桥接进程，应在约 3 秒内自动重建
adb shell ps -A | grep logcat
```

诊断广播（无需真实按键即可触发测试）：

```bash
adb shell am broadcast -a com.example.ecarxbridge.SEND_NEXT
adb shell am broadcast -a com.example.ecarxbridge.SEND_PREV
```

---

## 七、目录结构

```
.
├── app/src/main/
│   ├── AndroidManifest.xml                 # 最小权限集 + BootReceiver
│   ├── java/com/example/ecarxbridge/
│   │   ├── ShellKeyBridge.java             # 核心：shell 侧日志监听 + 重注入
│   │   ├── BridgeService.java              # 前台服务、ADB 回环、看门狗、重连
│   │   ├── AdbConnection.java              # 极简 ADB 协议客户端
│   │   ├── AdbKeyUtil.java                 # 524 字节 mincrypt 公钥载荷
│   │   ├── BootReceiver.java               # 开机自启
│   │   ├── MainActivity.java               # 控制/诊断面板
│   │   └── MediaKeyProbeReceiver.java      # 诊断触发器
│   └── res/raw/adb_key                     # 内置 ADB 私钥（对应车机已授权公钥）
├── build.ps1                               # 构建脚本
├── ecarx-bridge.apk                        # 构建产物
├── README-使用说明.md                       # 详细运维 / 调试记录
└── PROMPT.md                               # 完整提示词（用于复现 / 交给 AI 重建）
```

---

## 八、风险与免责

- 本项目是面向特定车机固件的**个人研究性 workaround**，不是通用产品，也不保证在所有 ROM / 版本上可用。
- 修改系统分区、开启 ADB 回环、内置私钥等行为可能触及车辆保修条款；刷机有变砖风险，请自行评估。
- 内置私钥对应一把已在特定车机授权的 ADB key。**不要在你不拥有或未经授权的设备上使用本方案。**
- 请勿将此机制用于任何与安全驾驶或车辆控制相关的功能；本项目仅处理媒体切歌。
- 按「原样」提供，不附带任何明示或暗示的担保。使用即表示你理解并承担上述风险。

---

> 想从零复现这套方案？见 [PROMPT.md](PROMPT.md) —— 一份可直接粘贴给 AI 编码助手的完整提示词，包含全部环境事实、约束、踩坑记录与验收标准。
