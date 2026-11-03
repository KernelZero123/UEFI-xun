# 屏幕调试控制台

`-r DEBUG` 编译时，`SiliciumPkg.dsc.inc` 的 DEBUG 分支会自动切换到：

```
SerialPortLib   → FrameBufferSerialPortLib
DebugLib        → BaseDebugLibSerialPort
PcdDebugPrintErrorLevel → 0x8007EE0F
```

也就是所有 `DEBUG(...)` 输出会被**画到屏幕上**。xun 没有可用的串口，这是唯一能
看到固件日志的途径，也是后面调 XNU 的前提。

## 问题：文字是倒的

`FrameBufferSerialPortLib` 直接往 framebuffer 内存写像素，**不经过 SimpleFbDxe 的
Blt 层**，所以 `PcdFrameBufferRotate` 对它无效。面板既然装反了，日志就是倒的。

## 修法：三处改动

本目录下的三个文件是改好的版本，直接覆盖对应文件：

```
FrameBuffer.h                → Silicon/Silicium/SiliciumPkg/Library/FrameBufferSerialPortLib/
FrameBufferSerialPortLib.c   → 同上
FrameBufferSerialPortLib.inf → 同上
```

改动要点：

1. `FrameBuffer.h` 加一行
   ```
   #define FB_ROTATE   FixedPcdGet32 (PcdFrameBufferRotate)
   ```
2. `.inf` 的 `[FixedPcd]` 登记 `gSiliciumPkgTokenSpaceGuid.PcdFrameBufferRotate`
3. `.c` 里做两件事：
   - `WriteFrameBuffer` 把字形单元的起始位置镜像到对角
     （`FB_WIDTH - (XPos+1)*GlyphWidth` / `FB_HEIGHT - (YPos+1)*GlyphHeight`）
   - `DrawGlyph` 行序倒过来、`DrawRow` 内位序倒过来

行序加位序都反过来，等价于把字形旋转 180 度；配合镜像后的位置，屏幕翻转后
正好正着。

## 验证

日志文字与 UniXNU 菜单方向应当一致。两边都正就对了。

## 注意

这是直接操作 framebuffer 的路径，和 `SimpleFbDxe` 的 Blt 旋转是**两套独立逻辑**，
两边都要设对。只改一边会出现"字正菜单倒"或反过来的情况。
