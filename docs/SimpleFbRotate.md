# SimpleFbDxe 180 度旋转

这块屏装反了，需要在 `SimpleFbDxe` 的 Blt 层翻转。

## 1. 声明 PCD

`Silicon/Silicium/SiliciumPkg/SiliciumPkg.dec`，紧跟 `PcdFrameBufferDelay`：

```
  gSiliciumPkgTokenSpaceGuid.PcdFrameBufferRotate|0|UINT32|0x0000A004
```

默认 `0`，不改变其他机型的行为。

## 2. 登记到模块

`Silicon/Silicium/SiliciumPkg/Drivers/SimpleFbDxe/SimpleFbDxe.inf` 的
`[FixedPcd]` 段加上：

```
  gSiliciumPkgTokenSpaceGuid.PcdFrameBufferRotate
```

## 3. 平台里打开

`xun.dsc` 的 `[PcdsFixedAtBuild]`：

```
  gSiliciumPkgTokenSpaceGuid.PcdFrameBufferRotate|180
```

## 4. 换 SimpleFb.c

直接用 `docs/SimpleFb.c.rotated` 覆盖同名文件。要点：

- `mFrameBuffer` 定义在 `FrameBufferGopBlt` 之后，需要前置声明
- 目标矩形映射到 `(W - x - w, H - y - h)`
- `EfiBltVideoFill` 只搬矩形，不用翻像素
- 带像素的操作逐行翻转，**用两个复用的小行缓冲**

## 关键：不要开整块缓冲

第一版是每次 blt 申请 `Width × Height × 4` 再整体翻转：

```
1200 × 1920 × 4 ≈ 9.2 MiB / 次
```

Mu 自己界面 blt 少，能过；接上 UniXNU 菜单后刷新频繁，内存耗尽，
表现为开机直接无显示，看起来像 UniXNU 有问题。

现在改成两个 `ScreenWidth × 4` 的行缓冲（约 4.8 KiB × 2），逐行翻转送出。

## 注意

这个 EDK2 版本的 `EFI_GRAPHICS_OUTPUT_BLT_OPERATION` 只有四个值，
没有 `EfiBltBltBufferToBltBuffer`。
