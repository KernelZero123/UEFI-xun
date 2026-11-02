# xun — Redmi Pad SE 的 Mu UEFI 固件

Xiaomi Redmi Pad SE（`xun`，SM6225 / khaje，eMMC）的 Project Mu 平台包。

基于上游 [Project-Silicium/Mu-Silicium](https://github.com/Project-Silicium/Mu-Silicium)，
以同代机型的 `Platforms/Xiaomi/tapasPkg`（Redmi Note 12 4G，SM6225-AD）为模板移植。

## 现在的状态

**已经在真机上跑起来了。**

```
Si / Project Silicium 启动画面  ✓
Silicium UEFI Boot Manager      ✓
然后停在 "Failed to Find an Operating System!"
```

那个提示是**正常的**：Mu 本体、SEC、DXE、BDS、GOP 显示全部走通，只是镜像里还没有
可以交接的启动项。下一步接 UniXNU 才会有可启动的目标。

## 这个分支是什么

**纯 Mu**，不含 `UniXNUPkg` / `DarwinBootPkg`。

上游树里没有这两个包，而 fork 版本引用了它们，所以那边无论怎么改都编译不过。
这里只放能独立编译的最小集合。

## 文件

| 路径 | 说明 |
| --- | --- |
| `Platforms/Xiaomi/xunPkg/` | 平台包：dsc / fdf / dec / 库 / 构建脚本 |
| `Resources/Configs/xun.toml` | FD 基址 `0x5FC00000`、3 MiB、boot 镜像参数 |
| `Resources/DTBs/xun.dtb` | 从 OTA 提取并合并的真设备树 |

这些文件要放到 Mu-Silicium 树的对应位置上：

```
Platforms/Xiaomi/xunPkg        → <Mu 树>/Platforms/Xiaomi/
Resources/Configs/xun.toml     → <Mu 树>/Resources/Configs/
Resources/DTBs/xun.dtb         → <Mu 树>/Resources/DTBs/
Platforms/Xiaomi/xunPkg/Include/SimpleFbRotate.md  → 见下
```

`Binaries/xun/` 目前是 `Binaries/tapas/` 的副本。同 SoC 但不同板，充电、时钟、
TLMM、UFS 这些设备专属 DXE 还没换成本机自己的。

## 设备树怎么来的

OTA 线刷包里的 `vendor_boot.img`（header v4）含基础设备树：

```
model = "Qualcomm Technologies, Inc. Khaje SoC"   280,683 字节
```

`dtbo.img` 里 47 个 DTB 中，#45 / #46 是 `KHAJE IDP nopmi xun`。
把 `msm-id 0x206` 那个覆盖层叠到基础树上，得到 495,994 字节的完整设备树。

合并后确认包含 `nopmi`、`m84_42_03_0c`（华星面板）、`xiaomi`。

## 显示旋转

这块屏的安装方向是反的：不处理的话，开机 logo 和菜单都是 180 度倒的。

旋转做在 `SimpleFbDxe` 的 Blt 层，用 PCD 控制：

```
gSiliciumPkgTokenSpaceGuid.PcdFrameBufferRotate|180
```

`0` 为不旋转（默认，其他机型不受影响），`180` 为翻转。GOP 对外仍然报告
面板的原生分辨率，镜像在软件层完成。

需要同时改上游这两处才能编译：

- `Silicon/Silicium/SiliciumPkg/SiliciumPkg.dec` 增加 PCD 声明
- `Silicon/Silicium/SiliciumPkg/Drivers/SimpleFbDxe/SimpleFbDxe.inf`
  的 `[FixedPcd]` 里登记

`Platforms/Xiaomi/xunPkg/Include/SimpleFbRotate.md` 记录了改法。

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
Mu-xun.img                                           1.4 MiB
```

## 环境上踩过的坑

- **NuGet 需要 Mono**，而且 Mono 自带的 BoringSSL 连 Azure DevOps 会报
  `CERTIFICATE_VERIFY_FAILED`。解决办法是把 `mu_nasm` 下成本地源再改
  `nasm_ext_dep.yaml` 指向它。
- 包名必须是 `<id>.<version>.nupkg`，否则本地源认不出来。
- `nuget.exe` 下载失败时用 `NUGET_PATH` 指定手动下载的位置。
- 用 `-j3`，`-j8` 在 LTO 链接阶段会段错误。

## 刷机

```bash
fastboot boot Mu-xun.img
```

`boot` 不写分区，失败重启即恢复。首次尝试前建议先备份 `boot_a`。

## 还没做

- `Binaries/xun` 仍是 tapas 的副本
- ACPI 表：`Silicium-ACPI` 里没有 xun 的包
- UniXNU 未接入，所以没有可启动的操作系统
- 触控、面板初始化都还没做
