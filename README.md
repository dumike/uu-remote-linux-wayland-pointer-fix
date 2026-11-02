# UU 远程（uu-remote-for-linux）GNOME 50 Wayland 分数缩放鼠标失灵修复

针对 uu-remote-for-linux（UU 远程 Linux 兼容包装器，1.1.43）在 **GNOME 50 (Wayland-only) + 4K 显示器 150% 分数缩放** 环境下，鼠标在屏幕**下方 ⅓ 与右侧 ⅓ 区域完全无法点击**问题的根因分析与修复补丁。

> 一句话结论：**xdg-desktop-portal 按逻辑分辨率（2560×1440）校验绝对坐标，Mutter 却把坐标当物理像素（除以 1.5 缩放比）解释——150% 缩放下两个约束无交集，绝对坐标注入天然不可用。改用"桥内自跟踪位置 + 相对位移"注入后一切正常。**

---

## 1. 出现了什么问题？

### 环境

| 组件 | 版本 |
|---|---|
| 系统 | Linux（Wayland 桌面会话） |
| GNOME Shell / GDM / Mutter | 50.1（GNOME 49 起 X11 会话已被移除，Wayland-only） |
| xdg-desktop-portal | 1.21.1 |
| 显示器 | 3840×2160 (4K)，缩放 **150%** → 逻辑分辨率 **2560×1440** |
| uu-remote-for-linux | 1.1.43（非官方 UU 远程 Linux 包装器，Wine + PipeWire） |

### 现象

- 远程连接后，**光标可以移动到屏幕任何位置**（远端自绘光标），但点击**左下角（Dock/回收站）以及大量位于屏幕下部、右侧的按钮完全无反应**——左键右键都没用。
- 受影响区域恰好是屏幕的**下方 ⅓ 与右侧 ⅓**（即 y≥1440 或 x≥2560 的物理坐标区域）。
- 输入桥日志（`~/.local/state/uu-remote-for-linux/input-bridge.log`）里积累了 **14205 条**同样的错误：

```
输入事件注入失败：g-dbus-error-quark: GDBus.Error:org.freedesktop.DBus.Error.Failed: Invalid position (0)
```

## 2. 根因分析

UU 远程包装器的输入桥（`/usr/lib/uu-remote-for-linux/uu-remote-input-bridge`，Python）通过 **xdg-desktop-portal 的 RemoteDesktop 接口**注入鼠标事件。问题出在三层软件对"绝对坐标属于哪个坐标系"的理解互相矛盾：

### 证据链

**① xdg-desktop-portal 1.21.1 —— 按逻辑尺寸做边界校验**

`src/remote-desktop.c` 的 `check_position()`（[源码](https://github.com/flatpak/xdg-desktop-portal/blob/master/src/remote-desktop.c)）：

```c
screen_cast_stream_get_size (stream, &width, &height);   // 逻辑尺寸 2560×1440
if (x >= 0.0 && x < width && y >= 0.0 && y < height)
    return TRUE;
...
g_dbus_method_invocation_return_error (..., "Invalid position");
```

坐标必须 **小于流的逻辑尺寸（2560×1440）**，否则直接拒绝——这就是日志里 `Invalid position` 的来源（Mutter 50.1 二进制中已无此字符串，实测确认错误由 portal 抛出）。

**② Mutter 50.1 —— 把坐标当物理像素解释（除以缩放比）**

实测：发送逻辑坐标 (x=1400) 时光标实际落在 y≈933 = 1400 ÷ 1.5 处（差三个 Dock 图标的距离）。即 Mutter 内部把收到的值**除以显示器缩放比**当作物理像素处理——它期望收到的是**物理像素（最大 3839×2159）**。

**③ 结论：两个约束无交集**

| 发送坐标 | portal 校验（须 <2560×1440） | Mutter 落点（除以 1.5） |
|---|---|---|
| 物理像素 | ❌ 外圈区域被拒（原始 bug：下方/右侧死区） | ✅ 准确 |
| 逻辑像素 | ✅ 全部通过 | ❌ 全部偏移到 ⅔ 处 |

**在 150% 分数缩放下，不存在能同时满足两层的绝对坐标表示法。** 这是 xdg-desktop-portal 与 Mutter 之间的上游组合缺陷（参见 [xdg-desktop-portal #2077](https://github.com/flatpak/xdg-desktop-portal/issues/2077)、[gnome-bridge CHANGELOG 中对同一问题的记录](https://github.com/patrickjaja/gnome-bridge/blob/main/CHANGELOG.md)：*"the outer band of a scaled monitor was unreachable ('Invalid position' ... bounds-checks against the logical stream size)"*）。

### 排除掉的方案

- **切回 X11 会话**：GNOME 50 已彻底移除 X11 会话（`/usr/share/xsessions/` 无 GNOME 项），GDM 的 `WaylandEnable=false` 无处生效。ToDesk Linux 版（仅支持 X11）即因此在本环境完全不可用。
- **包装器的 XTest/X11 注入后端**（`UU_REMOTE_DESKTOP_BACKEND=x11`）：实测 XWayland 的 XTest 事件无法到达 GNOME Shell（用 Overview 状态探针验证），弃用。
- **用 XQueryPointer 做位置反馈、把绝对目标换算成相对位移**：X11 的指针位置反馈**不跟随门户虚拟指针**——当光标悬停在 Wayland 原生界面（Shell/Dock）上时 XWayland 收不到移动事件，反馈冻结 → 位移差值持续发散 → 光标被推到左上角 (0,0) 钉死。

## 3. 怎样修改的？

最终方案：**在输入桥内部自己记账跟踪光标逻辑位置，把所有绝对定位目标改用 `NotifyPointerMotion`（相对位移）发送**。相对位移：

- 在 portal 侧**没有任何位置校验**（`handle_notify_pointer_motion` 不调用 `check_position`）；
- 在 Mutter 侧**与缩放无关**（`notify_relative_motion` 直接作用于逻辑坐标系）；
- 位移量 = 目标逻辑位置 − 桥内记录的当前逻辑位置，逐事件精确、零漂移。

首次定位需要一个"播种"：发送一次绝对坐标 (200, 200)（必过 portal 校验），Mutter 会把它落在 (200÷scale, 200÷scale)——桥内按同样的公式记录落点，之后全部走相对位移。原生相对事件同样更新记账，保证两条输入路径一致。

补丁的完整迭代过程（三次尝试）：

| 版本 | 思路 | 结果 |
|---|---|---|
| v1 | 把 `stream_buffer_point()` 改为恒等映射（直发逻辑像素） | ❌ 全屏点击偏移到 ⅔ 处（点回收站落在上方三个图标处）——证明 Mutter 除以缩放比 |
| v2 | 绝对目标换算相对位移，用 XQueryPointer 做位置反馈 | ❌ 光标钉死左上角"活动"按钮——证明 X11 反馈不跟随门户指针 |
| v3（最终） | 桥内自跟踪逻辑位置 + 一次性播种 + 纯相对位移 | ✅ **全屏任意位置精准点击，无死区、无偏移** |

### 修改点（对照 `patches/uu-remote-input-bridge-wayland-pointer.patch`）

1. `stream_buffer_point()`：恒等返回（作为无反馈时的后备路径，逻辑像素直发）。
2. `motion()` 绝对分支：播种 + 相对位移发送 + 记账更新。
3. `motion()` 相对分支：原生相对事件同步更新记账。

## 4. 最后的效果

- ✅ 屏幕四角、Dock、任务栏、任意窗口按钮**全部可以精准点击**（左下角回收站、右下角按钮实测正常）；
- ✅ 光标定位无偏移、无漂移（每次移动逐事件校正记账）；
- ✅ `input-bridge.log` 不再出现 `Invalid position`；
- ✅ 键盘、滚轮、拖拽不受影响（本来就不走绝对坐标路径）。

## 如何应用

```bash
# 1. 备份原文件
sudo cp -a /usr/lib/uu-remote-for-linux/uu-remote-input-bridge \
           /usr/lib/uu-remote-for-linux/uu-remote-input-bridge.bak

# 2. 打补丁（在仓库根目录）
sudo patch /usr/lib/uu-remote-for-linux/uu-remote-input-bridge \
           -i patches/uu-remote-input-bridge-wayland-pointer.patch

# 3. 语法自检
python3 -m py_compile /usr/lib/uu-remote-for-linux/uu-remote-input-bridge

# 4. 重启 UU 远程
uu-remote-for-linux --stop && sleep 3 && uu-remote-for-linux &
```

## 如何回滚

```bash
sudo cp -a /usr/lib/uu-remote-for-linux/uu-remote-input-bridge.bak \
           /usr/lib/uu-remote-for-linux/uu-remote-input-bridge
uu-remote-for-linux --stop && sleep 3 && uu-remote-for-linux &
```

## 已知限制

- 包装器**升级会覆盖补丁文件**（升级后需重新应用；上游若修复此问题则无需）。
- 若远程会话断开重连后出现整体偏移，重启一次 UU 远程即可重新播种。
- 仅在单显示器场景验证；多显示器下跨屏移动的记账边界未测试。

## 参考

- [xdg-desktop-portal #2077 — NotifyPointerMotionAbsolute fails with "Invalid position"](https://github.com/flatpak/xdg-desktop-portal/issues/2077)
- [gnome-bridge CHANGELOG — 同一问题的记录与修复思路](https://github.com/patrickjaja/gnome-bridge/blob/main/CHANGELOG.md)
- [Mutter meta-remote-desktop-session.c 源码](https://gitlab.gnome.org/GNOME/mutter/-/blob/main/src/backends/meta-remote-desktop-session.c)
- [xdg-desktop-portal RemoteDesktop 源码](https://github.com/flatpak/xdg-desktop-portal/blob/master/src/remote-desktop.c)

## License

MIT
