# xun — Redmi Pad SE 的 Mu UEFI 固件

Xiaomi Redmi Pad SE（`xun`，SM6225 / khaje，eMMC）的 Project Mu 平台包，
带 UniXNU 启动菜单。

基于上游 [Project-Silicium/Mu-Silicium](https://github.com/Project-Silicium/Mu-Silicium)，
以同代机型的 `Platforms/Xiaomi/tapasPkg`（Redmi Note 12 4G，SM6225-AD）为模板移植。

## 实际结果

**在真机上跑通了。**

```
Project Silicium logo / Boot Manager   ✓
UniXNU 菜单                            ✓   方向正确
1 Return to firmware
2 Restart
Booting Return to firmware in 2 s
↑/↓: select   Enter: boot   Esc: stop timer
按键控制                                ✓
```

底部统计显示 `no boot entry found`、`drivers 0 loaded / 0 failed`、`kits 0`、`acpi 0`。

**这是当前能到达的边界，不是缺陷。** 再往下需要 eMMC 主机驱动，见文末。

## 文件

| 路径 | 说明 |
| --- | --- |
| `Platforms/Xiaomi/xunPkg/` | 平台包：dsc / fdf / dec / 库 / 构建脚本 |
| `Resources/Configs/xun.toml` | FD 基址 `0x5FC00000`、3 MiB、boot 镜像参数 |
| `Resources/DTBs/xun.dtb` | 从 OTA 提取并合并的真设备树 |
| `docs/SimpleFb.c.rotated` | 带 180 度旋转的 SimpleFb.c 完整副本 |
| `docs/SimpleFbRotate.md` | 旋转改法的说明 |

放置位置：

```
Platforms/Xiaomi/xunPkg     → <Mu 树>/Platforms/Xiaomi/
Resources/Configs/xun.toml  → <Mu 树>/Resources/Configs/
Resources/DTBs/xun.dtb      → <Mu 树>/Resources/DTBs/
```

还需要往上游改三处才能编译，见 `docs/SimpleFbRotate.md`：

- `Silicon/Silicium/SiliciumPkg/SiliciumPkg.dec` 增加 `PcdFrameBufferRotate`
- `Silicon/Silicium/SiliciumPkg/Drivers/SimpleFbDxe/SimpleFbDxe.inf` 的 `[FixedPcd]` 登记
- `Silicon/Silicium/SiliciumPkg/Drivers/SimpleFbDxe/SimpleFb.c` 换成本仓库的版本

`UniXNUPkg` 来自 [UniXNU/UniXNU](https://github.com/UniXNU/UniXNU) 的
`package/UniXNUPkg`，放到 Mu 树根目录即可。`Binaries/xun/` 目前是
`Binaries/tapas/` 的副本：同 SoC 但不同板，充电、时钟、TLMM、UFS 这些
设备专属 DXE 尚未替换。

## 设备树怎么来的

OTA 线刷包里的 `vendor_boot.img`（header v4）含基础设备树：

```
model = "Qualcomm Technologies, Inc. Khaje SoC"   280,683 字节
```

`dtbo.img` 里 47 个 DTB 中，#45 / #46 是 `KHAJE IDP nopmi xun`。
把 `msm-id 0x206` 那个覆盖层叠到基础树上，得到 495,994 字节的完整设备树。
合并后确认包含 `nopmi`、`m84_42_03_0c`（华星面板）、`xiaomi`。

## 显示旋转

这块屏的安装方向是反的，需要 180 度。

旋转做在 `SimpleFbDxe` 的 Blt 层，由 PCD 控制：

```
gSiliciumPkgTokenSpaceGuid.PcdFrameBufferRotate|180
```

## 一个踩过的坑：不要在 Blt 里开大缓冲

最初的做法是每次 blt 申请 `Width × Height × 4` 的临时缓冲再整体翻转。
全屏一次就是 1200 × 1920 × 4 ≈ **9.2 MiB**。

```
只有 Mu 自己界面时    blt 少，能扛住
接上 UniXNU 菜单后    文字和高亮刷新频繁 → 内存耗尽 → 开机无显示
```

现象很迷惑：加 UniXNU 就黑屏，去掉就好，看起来像是 UniXNU 的问题。

现在改成两个复用的行缓冲，各 `ScreenWidth × 4` 字节（约 4.8 KiB），
逐行翻转再送出。常驻内存从 9.2 MiB 降到约 9.6 KiB，且不随刷新次数增长。

## 构建

```bash
cd <Mu 树>
NUGET_PATH=/path/to/nugetcli \
PATH=/path/to/llvm17:/path/to/venv/bin:$PATH \
python build_uefi.py -d xun
```

产物：

```
Build/xunPkg/RELEASE_CLANGPDB/FV/SILICIUM_UEFI.fd    3 MiB
Mu-xun.img                                           1.41 MiB
```

## 环境上踩过的坑

- **NuGet 需要 Mono**，且 Mono 自带的 BoringSSL 连 Azure DevOps 会报
  `CERTIFICATE_VERIFY_FAILED`。解决办法是把 `mu_nasm` 下成本地源再改
  `nasm_ext_dep.yaml` 指向它。
- 本地源包名必须是 `<id>.<version>.nupkg`。
- `nuget.exe` 下载失败时用 `NUGET_PATH` 指定手动下载的位置。
- 用 `-j3`，`-j8` 在 LTO 链接阶段会段错误。

## 刷机

```bash
fastboot boot Mu-xun.img
```

`boot` 不写分区，失败重启即恢复。建议先备份 `boot_a`。

注意 `Mu-xun.img` 的 DTB 是目标设备树，换机不可用。

## 为什么止步于此

菜单能出、能按键，但没有可启动项（`no boot entry found`）。要继续往下走，
缺的是一块**存储驱动**：

```
整棵 Mu 树里只有 UFSDxe，没有任何 MMC / SDHC 主机驱动
xun 是 eMMC，走的是 SDHCI
```

没有 eMMC 驱动就没有文件系统，UniXNU 读不到 `/EFI/UniXNU/` 下的配置和 kit，
也就没有可启动项。给 SM6225 的 SDCC 控制器写一个 UEFI 驱动是下一步的前提。

另外 `Binaries/xun` 仍是 tapas 的副本，`Silicium-ACPI` 也没有 xun 的包。

## 调试控制台：把日志画到屏幕上

xun 没有可用的串口，看不到固件日志。用 `-r DEBUG` 编译即可：

```bash
python build_uefi.py -d xun -r DEBUG
```

`SiliciumPkg.dsc.inc` 的 DEBUG 分支会自动切到 `FrameBufferSerialPortLib`，
所有 `DEBUG(...)` 输出直接画在屏幕上。这一版实测可读，能看到完整的 DXE 加载序列，
最后一行是 `UniXNUDxe.efi`。

**文字方向要单独处理。** 这个库直写显存，不经过 `SimpleFbDxe` 的 Blt 层，
`PcdFrameBufferRotate` 对它无效，面板装反的话日志是倒的。改法见
`docs/framebuffer-serial/PATCH.md`，三个文件在同目录下。

注意这是**两套独立的旋转逻辑**：`SimpleFb` 管 GOP（UniXNU 画的），
`FrameBufferSerialPortLib` 管日志文字。只设一边会出现"字正菜单倒"。

## 已知问题

### 高通 USB 驱动在 xun 上起不来

`Binaries/xun/QcomPkg/Drivers/Usb*` 是从 `Binaries/tapas/` 复制的，同 SoC 但板级
配置不同，加载后拿不到 USB core 状态，会在控制台无限刷：

```
UsbBusSupportDxe: Failed, Error: [Device Error]
Failed to get the Usb status for the USB core 0: Device Error
```

`Include/DXE.inc` 里已把五个高通 USB DXE 注释掉（标了 `DIAG: disabled`）：
`UsbPwrCtrlDxe`、`UsbfnDwc3Dxe`、`UsbMsdDxe`、`UsbDeviceDxe`、`UsbConfigDxe`。

`UsbBusDxe` / `UsbKbDxe` / `UsbMassStorageDxe` 保留着，所以刷屏仍会出现——
`UsbBusDxe` 找不到控制器时会一直重试。它不影响启动，但会盖住其他日志。

### 有一个驱动加载失败

DEBUG 日志里能看到：

```
Error: Image at 0x0005Fxxxxx did not start successfully.
```

不影响走到 UniXNU 菜单，但若要查显示相关问题，这个需要回头看。

## 当前边界

菜单能出、按键可用、方向正确，但 `no boot entry found`。原因是没有 eMMC 主机驱动，
UniXNU 读不到 `/EFI/UniXNU/` 下的配置和 kit。再往下需要：

- SM6225 的 SDCC/eMMC UEFI 驱动
- ACPI 表（`Silicium-ACPI` 无 xun 包）
- IPSW 里的 XNU 内核与 kext

