# 小米平板 5 Pro (elish) 开机默认横屏说明

本文介绍如何在 LineageOS 23.2 (Android 16) 上，让小米平板 5 Pro (elish) 实现
**开机初始即为横屏**，同时**保留自动旋转、可正常切换竖屏**的效果。

> 说明：面板物理竖装（`ROTATION_0` 为竖屏 1600×2560），本文方案通过 Android
> 的显示旋转机制实现"系统默认方向为横屏"，而非物理翻转面板。

---

## 一、效果

- 开机启动动画、锁屏、首屏**默认横屏**（2560×1600）。
- 解锁后**自动旋转正常**，重力方向与屏幕旋转一致，可自由转竖屏。
- **触摸完全对齐**，无需额外触摸映射配置。

## 二、原理

Android 的 `DisplayRotation` 在初始化时会读取一个 `ro.` 系统属性来决定
**开机默认旋转方向**，见 `services/core/java/com/android/server/wm/DisplayRotation.java`
中的 `readDefaultDisplayRotation()`：

- `ro.bootanim.set_orientation_<physical_display_id>`
- `ro.bootanim.set_orientation_logical_<logical_display_id>`

取值为 `ORIENTATION_90` / `ORIENTATION_180` / `ORIENTATION_270` 时，对应系统
默认方向为 `ROTATION_90/180/270`；未设置时默认为 `ROTATION_0`（竖屏）。

该机制**只改变系统默认方向，不改变面板物理方向与传感器自然方向**，因此
既能开机默认横屏，又不会像 `ro.surface_flinger.primary_display_orientation`
那样破坏自动旋转和触摸。

> 对比：`ro.surface_flinger.primary_display_orientation=ORIENTATION_90` 会强制
> **全局横屏**（强制横屏后无法转竖屏，且自动旋转重力偏移 90°、触摸错位），
> 不适用于"仅开机默认横屏"的需求。

## 三、实现方式（KernelSU 开机脚本，无需重编译）

`ro.` 属性在重启后会丢失，且必须在 WindowManager/`system_server` 启动前设置。
KernelSU 的 `post-fs-data` 阶段在 `system_server` 之前执行，可在此处注入属性。

### 1. 创建开机脚本

在设备上（需 root，KernelSU）执行：

```sh
su -c 'mkdir -p /data/adb/post-fs-data.d'
su -c 'printf "#!/system/bin/sh\nresetprop ro.bootanim.set_orientation_logical_0 ORIENTATION_90\n" > /data/adb/post-fs-data.d/boot-landscape.sh'
su -c 'chmod 755 /data/adb/post-fs-data.d/boot-landscape.sh'
```

脚本内容：

```sh
#!/system/bin/sh
resetprop ro.bootanim.set_orientation_logical_0 ORIENTATION_90
```

`logical_display_id` 对主屏通常为 `0`；如需其它物理屏，可用
`ro.bootanim.set_orientation_<physical_display_id>`。

### 2. 完整重启验证

```sh
adb reboot
adb wait-for-device
adb shell getprop ro.bootanim.set_orientation_logical_0   # 应输出 ORIENTATION_90
```

重启后应默认横屏，自动旋转与触摸正常。

### 3. 临时手动测试（不持久化）

```sh
adb shell "su -c 'resetprop ro.bootanim.set_orientation_logical_0 ORIENTATION_90'"
adb shell "su -c 'stop' && su -c 'start'"   # 软重启 framework 生效
```

## 四、替代方案（需重编译）

若不想用 KernelSU 脚本，可在 ROM 设备树中固化属性（重新编译 ROM）：

- 在 `device/xiaomi/elish/device.mk` 或对应 `*.prop` 中加入：

```makefile
PRODUCT_PROPERTY_OVERRIDES += \
    ro.bootanim.set_orientation_logical_0=ORIENTATION_90
```

## 五、撤销

删除开机脚本并重启即可恢复默认竖屏：

```sh
adb shell "su -c 'rm /data/adb/post-fs-data.d/boot-landscape.sh'"
adb reboot
```

## 六、环境信息

- 设备：小米平板 5 Pro (elish)
- ROM：LineageOS 23.2 (Android 16, userdebug)
- Root：KernelSU
- 面板：`qcom,mdss-dsi-panel-width=800`(dual 1600)、`height=2560`
  （`ROTATION_0` = 竖屏 1600×2560）
