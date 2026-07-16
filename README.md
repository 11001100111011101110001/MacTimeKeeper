# Mac mini 低开销时间校准

这是一套准备复制到目标 Mac mini 上运行的方案；它没有在当前机器安装，也没有修改当前机器的系统时间。

它只依赖 macOS 自带的 `timed`、`sntp` 和 `launchd`：系统 `timed` 负责持续同步，额外的 LaunchDaemon 每 60 秒做一次轻量校准。默认采用交易安全策略：小于 50 ms 的偏差通过 `adjtime` 平滑修正；达到或超过 50 ms 时只记录告警，不直接跳动系统时钟。

> 无论脚本运行多频繁，公网 NTP 都无法保证“绝对零偏差”。这套默认配置的目标是在不让 wall clock 突然前跳/后跳的前提下，把正常偏差控制在公网和网络路径允许的毫秒级范围内。

## 安装

把这四个文件复制到目标 Mac mini，在目标机器的本目录运行：

```sh
chmod +x mac-time-keeper
sudo ./mac-time-keeper install
```

安装器会先测试 NTP 连通性，失败时不会加载定时任务；随后启用 macOS 自动网络时间、设置同一个 NTP 源，并执行一次首次对齐。

**请先停掉交易程序再安装。** 首次对齐使用 `sntp -sS`：偏差小于 50 ms 时平滑修正，偏差较大时会直接设置系统时钟。安装完成并确认偏差后再启动交易程序。要求 macOS 11 或更高版本，并确保防火墙允许出站 UDP 123。

## 检查

```sh
sudo /usr/local/libexec/mac-time-keeper status
```

输出末尾类似：

```text
+0.002314 +/- 0.006100 time.apple.com 17.x.x.x
```

第一项是时钟偏差（秒）。正数表示本机慢，负数表示本机快；`+/-` 后是本次网络测量的不确定度。查看偏差只做 NTP 查询，不会修改时间。

其他命令：

```sh
# 按周期安全策略立即校时（默认不会跳时）
sudo /usr/local/libexec/mac-time-keeper sync

# 只检测；绝对偏差达到 WARN_OFFSET_MS 时返回退出码 2
/usr/local/libexec/mac-time-keeper health

# 停掉交易程序后强制对齐（偏差大时可能跳时）
sudo /usr/local/libexec/mac-time-keeper bootstrap

# 查看最近一小时的失败/偏差告警日志
/usr/local/libexec/mac-time-keeper logs

# 卸载；自定义配置会保留
sudo /usr/local/libexec/mac-time-keeper uninstall
```

配置文件位于 `/etc/mac-time-keeper.conf`。如要改用交易所、券商或机房提供的 NTP 服务器，修改 `TIME_SERVER` 后运行：

```sh
sudo /usr/local/libexec/mac-time-keeper reload
```

`reload` 会先验证新服务器可达，再把 macOS `timed` 和本工具切换到同一个 NTP 源。

默认的 `ALLOW_TIME_STEP=0` 不允许周期任务直接设置时钟。若改成 `1`，偏差超过 50 ms 时也会自动强制对齐；这能缩短故障后的错误时间窗口，但交易程序必须能够承受 wall clock 前跳或后跳，通常不建议。

## 性能和精度说明

- 每 60 秒启动一个短生命周期进程，最多查询 3 个 DNS 记录对应的 NTP 端点；任务使用 `Background` 调度和低优先级磁盘 I/O。正常且偏差低于阈值时不写日志。
- 优先使用目标 Mac mini 同机房、同局域网或交易接入商提供的可信 NTP 源，并使用有线网络。公网 NTP 的精度受路由、拥塞和链路不对称影响；要求亚毫秒时应采用机房级 NTP/PTP 及相应硬件，而不是提高脚本执行频率。
- 交易程序计算超时、耗时和重试间隔时应使用单调时钟；系统 wall clock 只用于时间戳。这样即使发生首次大幅校时，也不会把持续时间算错。
- `launchd` 在 Mac 睡眠期间不会执行 60 秒任务。作为服务器使用时，应在“系统设置 → 节能”中避免自动睡眠。

## 卸载行为

```sh
sudo /usr/local/libexec/mac-time-keeper uninstall
```

卸载会删除本工具的 LaunchDaemon 和执行文件，但保留 `/etc/mac-time-keeper.conf`，也保留 macOS 自带的自动网络时间及已选 NTP 服务器。这避免服务器在卸载后失去系统级时间同步。

## 参考

- [Apple：在 Mac 上自动设置日期与时间](https://support.apple.com/guide/mac-help/set-the-date-and-time-automatically-mchlp2996/mac)
- [macOS `sntp(1)` 手册](https://keith.github.io/xcode-man-pages/sntp.1)
- [macOS `timed(8)` 手册](https://keith.github.io/xcode-man-pages/timed.8.html)
- [macOS `launchd.plist(5)` 手册](https://keith.github.io/xcode-man-pages/launchd.plist.5.html)
