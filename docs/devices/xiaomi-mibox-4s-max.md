# 小米盒子 4S Max（MiTV-AZFP0）

- 型号 `MiTV-AZFP0`，Android 9（SDK 28），32 位 ARM（armeabi-v7a）。
- 原桌面：`com.mitv.tvhome`（PatchWall）。
- 安装的 flavor：`standard`。

下面四项是这台上要特别处理的地方。每一项都写明了查到了什么、怎么处理的、哪些验证过。

## 强停后守护「已启用但从未绑定」

- 现象：`am force-stop app.orchard.tv` 恰好发生在通知监听（`RevivalListener`）把守护写回 `enabled_accessibility_services` 之后，守护在设置里是启用的，Aerial 在运行，但系统一直不绑定它。`dumpsys activity services` 里 `HomeGuardService` 的连接是 DEAD。往列表里再写什么都没用，只有重启电视才恢复。这台上复现过一次。
- 原因：读 AOSP Pie 的 `AccessibilityManagerService` 源码得到的结论：`bindLocked` 把服务放进 `UserState.mBindingServices`，只有 `onServiceConnected`（或切换用户）才会把它拿出来；这里没有 `onBindingDied`，而 `updateServicesLocked` 对这个集合里的服务既不绑定也不解绑。Android 10 同样没有 `onBindingDied`。所以这是 Android 9 和 10 的行为，Android 11 起 `rebindIfStalled` 沿用原来的做法。这一条是读源码的结论，没有在其他机型上复现。
- 处理（`GuardSetup`）：
  - Android 9/10 上，每次「拿出再放回」之后过 25 秒看一次有没有绑上；连续 2 次没绑上，就在本进程里不再拿出再放回（那样没用，还会让系统反复绑定、解绑列表里的其他无障碍服务，例如小爱）。
  - 同一批检查（`MainActivity`、通知监听、同步适配器）几秒内会连着来，Android 9/10 上两次「拿出」之间至少隔 27.5 秒，让上一次有时间绑上。
  - 放回前再读一次列表，已经有守护就不写。
  - 日志只打一次，`Diagnostics` 的 Home Guard 行带上 `binding_wedged=true`，提示重启一次电视。
- 注意：`tools/install.sh` 和 `site/agent.md` 的安装步骤在装完后会执行 `am force-stop`，这在 Android 9 上正好可能触发这个状态。这里没有改这两处，只作记录。这台上不带 `force-stop` 安装后，守护正常绑定，重启后也正常。
- `MainActivity` 在 `onStart`、`onResume`、`onNewIntent` 各排一次检查（同一时间最多一个待执行），因为从别的 App 回来、按主页键回来都不是 `onStart`。

## 遥控器接收器被当成手柄

- 现象：诊断行里 `gamepad=true`。守护发现有手柄时，在别的 App 里不再过滤 Home 键（#774 的设计，防止手柄按键被吞）。这台上过滤一关，Home 先到小米桌面，Aerial 要等大约 5 秒才回来（Android 的后台启动限制）。
- 原因：电视自带的遥控器接收器（`aml_keypad`，vendor/product 都是 0x0001）报了 `SOURCE_GAMEPAD`；软件的 `eventserver-Joystick`（0xdead:0xbeef）也被报成手柄。
- 处理（`guard/GamepadDevices.kt`）：名字、vendor、product 三项都完全相同才排除；名字相同但 ID 不同，或名字只是以它们开头（如 `eventserver-Joystick Pro`）的设备仍然算手柄。真手柄照旧算。
- 实测：修后诊断行是 `gamepad=false`（修前是 `gamepad=true`，来自遥控器接收器）。在别的 App 里按 Home，日志里从守护的「Home key taken before the system saw it」到 Aerial 的 `onResume` 不到 0.1 秒；修前约 5 秒。

## 开机和唤醒时小米桌面抢到前台

- 现象：PatchWall 的 `com.mitv.tvhome.BootActivity` 不是 HOME 活动。Android 9 上它到前台时，守护只认 HOME 活动，忽略了它的窗口事件，要等开机后的复查才处理。
- 处理（`HomeGuardService`）：
  - 开机后 3 分钟内、亮屏后 15 秒内，把 `BootActivity` 当作原桌面处理，并只在这两个时间窗里每秒查一次用量统计（`foregroundCheck`）。平时不轮询；Android 6 及以下、聚好看、极光桌面原有的持续轮询不变。
  - 按主页键后 5.5 秒内也认它。
  - 「桌面」磁贴（`openStockLauncher`）：直接启动失败时，找该包里没有签名权限的 MAIN 入口（别名），失败后才用无障碍的 HOME 动作；都不成功就恢复原来的放行时间，不留下一小时的放行。
  - 原桌面名单里同时记入别名指向的真实活动（`targetActivity`），因为无障碍上报的是真实活动，不一定是别名。
- 验证情况：根据这台一次开机的日志实现。提交者没有在最后一次编译后重测，第二次开机、唤醒时间窗没有再看过。

## 长按主页键打开小米任务中心

- 小米的任务中心是 `com.xiaomi.mitv.systemui` 里的系统浮层，没有可以直接启动的 `RecentsActivity`。
- 处理：`HomeGuardService.openRecents()` 先检查 `com.xiaomi.mitv.systemui/.MiTVSystemUIService` 是否存在、启用、已导出、不需要我们没有的权限，满足就 `startService` 并带上 `recent_control=2`，然后才走原来的几个通用动作。没有这个服务的电视走原来的路径。
- 提交者在这台上亲眼确认：重启后长按主页键打开的是小米的任务中心。

## 还没验证的

- 其他小米型号和 Android 11 以上：卡住绑定的行为按 AOSP 只在 Android 9/10，SDK ≥ 30 沿用原来的做法。
- 新的检测代码（`binding_wedged=true`）有单元测试，但这台上没有再让它真正触发过。
- 开机和唤醒时间窗在第二次开机上的表现；小米固件升级之后的表现。
