# 完整提示词（复现 / 交给 AI 重建用）

> **用途**：把下面整段复制给一个支持读写文件与终端的 AI 编码助手（如 WorkBuddy / Claude Code / Cursor 等），即可在本机从零重建本方案。它已经把所有环境事实、约束、踩坑记录和验收标准写全，不需要依赖本仓库以外的上下文。
>
> **建议**：如果手上有本仓库源码，把源码一起提供给 AI，效果更好；如果没有，这段提示词本身足以驱动完整实现。
>
> **语言**：提示词正文为中文，符合本项目的沟通语言。

---



```text
# 角色与目标

你是一名资深 Android 系统工程师。请在本机（Windows，已安装 Android Studio 与 Android SDK）为一个 ECARX 车机开发并验证一个「方向盘切歌桥接」APK。请严格按下面给出的背景、环境事实、约束和验收标准执行，不要自行更换技术路线；遇到与事实冲突的地方，以本提示词给出的事实为准。

目标：车机开机后自动在后台运行一个 APK，监听方向盘「上一曲/下一曲」按键引发的系统日志，检测到后以 shell 身份重新注入媒体键事件，使 CarMore / CarLife 能正确切歌。运行时不得依赖电脑，不得依赖 root。

# 一、问题背景（已诊断确认，不要重新论证）

- 车型/系统：ECARX 车机，Android 9 / SDK 28，SELinux 为 Permissive。
- 现象：方向盘「上一曲/下一曲」的原始广播被系统媒体中心 `ecarx.xsf.mediacenter` 拦截并消费，CarMore / CarLife 收不到。用 CarMore 听歌时方向盘切歌无效。
- 证据：`dumpsys media_session` 显示 mediacenter 的 MediaSession `active=true` 且持有 `mediaButtonReceiver`，`Media button session is null`；音频焦点 GAIN 在 CarMore（`com.unifiedconnect.app`），但 MEDIA_BUTTON 会话被 mediacenter 占据。真实按键时日志出现 `Mediacenter next` / `playCtrlNext execute true`，即媒体中心自己切了歌。
- 已验证可行的路径：媒体中心处理按键时会打印日志 `MDC-[31000064]-MediaKeyReceiver: action: 0,keyCode:87`（下一曲按下）/ `action: 1,keyCode:87`（抬起）；上一曲为 `88`。该日志以 shell 身份可读，用 `input keyevent 87/88` 重注入能让 CarMore 切歌。前期已用电脑 adb 手工验证通过，现在要把它固化到车机本地。

# 二、设备环境事实（已实测，直接采用）

- `persist.adb.tcp.port=5555`，adbd 监听 `:::5555`，回环地址 `127.0.0.1:5555` 可达。
- `/data/misc/adb/adb_keys` 中已授权某把 ADB 私钥的公钥（注释形如 `io@JOY2GO`），文件为 `system:shell 0640`，无 root 不可写。
- `su` 在该车机只能切到 shell context，无法写 system 分区或 adb_keys —— 所以采用「把电脑侧已授权的 adb 私钥内置进 APK」的方案。
- `shell`（uid=2000）属于 `log` / `input` 组，可读 logcat、可 `input keyevent`。
- 车机常态负载很高（load average 27+），开机阶段更甚 —— 主线程初始化必须尽量轻。

# 三、硬性约束

1. 运行时零电脑依赖、零 root 依赖：靠内置已授权 adb key 连接本机 adbd。
2. 开机后台自动启动，进程常驻，断线/卡死能自愈。
3. 不能拖慢开机（车机本身开机就慢）。
4. 只在前台是 CarMore 时转发，避免误切其他音源；一对 `action:0` + `action:1` 只能切一次歌；真实连按（>300ms）要能正确切多次。
5. 保持 Manifest 最小权限集；不要为自启添加 `persistent` / `directBootAware` / `READ_LOGS` / `LOCKED_BOOT_COMPLETED`。

# 四、必须实现的组件

包名 `com.example.ecarxbridge`，至少包含：

1. `BootReceiver`：监听 `BOOT_COMPLETED`、`android.intent.action.QUICKBOOT_POWERON`、`com.htc.intent.action.QUICKBOOT_POWERON`、`MY_PACKAGE_REPLACED`，调用 `startForegroundService()` 拉起服务。导出并声明 `RECEIVE_BOOT_COMPLETED` 权限。
2. `BridgeService`：前台服务（常驻通知，Android O+ 需 NotificationChannel），持 `PARTIAL_WAKE_LOCK`；`onStartCommand` 返回 `START_STICKY`。onCreate 保持轻量：**RSA 密钥加载放到 worker 线程**，不要放主线程。worker 流程：
   - 从 `res/raw/adb_key` 读取内置私钥，派生 524 字节公钥载荷；
   - 用独立连接先做一次残留桥接进程清理（`for p in $(pidof app_process); do grep -q ShellKeyBridge /proc/$p/cmdline && kill -9 $p; done`）；
   - 连接 `127.0.0.1:5555`，完成 ADB 握手（CNXN → AUTH 签名 → AUTH 公钥回退 → CNXN）；
   - `exec` 命令：`CLASSPATH=<getPackageCodePath()> app_process /system/bin com.example.ecarxbridge.ShellKeyBridge`；
   - 循环读取输出转发到 logcat（tag `ShellBridge`），维护 `lastBridgeOutputAt`；
   - 看门狗线程：每 10s 检查，若 70s 无任何输出则 `closeConn()` 强制中断阻塞中的 `stream.read()`，走重连（重连前睡眠 3s）。
3. `AdbConnection`：极简 ADB 协议客户端。握手阶段给 socket 设置超时（例如 5s），CNXN 成功后恢复阻塞读（setSoTimeout(0)），避免 AUTH 卡死。提供 `exec(String cmd)` 返回可读的 `AdbStream`。
4. `AdbKeyUtil`：
   - 从私钥还原公钥，兼容 `RSAPrivateCrtKey` 与 PKCS#8 `RSAPrivateKey`（后者只有 modulus，公开指数按 65537 还原）；
   - 生成 ADB AUTH_RSAPUBLICKEY 载荷，必须是 **524 字节**的 mincrypt RSAPublicKey 小端结构：`words(4) + n0inv(4) + modulus(64*4) + rr(64*4) + exponent(4)`。**comment 绝不能编进二进制载荷**（它在 adb_keys 里是 base64 串后的空格明文）。编成 547 字节会导致公钥回退匹配失败、AUTH 卡死。
5. `ShellKeyBridge`：由 app_process 以 shell 身份运行的 `main()`：
   - 打印 `ShellKeyBridge START v2 uid=<myUid()>`；
   - 启动 logcat 子进程：`logcat -b all -v brief MDC-[31000064]-MediaKeyReceiver:D *:S`。**不要用 `-T 0`**（本 ROM 报 `-T 0 invalid, setting to 1`，行为不确定）；`-b all` 防止真实日志落在非 main buffer 被漏读。
   - 正则只认按下：`action:\s*0\s*,\s*keyCode:87(?!\d)` / `...keyCode:88(?!\d)`。必须锚定 `action:0`——媒体中心每次按键打印两条（按下 action:0、抬起 action:1），不区分就会按一次切两次。`(?!\d)` 防止误匹配 870 等。
   - 300ms 去抖：相同 keyCode 在窗口内的重复日志只转发第一次，且只在真正转发成功后更新时间戳（SKIP 时不更新）。
   - 前台判定：`dumpsys activity activities` 输出中 `mResumedActivity` 或 `topResumedActivity` 行含 `com.unifiedconnect.app` 才转发，否则打印 SKIP。
   - 转发：`input keyevent 87`（下一曲）/ `88`（上一曲），打印 `REINJECTED NEXT/PREVIOUS`。重注入不产生被监听的 MDC 日志，无反馈环。
   - 心跳 HB：每 20s 打印一次（含 ack/pending/logcatAlive）。
   - 自检 SELFTEST：每 3 次心跳（60s）用 `log -p d -t MDC-[31000064]-MediaKeyReceiver "action: 0,keyCode:9999 selftest:N"` 注入探针并确认能被本进程 logcat 读回；15s 内读不回则 `logcat.destroy(); System.exit(2)` 自杀，让 BridgeService 重建整条链路。探针用 9999 不会被 87/88 正则误判。
   - logcat 流 EOF（reader.readLine() 返回 null）时退出，触发重建。
6. `MainActivity`：控制/诊断面板，显示 `BridgeService.STATUS`，提供启动/重连、停止按钮。
7. `MediaKeyProbeReceiver`：诊断触发器，接收显式广播 `com.example.ecarxbridge.SEND_NEXT` / `SEND_PREV` 用于测试。
8. `AndroidManifest.xml`：最小权限 `RECEIVE_BOOT_COMPLETED`、`FOREGROUND_SERVICE`、`INTERNET`（必须有，否则无法开回环 socket）、`WAKE_LOCK`。**不要**加 persistent / directBootAware / READ_LOGS / LOCKED_BOOT_COMPLETED。

# 五、必须规避的坑（血泪教训）

1. 不要在同一 ADB 连接上先 `exec kill` 清理再 `exec` 桥接 —— 第二个 exec 会被 adbd 以 `CLSE (0x45534c43)` 拒绝。清理只在 loop 开始前用独立连接做一次；exec 子进程生命周期绑定 TCP 连接，断连即被 adbd 杀死，重连时无需再 kill。
2. ADB 公钥载荷不能带 comment，必须 524 字节（2048 位 RSA）。
3. 不要用 `logcat -T 0`。
4. 不要只匹配 keyCode 不匹配 action，否则按一次切两次。
5. private key 加载不要放 onCreate 主线程。
6. 不要为自启添加 persistent / directBootAware 等属性 —— 会让系统在开机早期 fork 并等待本进程，实测明显拖慢开机。
7. 握手阶段必须设超时，否则 ADB AUTH 可能永久阻塞。

# 六、构建（build.ps1，Windows PowerShell）

- 用 Android Studio jbr 的 javac，Android SDK 35 平台 android.jar 与 build-tools 35.0.0。
- 步骤：① 编译 sdk stubs → ② 打 stub jar → ③ aapt2 compile + link（**必须加 `--java build/gen` 生成 R.java**，否则后续编译失败）→ ④ javac 编译 app 源码 + 生成的 R.java → ⑤ d8 转 dex 并塞进 base.apk → ⑥ aapt2 dump badging 校验 → ⑦ zipalign + apksigner 签名（用上一级目录的 `carmore.keystore`，alias `carmore`，密码 `android`）→ 输出 `ecarx-bridge.apk`。
- 用 `adb install -r ecarx-bridge.apk` 安装。

# 七、验收标准（逐条在真机上验证并输出结论）

1. ADB 认证成功：日志出现 ADB connected / CNXN 成功，不再卡在 AUTH。若曾报 `private key is not RSAPrivateCrtKey` 或公钥载荷 547 字节，说明第 4 条未修好。
2. `ps -A | grep app_process` 能看到 ShellKeyBridge 进程，日志 `ShellKeyBridge START v2 uid=2000`。
3. 心跳每 20s 输出，自检 `SELFTEST sent seq=N` 与 `acked=N` 对应。
4. 注入合成日志 `log -p d -t "MDC-[31000064]-MediaKeyReceiver" "action: 0,keyCode:87"`，CarMore 前台时应看到 `KEY NEXT` → `REINJECTED NEXT`，且恰好 1 次；`keyCode:88` 同理。
5. 注入一对 `action:0` + `action:1` 日志，`REINJECTED` 恰好 1 次（验证不会切两次）。
6. 间隔 800ms 连按两次，`REINJECTED` 应为 2 次（验证去抖不过度抑制）。
7. 杀掉 logcat 子进程、杀掉桥接 app_process，均应在约 3 秒内自动重建。
8. dumpsys 中应用无 PERSISTENT 标志，确认没有拖慢开机的属性。
9. 重启车机后 BootReceiver 能触发自启（若模拟广播被系统拒绝，说明需真实断电重启验证，据实说明）。

# 八、已知局限（请在交付说明中原样列出，不要隐瞒）

- 本质是日志层 workaround，依赖媒体中心的日志 tag 与文本格式；ROM 升级可能失效。
- 依赖 adbd TCP 5555、已授权 adb key、SELinux Permissive、shell 在 log/input 组。
- 「撤销 USB 调试授权」/恢复出厂/OTA 清 data 会使内置 key 失效，需重新授权并重建 APK。
- 焦点问题未根治：使用全局 input keyevent，媒体中心仍持有 active MediaSession 与 mediaButtonReceiver，某些场景仍可能抢焦点。根治需让 CarMore 注册 mediaButtonReceiver 或显式广播到 CarMore 的 WheelMediaKeyReceiver。
- 前台判定靠 dumpsys 字符串匹配，分屏/画中画可能误判（误判时跳过，不误切）。
- 仅 CarMore 前台生效；300ms 内极快连按会被去抖吞掉。
- 保活非绝对（用户可清除通知/系统可杀进程）；刷 system/priv-app 有变砖风险，且不要为此加 persistent。
- 真实断电重启复测尚未完成，据实说明。

# 九、交付物

- 可安装的 `ecarx-bridge.apk` 与完整源码。
- 一份面向 GitHub 的 README，包含「必要性 / 工作原理 / 实施局限性 / 前置条件 / 构建安装 / 验证 / 风险免责」。
- 逐条列出第八节局限，并附上真机验收结论。
```

---

## 使用建议

- **复现方案**：整段粘贴给 AI 编码助手，并附上本仓库源码，按 [README.md](README.md) 的验收方法在真机验证。
- **改造移植**：如果目标车机的日志 tag、媒体中心包名或 CarMore 包名不同，重点修改提示词中「背景」一节的日志特征，以及 `ShellKeyBridge.java` 的 `MEDIA_TAG` / `CARMORE` 常量和正则。
- **分享到 GitHub**：`README.md` 作为仓库首页，`PROMPT.md` 作为「如何复现」的附录，两者互相链接。
