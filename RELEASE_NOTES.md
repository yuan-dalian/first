# USB Speaker 1.0.0

把你的平板变成电脑的扬声器，**只走那根 USB 数据线**。不用 WiFi，不用蓝牙，不用网络共享。

## 下载什么

| 文件 | 用途 |
| --- | --- |
| `usbspeaker-1.0.0.apk` | 装到平板上的接收端（arm64-v8a，Android 8.0+） |
| `usb-speaker-1.0.0-source.zip` | PC 端脚本 + 全部源码，解开就能跑 |

## 快速开始

1. 平板打开「开发者选项 → USB 调试」，用数据线连电脑，在平板上允许调试。
2. 解压源码包，装依赖：

   ```
   python -m pip install -r requirements.txt
   ```

3. 装 APK 到平板：

   ```
   adb install usbspeaker-1.0.0.apk
   ```

4. 启动：

   ```
   start.cmd
   ```

   平板会自动亮屏→启动接收端→连上后自动息屏，然后开始出声。

想开机自启就提权跑一次 `pc\set_autostart.ps1`。

## 它做了什么

```
PC 正在播放的音频
  → WASAPI loopback 采集（PyAudioWPatch）
  → 16-bit PCM → TCP 127.0.0.1:27182
  → adb forward（走 USB 线）
  → com.usbspeaker（前台 Service）
  → AAudio / MMAP（独占共享内存通路）
  → 平板喇叭
```

镜像的是 **Windows 当前正在用的默认输出**，物理声卡还是虚拟声卡都跟着走。想固定某一路用 `--device`。

## 延迟

实测端到端 **约 51 ms**，出声时零丢帧：

```
playing 48000 Hz / 2 ch  native mmap  lat 25 ms  ring 26 ms  kq 0 ms
  low 16 ms  high 36 ms  gap 0 ms  calls 51  lost 0.0 ms/s  drop 0.0 ms/s  trim 0.0 ms/s
```

怎么读这几个数：

- `lat` — AAudio 报告的输出延迟
- `ring` — 平板端环形缓冲的水位
- `kq` — 平板内核 socket 接收队列的积压（正常 0–30 ms）
- `lost` / `drop` / `trim` — 三个「丢音频」的计数器，**出声时都应该正好是 0.0**

> PC 静音时 `lost` 会自己涨起来，这是正常的：loopback 在没有声音可采时不产帧，播放器只能补静音。只有真的在放东西时的采样才有意义。

### 为什么是 51 ms 而不是个位数

延迟全部来自缓冲，跟线材无关（10 ms 立体声走 USB 2.0 只要微秒级）。这 51 ms 的构成：

| 环节 | 耗时 | 能不能改 |
| --- | --- | --- |
| WASAPI 共享模式采集 | 22 ms | 改不了。四端点 loopback、独立采集引脚、独占模式全试过，都是这个数或更差 |
| 平板环形缓冲 | 18–36 ms | 已调到最小值（20 ms 天花板，低于它每个 HAL 周期都要欠载） |
| AAudio/MMAP 输出 | 0–50 ms | 已走 MMAP 原生通路 |
| 内核 socket 积压 | 0–30 ms | 已加主动修剪 |

用 Java AudioTrack 会是 126 ms：平台把这个 track 分到 `AudioOut_D`（PRIMARY，HAL 960 帧），那一路自带约 82 ms 的 mixer 管线。改走 AAudio 的 MMAP 通路才绕开它。

## 已知的坑

- **单实例锁与完整性级别**：早期版本用命名互斥体做单实例锁，但当持有者是登录自启（`wscript`）、启动者来自普通 shell 时，`CreateMutexW` 返回的不是 `ERROR_ALREADY_EXISTS`(183) 而是 `ERROR_ACCESS_DENIED`(5)，导致锁形同虚设、跑起两个桥接互相踩。现在两种错误码都判定为「已有实例」。
- **接收窗口不是延迟旋钮**：窗口必须留大（128 KB），因为 `adb forward` 是用户态中继、约 40 kB 在途；窗口小于它就会停顿然后整段倾泻。延迟靠主动修剪积压来控制，不靠缩窗口。
- **`kq` 曾经无上限**：平板内核接收队列原先没有任何约束，加上两颗晶振约 0.5% 的漂移，多出来的音频会一直沉进队列且永不排出——实测积到 2 MB（约 10.9 秒）。现在有硬上限 + 主动修剪。

## 支持的设备

主要在华为 MatePad 11（DBY-W09，Android 12 / arm64-v8a）上开发和验证。任何开了 USB 调试的 Android 8.0+ arm64 设备理论上都能用，但 APK 只打了 `arm64-v8a` 的 `.so`。

## 许可

MIT，见 LICENSE。
