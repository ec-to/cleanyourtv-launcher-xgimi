# cleanyourtv-launcher-xgimi

[CleanYourTV](https://github.com/Gathin23/cleanyourtv) 启动器在 **XGIMI 极米Z6x Pro投影仪** 上的适配补丁集。
理论上适配于GMUI6系统，测试在GMUI 6.17上正常运行。
安装后通过adb停用原有**极米桌面（com.xgimi.home）** 方可替换为第三方桌面。

解决的问题：

1. **输入源 / Wi-Fi / 设置 三个胶囊点了没反应** —— 此固件的 `android.settings.SETTINGS` / `WIFI_SETTINGS` 无人应答，设置和 Wi-Fi 实际是厂商系统应用弹出的浮窗，不是 Activity。
2. **极米自带应用的桌面图标消失** —— 换掉极米桌面后，极米投屏 / 应用中心 / 资源管理器 / 投影管家 四个应用在首页看不到（它们只声明了手机类的 `LAUNCHER`，而 CleanYourTV 首页只扫 `LEANBACK_LAUNCHER`）。

> 本仓库是**补丁集，不含完整源码**。完整工程请取上游 [Gathin23/cleanyourtv](https://github.com/Gathin23/cleanyourtv) 的 `launcher/` 目录，再按下面步骤应用补丁。

## 环境

| 项 | 值 |
| --- | --- |
| 设备 | XGIMI 电视 / 投影仪，Android 12（SDK 31），GMUI 6（`com.android.newsettings` versionName 6.02.12.11） |
| 基线 | 上游 `launcher/` v3.3 / versionCode 12 |
| 补丁结果 | 01 → v3.5 / 14，02 → v3.6 / 15 |
| 编译 | Gradle 8.9 + JDK 21 + build-tools 34.0.0（`./gradlew assembleRelease`） |

## 应用补丁

```bash
git clone https://github.com/Gathin23/cleanyourtv.git
cd cleanyourtv
git apply patches/01-xgimi-entrances.patch    # 三大入口修复
git apply patches/02-xgimi-home-icons.patch   # 四个图标恢复
cd launcher && ./gradlew assembleRelease
adb install -r app/build/outputs/apk/release/app-release.apk
```

两个补丁按顺序应用，前缀路径是 `launcher/...`，在仓库根目录执行即可。

---

## 一、输入源 / Wi-Fi / 设置 三个入口

### 现象与根因

```
$ adb shell cmd package resolve-activity -a android.settings.SETTINGS
No activity found
```

这台机的设置与 Wi-Fi 都不是 Activity，而是 `com.android.newsettings`（`sharedUserId=android.uid.system`，uid 1000）里 `com.xgimi.newsettings.SettingService` 弹出的 **`SYSTEM_DIALOG` 浮窗**，浮在任意界面上。`Settings` / `WIFI_SETTINGS` 两个标准 action 没有任何组件应答，所以 CleanYourTV 的三条 `systemIntent(...)` 路线全部落到空处。

### 逆向依据

反编译极米桌面 `com.xgimi.home.apk`（baksmali 2.5.2）拿到 `SettingPageManager.startServiceForSetting` 的原始调用链，三个要点：

- 页面标识放在 Intent 的 **extra `data`**（String），不是 `-d` URI，也不是 `page` / `pageId` / `path` 之类的键；
- 必须带 **`unlocked=true`**（极米桌面自己的调用就带，缺了服务不开窗）；
- 用 **`startService`** 发给 `com.android.newsettings/com.xgimi.newsettings.SettingService`，action `com.xgimi.settings.SETTINGS`。

页面字符串（GMUI 6 口径）：

| 入口 | extra `data` |
| --- | --- |
| 设置 | `Settings://com.xgimi.settings/main` |
| Wi-Fi | `Settings://com.xgimi.settings.net/wifi_misckey` |
| 输入源 | 不走这条，见下 |

输入源走标准 TV input：action `com.xgimi.action.hdmiPlayer`，数据是 passthrough channel URI
`content://android.media.tv/passthrough/com.xgimi.tvinput%2F.inputservice.Hdmi1TvInputService%2FHW1` +
`vnd.android.cursor.item/channel`，落到 `com.xgimi.xhplayer/.InputActivity`。该机只注册了 1 个 TV input，所以菜单只有 HDMI 1 一行。

### 改动（`patches/01`，8 个文件）

| 文件 | 改动 |
| --- | --- |
| `SystemSettings.java` | 新增 `startServicePage(Context, String standardAction)`：无标准 settings activity 时改走厂商服务页；失败返回 false，调用方保留原路线 |
| `config.xml` | 新增 `settings_service` / `settings_service_action` / `settings_service_page` / `wifi_service_page` 四个键，并把 `input_action` 设为 `com.xgimi.action.hdmiPlayer` |
| `MainActivity.java` | 新增 `openSettingsEntry(...)`，替换 5 处旧的 `systemIntent(Settings.ACTION_*)` 调用点 |
| `RemoteKeys.java` | `settings` / `wifi` 两个遥控键改为先试服务页 |
| `HomeKeyService.java` | 设置键（含 `settings_key_clear_task` 分支）改为先试服务页 |
| `LauncherSettingsActivity.java` | 「电视设置」入口同样先试服务页 |
| `AndroidManifest.xml` | `<queries>` 补 `com.android.newsettings`、`com.xgimi.xhplayer` |
| `build.gradle` | `versionCode 13 → 14`，`versionName '3.4' → '3.5'` |

设计上保持上游的通用结构：`config.xml` 四个键留空即关闭该能力，其它机型只需要填自己固件的服务组件和页面字符串。

### 实机验证（`dumpsys window`）

点击 Wi-Fi 胶囊后，顶层窗口为 `com.android.newsettings`、`ty=SYSTEM_DIALOG`、`mOwnerUid=1000`，且 `launchedFromPackage=com.gathin.tvhome`（launcher 自己的 uid 10045 发起），即浮窗确实由启动器拉起，而不是残留窗口。

### 已尝试且不通（避免重复踩坑）

- 把深链当 `-d` URI 传；改用 `--es page` / `pageId` / `path` / `key` / `nodeId`、`--ei page`、`--es from` 等 13 种键名 —— 全部落回首页；
- `com.xgimi.misckey.MisckeyService`（action `com.xgimi.misckey.MISCKEY` / `com.android.newsettings.MISCKEY`）只能拉起极米快捷面板，指定的 Wi-Fi 子页被忽略；
- `com.android.newsettings/...mock.MockActivity`、`com.xgimi.config/.MainActivity` 都是空壳；
- 该固件 `logcat` 拿不到 app / ATMS 日志，排错只能靠 `dumpsys activity activities` 与截图。

---

## 二、极米投屏 / 应用中心 / 资源管理器 / 投影管家 图标

### 根因

CleanYourTV 首页网格来自 `MainActivity.reloadTiles()` 里那一次 `CATEGORY_LEANBACK_LAUNCHER` 查询，而极米这四个自带应用只声明了手机侧的 `CATEGORY_LAUNCHER`：

```
$ adb shell cmd package query-activities --brief -a android.intent.action.MAIN -c android.intent.category.LEANBACK_LAUNCHER
→ 14 activities found，里面没有 wirelessscreen / newappmarket / filemanager / manager
```

| 应用 | 包 | 入口 Activity |
| --- | --- | --- |
| 极米投屏 | `com.xgimi.wirelessscreen` | `.activity.NewMainActivity` |
| 应用中心 | `com.xgimi.newappmarket` | `com.xgimi.module_appmarket.view.AppMarketActivity` |
| 资源管理器 | `com.xgimi.filemanager` | `com.xgimi.module.home.HomeActivity` |
| 投影管家 | `com.xgimi.manager` | `.ui.activitys.MainActivity` |

顺带说明：同一个仓库的「应用库」（`AppLibraryDialog`）两类 category 都扫，所以这四个在库里能看到，只是首页网格拿不到 —— 这也是为什么修复不需要动 Java。

### 改动（`patches/02`，2 个文件，全是配置）

```xml
<string-array name="library_apps" translatable="false">
    <item>com.xgimi.wirelessscreen</item>
    <item>com.xgimi.newappmarket</item>
    <item>com.xgimi.filemanager</item>
    <item>com.xgimi.manager</item>
</string-array>
```

外加 `versionCode 14 → 15` / `versionName '3.5' → '3.6'`。

`library_apps` 是上游现成机制：启动时把这批包预置进「应用库已添加」集合（`AppAdapter.reload()` 里的 `libraryPreload`），对这类只有 `LAUNCHER` 声明的包会用 `CATEGORY_LAUNCHER` 查询补进首页网格。`LibraryApps.visible()` 的判定是 `!removed && (explicitlyAdded || (tvApp && !themeHidden))`，因此不依赖 `app_library` 开关；预置进去的图标在应用库界面里依然可以移除，移除后不会自己回来。

### 实机验证

逐个点图标，`dumpsys activity activities | grep mResumedActivity`：

```
mResumedActivity: com.xgimi.filemanager/com.xgimi.module.home.HomeActivity
mResumedActivity: com.xgimi.newappmarket/com.xgimi.module_appmarket.view.AppMarketActivity
mResumedActivity: com.xgimi.manager/.ui.activitys.MainActivity
mResumedActivity: com.xgimi.wirelessscreen/.activity.NewMainActivity
```

### 两个使用注意

- 本机 `rows_below_dock=0`：首页初始只显示底部 dock 那一排，应用网格在它下方，需要按「下」翻过去，四个新图标就落在网格里（不是 dock 上）。想固定到 dock，把包名加进 `dock_apps` 即可。
- 电视触摸屏在这个启动器上，网格里的点按偶尔只移动焦点不触发点击；用遥控器方向键走到图标上再按 OK 一定可以打开（四个图标已用这个方式逐一验证）。

---

## 编译与安装

```powershell
$env:JAVA_HOME="<jdk-21>"
$env:ANDROID_HOME="$env:LOCALAPPDATA\Android\Sdk"
cd launcher
.\gradlew.bat assembleRelease --console=plain
adb -s <电视IP>:5555 install -r app\build\outputs\apk\release\app-release.apk
adb -s <电视IP>:5555 shell am start -W -n com.gathin.tvhome/.MainActivity
```

截图请用 `adb shell screencap -p /sdcard/x.png` + `adb pull`（`adb exec-out screencap -p > x.png` 在 PowerShell 会注入 BOM 把 PNG 写坏）。

## 边界与未做的事

- 三个入口只在这一台 GMUI 6 设备上验证过；别的固件请按自己的组件/页面字符串改 `config.xml` 的四个键。
- 设置服务除了 `main` 和 `wifi_misckey` 两个页面，其它子页（显示、声音等）的 `data` 字符串尚未逐个从极米桌面里翻出来，需要时再补。
- 四个图标只验证到「能进各自首页」，各应用内的功能/登录没有深入测。
- 四条应用的「桌面图标」本质是 leanback-only 首页 + 手机类应用之间的口径差，上游若将来给首页网格加了 `LAUNCHER` 兜底，这段配置就可以删掉。

## License

补丁与文档遵循上游 MIT 许可（Copyright (c) 2026 Gathin），见 [LICENSE](LICENSE)。上游内置的 Outfit 字体为 SIL OFL 1.1，见上游 `launcher/FONT-LICENSE-OFL.txt`。
