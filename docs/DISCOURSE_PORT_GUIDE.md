# 二开「山洞」客户端 · 完整改造清单

> 目标：把 fluxdo（Linux.do 专属）fork 为山洞论坛（forum.fuxi996.com）的定制客户端。
> 策略：**Linux.do 专属代码全部硬删除** + **品牌完全替换**（包名/名称/图标）+ **GPL v3 合规保留原作署名**。
>
> 目标站基本信息：
> - 主域名：`forum.fuxi996.com`
> - 图片 CDN：`cdn.fuxi996.com`（仅图片，无 S3 子域、无独立 MessageBus）
> - 无子路径部署、无 Cloudflare（如有也无害）
>
> 改造完仍可用：浏览/发帖/回帖/私信/通知/收藏/徽章/搜索/草稿/Markdown 编辑器/Discourse HTML 渲染/WebView 登录/Cookie + CSRF/外观与快捷键设置。
> 改造完移除：LDC 余额与授权、CDK 余额与授权、LDC 打赏、Connect 信任级别周期统计、自定义统计卡片、元宇宙页、信任级别要求页、剪贴板话题链接识别、邮箱链接登录 deep-link。

---

## 法律 · GPL v3 合规须知

原项目协议：**GNU GPL v3.0**。Fork 后必须做到：

1. **保留 `LICENSE` 文件不动**
2. **保留原作者署名**（README 标注「Fork from [fluxdo](原仓库 URL) by lingyan000」）
3. **你的修改版必须也以 GPL v3 开源**（发布 APK = 分发，需要附源码或源码下载地址）
4. **不能换成更宽松协议**（如 MIT/Apache）
5. **新增的 README/About 页中说明这是 fluxdo 的 fork**

如仅团队内部用（不公开分发 APK），上述义务暂不触发。

---

## 阶段 0 · 准备

```powershell
# 新建分支
git switch -c shandong-port

# 全局搜索关键词（前后对照用，改完应为 0 命中除注释外）
# 在 PowerShell 里：
rg -i "linux\.do|linuxdo|LinuxDo|fluxdo" lib/
```

---

## 阶段 1 · 核心域名替换

### 1.1 `lib/constants.dart`

```diff
- import 'config/sites/linuxdo.dart';
+ import 'config/sites/shandong.dart';
```

```diff
- static final SiteCustomization siteCustomization = linuxdoCustomization;
+ static const SiteCustomization siteCustomization = shandongCustomization;
```

```diff
- /// linux.do 域名
- static const String baseUrl = 'https://linux.do';
+ /// 山洞论坛域名
+ static const String baseUrl = 'https://forum.fuxi996.com';
```

### 1.2 新建 `lib/config/sites/shandong.dart`

**删除** `lib/config/sites/linuxdo.dart`，**新建**同目录文件：

```dart
import '../site_customization.dart';

/// 山洞论坛站点自定义配置
const shandongCustomization = SiteCustomization(
  linkSecurityConfig: _shandongLinkSecurityConfig,
);

const _shandongLinkSecurityConfig = LinkSecurityConfig(
  enableExitConfirmation: true,
  internalDomains: [
    '*.fuxi996.com',
    'localhost',
    '*.local',
    r'^127(?:\.(?:25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)){3}',
    r'^10(?:\.(?:25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)){3}',
    r'^169\.254(?:\.(?:25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)){2}',
    r'^192\.168(?:\.(?:25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)){2}',
    r'^172\.(?:1[6-9]|2\d|3[0-1])(?:\.(?:25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)){2}',
  ],
  riskyDomains: [
    // 通用短链服务（可选保留）
    'bit.ly', 'tinyurl.com', 't.co', 'goo.gl', 'ow.ly', 'is.gd',
    'tiny.cc', 'v.gd', 'x.co', 'bit.do', 'cutt.us',
  ],
  dangerousDomains: ['**aff='],
);
```

> AvatarGlow / 头衔特殊渲染（如「种子用户」全息文字）山洞用不到，全删。山洞如果有自己的群组要打光晕，后续按 `linuxdo.dart` 的格式自行添加。

---

### 1.3 Android Deep Link 域名

**文件**：`android/app/src/main/AndroidManifest.xml`

**第 50–87 行**（三组 intent-filter）：

```diff
- <!-- Deep Links for linux.do - 话题链接 -->
- <data android:scheme="https" android:host="linux.do" android:pathPrefix="/t/" />
- <data android:scheme="http"  android:host="linux.do" android:pathPrefix="/t/" />
+ <!-- Deep Links for 山洞 - 话题链接 -->
+ <data android:scheme="https" android:host="forum.fuxi996.com" android:pathPrefix="/t/" />
```

```diff
- <!-- Deep Links for linux.do - 用户链接 -->
- <data android:scheme="https" android:host="linux.do" android:pathPrefix="/u/" />
- <data android:scheme="http"  android:host="linux.do" android:pathPrefix="/u/" />
+ <!-- Deep Links for 山洞 - 用户链接 -->
+ <data android:scheme="https" android:host="forum.fuxi996.com" android:pathPrefix="/u/" />
```

**删除整个邮箱登录 intent-filter**（山洞不需要）：

```diff
- <!-- Deep Links for linux.do - 邮箱链接登录 -->
- <intent-filter android:autoVerify="true">
-     <action android:name="android.intent.action.VIEW" />
-     ...
-     <data android:scheme="https" android:host="linux.do" android:pathPrefix="/session/email-login/" />
-     <data android:scheme="http"  android:host="linux.do" android:pathPrefix="/session/email-login/" />
- </intent-filter>
```

> 自定义 scheme `fluxdo://` 改名见阶段 5。

---

### 1.4 `lib/services/deep_link_service.dart`

第 139–150 行：

```diff
- // 邮箱链接登录：/session/email-login/{token}
- if (uri.host == 'linux.do' &&
-     uri.path.startsWith('/session/email-login/')) {
-   _handleEmailLogin(context, url);
-   return;
- }
-
- // 其他 linux.do 链接：使用内置浏览器
- if (uri.host == 'linux.do' || uri.host.endsWith('.linux.do')) {
+ // 其他站内链接：使用内置浏览器
+ if (_isAppHost(uri.host)) {
    WebViewPage.open(context, url);
    return;
  }
```

第 209–217 行 `_handleEmailLogin` 方法整个删除。

第 253–264 行：

```diff
  static bool _canHandleUri(Uri uri) {
    if (uri.scheme == 'fluxdo') return true;
    if (uri.scheme != 'http' && uri.scheme != 'https') return false;
-   return _isLinuxDoHost(uri.host);
+   return _isAppHost(uri.host);
  }

- static bool _isLinuxDoHost(String host) {
+ static bool _isAppHost(String host) {
+   final baseHost = Uri.parse(AppConstants.baseUrl).host.toLowerCase();
    final normalizedHost = host.toLowerCase();
-   return normalizedHost == 'linux.do' ||
-       normalizedHost == 'www.linux.do' ||
-       normalizedHost.endsWith('.linux.do');
+   return normalizedHost == baseHost ||
+       normalizedHost == 'www.$baseHost' ||
+       normalizedHost.endsWith('.$baseHost');
  }
```

需要 import：`import '../constants.dart';`（应该已经有）。

同时把第 11 行附近的 `WebViewLoginPage` import 删除（如果只被邮箱登录用过；不删也不报错，分析器会提示 unused）。

---

### 1.5 `lib/services/clipboard_topic_link_service.dart`

第 26–29 行正则改成山洞域名：

```diff
- static final RegExp _linuxDoUrlRegex = RegExp(
-   r'(?:(?:https?:)?//)?(?:www\.)?linux\.do(?::\d+)?/[^\s<>"\]\)）}】》]+',
+ static final RegExp _siteUrlRegex = RegExp(
+   r'(?:(?:https?:)?//)?(?:www\.)?forum\.fuxi996\.com(?::\d+)?/[^\s<>"\]\)）}】》]+',
    caseSensitive: false,
  );
```

第 78 行：`_linuxDoUrlRegex` → `_siteUrlRegex`

第 168–171 行：

```diff
  static bool _isAllowedHost(String host) {
    final normalizedHost = host.toLowerCase();
-   return normalizedHost == 'linux.do' || normalizedHost == 'www.linux.do';
+   return normalizedHost == 'forum.fuxi996.com' ||
+       normalizedHost == 'www.forum.fuxi996.com';
  }
```

第 104 行注释中的 `linux.do` 改成 `forum.fuxi996.com`。

---

### 1.6 `lib/services/network/cookie/app_cookie_manager.dart`

第 304–321 行（针对 `connect.linux.do` 的 debug 日志），**整段删除**：

```diff
-    if (options.uri.host == 'connect.linux.do') {
-      final authCookies = requestCookies
-          .where((cookie) => cookie.name == 'auth.session-token')
-          .map(
-            (cookie) =>
-                '${cookie.domain ?? '<host-only>'}|${cookie.path ?? '/'}|len=${cookie.value.length}',
-          )
-          .toList(growable: false);
-      if (authCookies.isNotEmpty) {
-        debugPrint(
-          '[CookieManager] request cookies for connect.linux.do: $authCookies',
-        );
-      } else {
-        debugPrint(
-          '[CookieManager] request cookies for connect.linux.do: <none>',
-        );
-      }
-    }
```

---

### 1.7 `lib/services/network/doh/network_settings_service.dart`

第 1030–1038 行：

```diff
  List<String> _collectCommonHosts() {
    final preloaded = PreloadedDataService();
-   final hosts = <String>{
-     'connect.linux.do',
-     'ping.linux.do',
-     'cdn.linux.do',
-     'credit.linux.do',
-     'cdk.linux.do',
-   };
+   final hosts = <String>{
+     'cdn.fuxi996.com',
+   };
```

---

### 1.8 `lib/services/message_bus_service.dart`

第 72 行注释微调（可选）：

```diff
- String? _baseUrl;  // 独立域名（如 https://ping.linux.do），null 表示用主站
+ String? _baseUrl;  // 独立 MessageBus 域名，null 表示用主站（山洞用主站）
```

---

### 1.9 `lib/services/preloaded_data_service.dart`

第 40、41、256、262、265、268 行注释中的 `linux.do` 举例可改成山洞或直接保留——不影响功能，建议改：

```diff
-  String? _s3CdnUrl; // S3 CDN 域名（如 https://cdn3.linux.do）
-  String? _s3BaseUrl; // S3 基础 URL（如 //linuxdo-uploads.s3.linux.do）
+  String? _s3CdnUrl; // S3 CDN 域名（山洞不使用）
+  String? _s3BaseUrl; // S3 基础 URL（山洞不使用）
```

```diff
-  /// 获取 MessageBus 长轮询 base URL（独立域名，如 https://ping.linux.do）
+  /// 获取 MessageBus 长轮询 base URL（独立域名，山洞不使用）
   String? get longPollingBaseUrl => _longPollingBaseUrl;

-  /// 获取 CDN URL（如 https://cdn.linux.do）
+  /// 获取 CDN URL（如 https://cdn.fuxi996.com）
   String? get cdnUrl => _cdnUrl;
```

---

### 1.10 `lib/models/user.dart`

第 184 行注释：

```diff
-      // 替换 src="/... 为 src="https://linux.do/...
+      // 替换 src="/... 为 src="${baseUrl}/..."
```

代码逻辑不变（已经走 `UrlHelper.resolveUrlWithCdn`）。

---

### 1.11 `lib/services/network/webview/webview_adapter_settings_service.dart`

第 12 行注释举例 `cdn.linux.do` → `cdn.fuxi996.com`。

---

### 1.12 `lib/services/network/cookie/raw_cookie_writer.dart`

第 23 行注释举例 `https://linux.do` → `https://forum.fuxi996.com`。

---

### 1.13 `lib/services/discourse/discourse_service.dart`

第 99 行 doc comment：

```diff
- /// Linux.do API 服务
+ /// 山洞论坛 Discourse API 服务
```

---

### 1.14 `lib/utils/link_launcher.dart`

第 26 行注释：

```diff
-  final baseHost = baseUri.host; // 如 'linux.do'
+  final baseHost = baseUri.host; // 如 'forum.fuxi996.com'
```

---

## 阶段 2 · 删除 LDC 模块

### 2.1 删除整个 LDC reward 目录

```powershell
Remove-Item -Recurse F:\Projects\Flutter\fluxdo\lib\modules\ldc_reward
```

### 2.2 删除 LDC 相关单文件

```powershell
Remove-Item F:\Projects\Flutter\fluxdo\lib\services\ldc_oauth_service.dart
Remove-Item F:\Projects\Flutter\fluxdo\lib\widgets\ldc_balance_card.dart
Remove-Item F:\Projects\Flutter\fluxdo\lib\providers\ldc_providers.dart
Remove-Item F:\Projects\Flutter\fluxdo\lib\models\ldc_user_info.dart
```

### 2.3 删除 i18n reward 模块（4 个文件）

```powershell
Remove-Item F:\Projects\Flutter\fluxdo\lib\l10n\modules\reward\*.arb
Remove-Item F:\Projects\Flutter\fluxdo\lib\l10n\modules\reward -Recurse
```

### 2.4 删除帖子菜单里的「打赏 LDC」按钮

**文件**：`lib/widgets/post/post_item/widgets/post_footer_section/actions/menu_actions.dart`

删除第 117–149 行的 `if (!isGuest) Builder(...)` 块（含 `showLdcRewardSheet`）。

**文件**：`lib/widgets/post/post_item/widgets/post_footer_section/post_footer_section.dart`

- 第 8 行：删除 `import '../../../../../modules/ldc_reward/ldc_reward.dart';`
- 第 284 行：删除 `ref.watch(ldcRewardCredentialsProvider);`

---

## 阶段 3 · 删除 CDK 模块

```powershell
Remove-Item F:\Projects\Flutter\fluxdo\lib\services\cdk_oauth_service.dart
Remove-Item F:\Projects\Flutter\fluxdo\lib\widgets\cdk_balance_card.dart
Remove-Item F:\Projects\Flutter\fluxdo\lib\providers\cdk_providers.dart
Remove-Item F:\Projects\Flutter\fluxdo\lib\models\cdk_user_info.dart
```

CDK 相关 i18n 在 `auth_*.arb` 里（`auth_cdkConfirmMessage`），见阶段 6.4。

---

## 阶段 4 · 删除元宇宙 / 信任级别要求 / Connect 统计 / 自定义统计卡片

### 4.1 删除页面文件

```powershell
Remove-Item F:\Projects\Flutter\fluxdo\lib\pages\metaverse_page.dart
Remove-Item F:\Projects\Flutter\fluxdo\lib\pages\trust_level_requirements_page.dart
Remove-Item F:\Projects\Flutter\fluxdo\lib\pages\profile_stats_edit_page.dart
Remove-Item F:\Projects\Flutter\fluxdo\lib\widgets\profile_stats_card.dart
Remove-Item F:\Projects\Flutter\fluxdo\lib\models\connect_stats.dart
Remove-Item F:\Projects\Flutter\fluxdo\lib\models\profile_stats_config.dart
Remove-Item F:\Projects\Flutter\fluxdo\lib\providers\profile_stats_provider.dart
Remove-Item F:\Projects\Flutter\fluxdo\lib\providers\directory_providers.dart
```

### 4.2 删除 i18n metaverse 模块

```powershell
Remove-Item F:\Projects\Flutter\fluxdo\lib\l10n\modules\metaverse\*.arb
Remove-Item F:\Projects\Flutter\fluxdo\lib\l10n\modules\metaverse -Recurse
```

### 4.3 删除 i18n profileStats_* 键

`lib/l10n/modules/user/user_{zh,zh_TW,zh_HK,en}.arb` 这 4 个文件里，**删除全部以 `profileStats_` 开头的键**。可用编辑器搜索 `profileStats_` 一行行删，或脚本批量。

完整需要删除的键列表（在 user_zh_TW.arb 等同样位置）：

```
profileStats_addItems
profileStats_allItemsAdded
profileStats_availableItems
profileStats_bookmarkCount
profileStats_columnsPerRow
profileStats_dataSource
profileStats_daysVisited
profileStats_editTitle
profileStats_enabledItems
profileStats_guideMessage
profileStats_incompatibleSource
profileStats_layoutGrid
profileStats_layoutMode
profileStats_layoutScroll
profileStats_layoutSettings
profileStats_likesGiven
profileStats_likesReceived
profileStats_likesReceivedDays
profileStats_likesReceivedUsers
profileStats_loadError
profileStats_noItemsSelected
profileStats_postCount
profileStats_postsRead
profileStats_selectItems
（如有其他 profileStats_ 前缀的也一并删除）
```

### 4.4 改造 `lib/pages/profile_page.dart`

**删除 imports**（第 18、26、32–36、38–40 行）：

```diff
- import 'trust_level_requirements_page.dart';
- import 'metaverse_page.dart';
- import '../providers/ldc_providers.dart';
- import '../widgets/ldc_balance_card.dart';
- import '../providers/cdk_providers.dart';
- import '../widgets/cdk_balance_card.dart';
- import '../widgets/profile_stats_card.dart';
- import 'profile_stats_edit_page.dart';
- import '../services/ldc_oauth_service.dart';
- import '../services/cdk_oauth_service.dart';
```

**删除统计卡片引导相关**（第 66–69 行）：

```diff
-  // 统计卡片引导
-  static const String _guideKey = 'profile_stats_card_guide_shown';
-  final GlobalKey _statsCardKey = GlobalKey();
-  bool _guideShown = false;
```

并清理 `_guideKey` `_statsCardKey` `_guideShown` 在 `initState`、`didChangeDependencies`、`build` 等位置的全部引用（用 IDE 的「Find usages」逐一删除）。

**删除 `_reauthorizeLdc` 和 `_reauthorizeCdk` 方法**（第 230–270 行整段）。

**修改 `_openProfileEdit` 中的硬编码 URL**（第 277 行）：

```diff
- 'https://linux.do/u/$username/preferences/account',
+ '${AppConstants.baseUrl}/u/$username/preferences/account',
```

需 import：`import '../constants.dart';`

**删除 LDC/CDK 余额卡片整段**（约第 555–595 行，整个 `_buildBalanceSection` 之类的方法）。同时删除 build 里对它的调用。

具体定位：搜索 `LdcBalanceCard` 与 `CdkBalanceCard`，把它们所在的整个 `Widget` 方法和调用位置都删掉。

**删除自定义统计卡片**（第 620–634 行 `_buildStatsCardWithGuide` 方法整段 + 在 build 中的调用）。

**删除「信任级别要求」选项**（第 774–779 行 `_buildOptionTile`）。

**删除「元宇宙」选项**（第 790–796 行 `_buildOptionTile`）。

**修改登录按钮文案 key**（第 926 行）：

```diff
- label: Text(context.l10n.profile_loginLinuxDo, ...)
+ label: Text(context.l10n.profile_loginShandong, ...)
```

文案 key 重命名见阶段 6.2。

---

### 4.5 `lib/pages/settings_page.dart` 或其他设置页

搜索 `LdcRewardConfigTile` 引用：

```powershell
rg "LdcRewardConfigTile" lib/
```

只在 metaverse_page（已删）里用过。若你的当前分支有别处引用，一并删除。

---

## 阶段 5 · 品牌完全替换（fluxdo → shandong）

### 5.1 `pubspec.yaml`

```diff
- name: fluxdo
- description: "A new Flutter project."
+ name: shandong
+ description: "山洞论坛非官方客户端"
```

### 5.2 Android applicationId / namespace

**文件**：`android/app/build.gradle.kts`

```diff
- namespace = "com.github.lingyan000.fluxdo"
+ namespace = "com.fuxi996.shandong"
```

```diff
- applicationId = "com.github.lingyan000.fluxdo"
+ applicationId = "com.fuxi996.shandong"
```

> 包名格式 `com.fuxi996.shandong` 仅为示例，按你的偏好定（注意必须全小写，且与 Google Play / 签名证书绑定）。

### 5.3 Android Kotlin 包目录重命名

```powershell
# 旧路径
F:\Projects\Flutter\fluxdo\android\app\src\main\kotlin\com\github\lingyan000\fluxdo\
# 含: AndroidCdpBridge.kt, FluxdoApplication.kt, MainActivity.kt
```

操作：
1. 新建目录 `android/app/src/main/kotlin/com/fuxi996/shandong/`
2. 把 3 个 `.kt` 文件移过去
3. 修改每个文件首行 `package com.github.lingyan000.fluxdo` → `package com.fuxi996.shandong`
4. **重命名** `FluxdoApplication.kt` → `ShandongApplication.kt`，类名同步改为 `ShandongApplication`
5. 在 `AndroidManifest.xml` 中搜索 `FluxdoApplication`，若有 `android:name=".FluxdoApplication"` 改成 `.ShandongApplication`
6. 删除旧目录 `com\github\lingyan000\fluxdo\`

### 5.4 Android 显示名

**文件**：`android/app/src/main/AndroidManifest.xml` 第 20 行：

```diff
- android:label="FluxDO"
+ android:label="山洞"
```

### 5.5 iOS Bundle ID 和显示名

**文件**：`ios/Runner/Info.plist`

- `CFBundleIdentifier`：改成 `com.fuxi996.shandong`
- `CFBundleName` / `CFBundleDisplayName`：改成 `山洞` 或 `Shandong`

**文件**：`ios/Runner.xcodeproj/project.pbxproj`

搜索 `PRODUCT_BUNDLE_IDENTIFIER = com.github.lingyan000.fluxdo` 全部替换为新包名（一般有 Debug/Release/Profile 三处）。

### 5.6 App 图标 / 启动图

**Android**：替换 `android/app/src/main/res/mipmap-*/ic_launcher.png`（5 种密度尺寸）

**iOS**：替换 `ios/Runner/Assets.xcassets/AppIcon.appiconset/` 下所有尺寸

**Flutter 启动图**：若用 `flutter_launcher_icons` 包，更新其配置后重跑：

```powershell
flutter pub run flutter_launcher_icons
```

需要你提供山洞的 logo PNG（建议 1024×1024 透明背景）。

### 5.7 自定义 scheme `fluxdo://` 改名

**文件**：`lib/services/deep_link_service.dart` 第 254 行：

```diff
- if (uri.scheme == 'fluxdo') return true;
+ if (uri.scheme == 'shandong') return true;
```

**文件**：`android/app/src/main/AndroidManifest.xml`，搜索 `android:scheme="fluxdo"` 改成 `"shandong"`。

**文件**：`ios/Runner/Info.plist` 中 `CFBundleURLSchemes` 数组里 `fluxdo` 改成 `shandong`。

### 5.8 ContentProvider authorities

**文件**：`android/app/src/main/AndroidManifest.xml`

第 132、142 行：

```xml
android:authorities="${applicationId}.ota_update_provider"
android:authorities="${applicationId}.SuperClipboardDataProvider"
```

这两行用了 `${applicationId}` 占位，**会自动跟着新包名变，不用改**。

### 5.9 Flatpak 元数据

**删除**：`flatpak/com.github.lingyan000.fluxdo.metainfo.xml`

或重命名为 `com.fuxi996.shandong.metainfo.xml`，并把里面所有 `fluxdo` / `lingyan000` 替换。

如果不打 flatpak 包，直接删除整个 `flatpak/` 目录最省事。

### 5.10 全局清理残留 `fluxdo` 字符串

```powershell
rg -i "fluxdo|FluxDO|FluxdoApplication" --type-add 'cfg:*.{yaml,gradle,kts,xml,plist,kt,swift,dart,md}' -t cfg
```

逐一检查。重点位置：
- `lib/services/screen_track.dart` 等可能有 `fluxdo` 字样的服务名
- `pubspec.lock`（一般不用手改，`flutter pub get` 会重生成）
- `README.md`（保留致谢，其余改）
- `CHANGELOG.md`（保留历史）

---

## 阶段 6 · i18n 文案替换

> **改完 ARB 必须跑** `flutter gen-l10n`（项目用的是 slang，实际命令见 `pubspec.yaml` 或 `tool/gen_l10n.dart`）。

### 6.1 删除 LDC/CDK/metaverse/reward 相关键

阶段 2.3、3、4.2 已经删掉整个 reward 与 metaverse 模块的 ARB 文件，下面是散落在其他 ARB 的清理。

### 6.2 重命名 `profile_loginLinuxDo` → `profile_loginShandong`

`lib/l10n/modules/user/user_{zh,zh_TW,zh_HK,en}.arb`：

```diff
- "profile_loginLinuxDo": "登录 Linux.do",
+ "profile_loginShandong": "登录山洞",
```

英文版：`"profile_loginShandong": "Log in to Shandong"`，繁/港版按需翻译。

### 6.3 删除 `profile_metaverse` / `profile_trustRequirements` / `profile_ldcReauthSuccess` / `profile_cdkReauthSuccess`

在 `lib/l10n/modules/user/user_*.arb` 中搜索并删除这 4 个键所在的行。

### 6.4 删除 `auth_cdkConfirmMessage` / `auth_ldcConfirmMessage`

`lib/l10n/modules/auth/auth_{zh,zh_TW,zh_HK,en}.arb` 删除这两个键。

### 6.5 webview 登录标题

`lib/l10n/modules/webview/webview_*.arb` 中 `webviewLogin_title`：

```diff
- "webviewLogin_title": "登录 Linux.do",
+ "webviewLogin_title": "登录山洞",
```

### 6.6 关于页文案

`lib/l10n/modules/aboutUpdate/aboutUpdate_*.arb` 中 `about_legalese`：

```diff
- "about_legalese": "非官方 Linux.do 客户端\n基于 Flutter & Material 3",
+ "about_legalese": "非官方山洞论坛客户端\n基于 fluxdo 二开 · Flutter & Material 3",
```

英文：`"Unofficial Shandong client\nForked from fluxdo · Built with Flutter & Material 3"`。

### 6.7 邀请文案

`lib/l10n/modules/invite/invite_*.arb` 中 `invite_shareSubject`：

```diff
- "invite_shareSubject": "Linux.do 邀请链接",
+ "invite_shareSubject": "山洞邀请链接",
```

### 6.8 代理测试文案

`lib/l10n/modules/proxy/proxy_*.arb` 中 `httpProxy_testingProxy`：

```diff
- "httpProxy_testingProxy": "正在验证是否能通过当前代理访问 linux.do",
+ "httpProxy_testingProxy": "正在验证是否能通过当前代理访问山洞",
```

### 6.9 剪贴板检测描述

`lib/l10n/modules/settings/settings_*.arb` 中 `preferences_clipboardTopicLinkDetectionDesc`：

```diff
- "preferences_clipboardTopicLinkDetectionDesc": "回到应用时检测剪贴板中的 Linux.do 话题链接，并在底部询问是否打开",
+ "preferences_clipboardTopicLinkDetectionDesc": "回到应用时检测剪贴板中的山洞话题链接，并在底部询问是否打开",
```

### 6.10 同步代码端

`lib/providers/preferences_provider.dart:45` 注释 `Linux.do` → `山洞`。

### 6.11 登录页大标题

`lib/pages/login_page.dart:27`：

```diff
- 'Linux.do',
+ '山洞',
```

或改成读 `AppConstants.baseUrl` 提取 host 后展示。

### 6.12 重新生成 i18n 代码

```powershell
flutter pub get
# 项目用 slang，执行：
dart run tool/gen_l10n.dart
# 或如果有 build_runner 配置：
flutter pub run build_runner build --delete-conflicting-outputs
```

---

## 阶段 7 · README 与 Fork 致谢

新建/修改 `README.md` 顶部加入：

```markdown
# 山洞 · Shandong

非官方的 [山洞论坛 forum.fuxi996.com](https://forum.fuxi996.com) Flutter 客户端。

> 本项目 Fork 自 [fluxdo](https://github.com/lingyan000/fluxdo) by **lingyan000**，
> 在其基础上二开适配山洞论坛。
> 原项目遵循 GPL v3 协议，本项目同样以 GPL v3 开源。
```

`LICENSE` 文件**保持不动**。

如有 `lib/pages/about_page.dart` 加显示来源致谢（可选，但 GPL 推荐）。

---

## 阶段 8 · 验证清单

按顺序勾选：

- [ ] `flutter clean && flutter pub get` 无报错
- [ ] `dart run tool/gen_l10n.dart`（或对应 slang 命令）无报错
- [ ] `flutter analyze` 无 error（warning 可暂时忽略，主要看 `error -`）
- [ ] `flutter run -d <device>` 启动成功
- [ ] **App 启动后**：图标和标题显示为「山洞」
- [ ] **首屏话题列表**正常加载
- [ ] **WebView 登录** 走 `forum.fuxi996.com` 成功
- [ ] 登录后个人主页：
  - [ ] 无 LDC/CDK 余额卡片
  - [ ] 无 Connect 信任级别统计卡片
  - [ ] 无「信任级别要求」「元宇宙」入口
  - [ ] 登录/退出按钮文案为「登录山洞」
- [ ] 话题详情：底部菜单无「打赏 LDC」按钮，其他按钮正常
- [ ] 发帖、回复、私信、通知、搜索、收藏、徽章 全部可用
- [ ] 设置页所有子页打开不崩
- [ ] 站外链接点击：弹出外部链接确认（普通级别）
- [ ] `cdn.fuxi996.com` 的图片正常加载
- [ ] 重启 App 后保持登录态
- [ ] **Android**：deep link `https://forum.fuxi996.com/t/xxx` 在浏览器点击能拉起 App
- [ ] **自定义 scheme**：`shandong://topic/123` 能正常处理（用 `adb shell am start` 测试）

---

## 阶段 9 · 已知坑

1. **首次 i18n 生成失败**：删 key 后必须重跑生成命令；若代码里还有 `S.current.profileStats_xxx` 这类调用会编译失败。`flutter analyze` 帮忙定位。
2. **`AppConstants.baseUrl` 是 const**：编译期内联，运行时不可切换。如果将来想做多站点选择器需要重构成 `static String` + SharedPreferences。
3. **MessageBus 长轮询**：山洞用主站 `/message-bus/`，preloaded 检测不到独立域名会自动回退。如果通知不实时，看 `app_logs_page` 里的轮询日志。
4. **CF 挑战逻辑**：山洞没用 Cloudflare 的话，`cf_challenge_service` 走不到，没影响。
5. **包名换了**：原 fluxdo 用户更新需重装。建议在 release notes 注明。
6. **签名证书**：换包名后 Google Play 上架需新建 App 条目；自签 APK 也要重新生成 keystore（或沿用旧的，反正包名变了 Android 视为新应用）。
7. **Sticker 表情市场**：`lib/services/sticker_market_service.dart` 拉的 Linux.do 自定义接口，山洞没有这个 API 的话会 404，前端表现为空表情市场，不崩。如想彻底干净，可把入口（`lib/widgets/markdown_editor/sticker_market_sheet.dart`）也屏蔽。
8. **`screenshot_utils.dart:21`** 注释提到 `linuxdo-scripts`，纯注释参考来源，不用改。

---

## 阶段 10 · 改完后的提交建议

```powershell
git add -A
git commit -m "feat: fork to shandong forum client

- Replace baseUrl: linux.do -> forum.fuxi996.com
- Remove LDC/CDK/metaverse/connect modules
- Rebrand: fluxdo -> shandong (package: com.fuxi996.shandong)
- Update i18n strings and assets
- Keep GPL v3 + original attribution"
```

---

## 附录 · 一次性删除清单（PowerShell）

复制整段执行（执行前 commit 一次便于回滚）：

```powershell
# 1. LDC / CDK 模块
Remove-Item -Recurse -Force F:\Projects\Flutter\fluxdo\lib\modules\ldc_reward
Remove-Item -Force F:\Projects\Flutter\fluxdo\lib\services\ldc_oauth_service.dart
Remove-Item -Force F:\Projects\Flutter\fluxdo\lib\services\cdk_oauth_service.dart
Remove-Item -Force F:\Projects\Flutter\fluxdo\lib\widgets\ldc_balance_card.dart
Remove-Item -Force F:\Projects\Flutter\fluxdo\lib\widgets\cdk_balance_card.dart
Remove-Item -Force F:\Projects\Flutter\fluxdo\lib\providers\ldc_providers.dart
Remove-Item -Force F:\Projects\Flutter\fluxdo\lib\providers\cdk_providers.dart
Remove-Item -Force F:\Projects\Flutter\fluxdo\lib\models\ldc_user_info.dart
Remove-Item -Force F:\Projects\Flutter\fluxdo\lib\models\cdk_user_info.dart

# 2. 元宇宙 / 信任级别要求 / Connect / 自定义统计卡
Remove-Item -Force F:\Projects\Flutter\fluxdo\lib\pages\metaverse_page.dart
Remove-Item -Force F:\Projects\Flutter\fluxdo\lib\pages\trust_level_requirements_page.dart
Remove-Item -Force F:\Projects\Flutter\fluxdo\lib\pages\profile_stats_edit_page.dart
Remove-Item -Force F:\Projects\Flutter\fluxdo\lib\widgets\profile_stats_card.dart
Remove-Item -Force F:\Projects\Flutter\fluxdo\lib\models\connect_stats.dart
Remove-Item -Force F:\Projects\Flutter\fluxdo\lib\models\profile_stats_config.dart
Remove-Item -Force F:\Projects\Flutter\fluxdo\lib\providers\profile_stats_provider.dart
Remove-Item -Force F:\Projects\Flutter\fluxdo\lib\providers\directory_providers.dart

# 3. 站点配置
Remove-Item -Force F:\Projects\Flutter\fluxdo\lib\config\sites\linuxdo.dart
# 然后按阶段 1.2 新建 shandong.dart

# 4. i18n 模块
Remove-Item -Recurse -Force F:\Projects\Flutter\fluxdo\lib\l10n\modules\reward
Remove-Item -Recurse -Force F:\Projects\Flutter\fluxdo\lib\l10n\modules\metaverse

# 5. Flatpak（不打包就删）
Remove-Item -Recurse -Force F:\Projects\Flutter\fluxdo\flatpak
```

执行后跑：

```powershell
flutter clean
flutter pub get
flutter analyze
```

按 analyze 输出的报错位置（多半是仍在 import 已删文件、引用已删 key）依次修复，直到 0 error。

---

## 附录 · 改造工作量估计

- 阶段 1（域名替换）：1 小时
- 阶段 2–4（删除 + 改造 profile_page）：2–3 小时
- 阶段 5（品牌切换 + 包名 + 图标）：2 小时（图标素材准备另算）
- 阶段 6（i18n）：1 小时
- 阶段 7–8（README + 验证）：1 小时
- 修 analyze 报错 + 调试：1–2 小时

**总计：1 个工作日左右**（不含 logo 美术设计）。
