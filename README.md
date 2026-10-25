# 红果要翻转 · HONGGUO Flip

让**红果免费短剧**（`com.phoenix.read`）横屏播放时随设备方向**自由左右翻转**——
即使系统"自动旋转"处于锁定状态也能翻。

**Scope**: `com.phoenix.read` ｜ **API**: libxposed 101 / 102（实测 LSPosed 2.2.1 / API 102）
**版本**：1.0.0（versionCode 1）｜ **诊断日志 tag**：`HongguoFlip`

## 问题

横屏看剧时画面永远停在进入时的方向，转动设备毫无反应。真机系统日志定位的根因：

```
WindowManager: request=SCREEN_ORIENTATION_SENSOR_LANDSCAPE
               sensor=ROTATION_0,true  user=ROTATION_0,USER_ROTATION_LOCKED
```

系统"自动旋转"锁定（`USER_ROTATION_LOCKED`）时，Android 对**所有传感器类方向模式**
（`SENSOR_LANDSCAPE`、`USER_LANDSCAPE`、Manifest 的 `userLandscape` 等）一律不启用
旋转传感器监听，方向永远保持进入时的方向。原版 App 的声明与"改写为 sensorLandscape"
的方案在此状态下全部无效。

突破点：**固定方向请求（`LANDSCAPE` / `REVERSE_LANDSCAPE`）不受旋转锁限制**，
系统会忠实执行。

## 修复（零副作用，不改任何系统设置）

1. 在短剧横屏页 `ShortSeriesLandActivity` 注册模块自持的 `OrientationEventListener`
   （与系统旋转锁无关）：设备顶部朝**左** → 下发 `LANDSCAPE(0)`，
   顶部朝**右** → 下发 `REVERSE_LANDSCAPE(8)`；
2. hook `Activity.setRequestedOrientation` 与播放器 SDK 漏斗
   `FullScreenOperator.requestOrientation`：横屏页之外的固定横屏请求改写为
   `SENSOR_LANDSCAPE(6)`；
3. 全量诊断日志：每次方向请求的参数、宿主、调用栈与 SDK 决策值全程可查。

方向监听仅存在于红果横屏页内，退出即注销；不影响其他 App 与系统设置。

## 使用

1. 安装 APK，在 LSPosed 中启用「红果要翻转」（作用域默认已含 `com.phoenix.read`）
2. 强行停止红果免费短剧（或重启手机）
3. 进入短剧横屏全屏，左右转动设备即可翻转

验证：

```
adb logcat -s HongguoFlip
```

预期看到 `land onCreate -> SENSOR_LANDSCAPE(6) + attach listener` 与
`listener: device top-left/top-right -> LANDSCAPE(0)/REVERSE_LANDSCAPE(8)`。

- 不需要打开系统"自动旋转"（模块意义所在）；打开自动旋转时红果自身声明即可翻转。
- 日志完全空白 → 模块未加载：检查 LSPosed 版本（≥API 101）、模块启用、作用域、强停应用。

## 构建与源码

源码仓库（基于 [libxposed/example](https://github.com/libxposed/example) 模板）：
https://github.com/dunxuan/hongguo-orientation-fix

Release tag 格式：`versionCode-versionName`（当前 `1-1.0.0`）。

## License

Apache-2.0