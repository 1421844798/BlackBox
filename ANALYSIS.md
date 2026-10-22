# 哔哩哔哩 9.15.0 开屏广告布局分析报告

分析时间：2026-10-17
分析对象：`tv.danmaku.bili` versionName 9.15.0 / versionCode 9150300 / targetSdk 35
分析设备：vivo 测试机，Android 16（API 36）

## 一、分析方法

全程离线，不依赖任何在线服务：

1. `adb pull` 取出设备上的 `base.apk`（211 MB）
2. `aapt2 dump resources` 导出资源表（37 万行），检索 `skip` / `splash` 相关 id
3. 定位开屏 layout：`0x7f0c0180 layout/bili_app_ad_splash_fragment`
4. 该 APK 经过 AndResGuard 资源混淆，文件名已被重写 → 从资源表反查真实路径 `res/zoo.xml`
5. `aapt2 dump xmltree --file res/zoo.xml` 反编译二进制 XML
6. 用资源表把 `@0x7f09xxxx` 形式的 id 还原成可读名称
7. `aapt2 dump strings` 检索文案格式

## 二、开屏视图树结构

根节点为 `tv.danmaku.bili.splash.ad.widget.SplashContainerView`：

| 层级 | 控件类 | viewId | 说明 |
|---|---|---|---|
| 1 | SplashContainerView | `root_container` | 全屏根容器 |
| 2 | SplashImageView | `mainImageView` | 广告主图 |
| 2 | SplashVideoLayerView | `main_video_layer` | 视频广告层 |
| 2 | FrameLayout | `hot_tap_zone` | 点击热区 |
| 2 | ComposeView | `physic_view` | 摇一摇物理感应（Compose 渲染） |
| 2 | View | `lottie_shadow` | 阴影（默认 GONE） |
| 2 | SplashInteractView | `splash_interact_view` | 互动广告（默认 GONE） |
| 2 | DeliverySlideEventFrameLayout | `splash_button_container` | **全屏**按钮容器（match_parent×match_parent） |
| 3 | LinearLayout | `label_layout` | 「广告」标签容器，32×18dp |
| 4 | TextView | `label_ad` | 「广告」字样 |
| 3 | ConstraintLayout | `logo_layout` | **wrap_content 高度，贴屏幕底部** |
| 4 | BiliImageView | `logo` | B 站 logo |
| 4 | TextView | `count_down` | **真正的跳过/倒计时控件**，80×40dp，默认 GONE |
| 3 | FrameLayout | `splash_second_page_container` | 第二页广告 |
| 3 | AdLynxHostView | `lynx_container` | 新版 Lynx 渲染广告容器 |

## 三、决定实现方式的三个关键结论

### 1. 跳过按钮在屏幕「底部右侧」，不是右上角

`count_down` 的约束为：

```
layout_constraintTop_toTopOf    = logo_layout
layout_constraintBottom_toBottomOf = logo_layout
layout_constraintRight_toRightOf   = logo_layout
layout_marginEnd = 16dp
```

而 `logo_layout` 是 `layout_height=wrap_content` 且 `constraintBottom_toBottomOf` 父容器 —— 也就是贴在屏幕底部。

**结论：跳过按钮位于底部右侧（约 y≈0.94、x≈0.85）**。
这解释了为什么大量默认点击「右上角」的跳过工具对 B 站无效。

### 2. 父容器是全屏，绝不能「向上找可点击祖先」

`splash_button_container` 是 `-1 × -1`（match_parent × match_parent）。
如果按常规做法「当前节点不可点就点父节点」，点击落点会变成整个广告区域，直接跳转到广告落地页。

**结论：必须点击 `count_down` 节点自身的中心坐标，不能做祖先上溯点击。**

### 3. 文案格式是「数字在前」

字符串资源中与跳过相关的关键项：

| 字符串资源 | 含义 |
|---|---|
| `跳过` | 基础文案 |
| `%d 跳过` | 倒计时在前，如「5 跳过」 |
| `%02d \| 跳过` | 带分隔符变体 |

**结论：只匹配「跳过5」的正则会漏，必须用子串匹配「跳过」。**

## 四、其它版本兼容 ids

资源表里还存在其它跳过类 id，作为兼容池保留：

`tv_skip`、`skip`、`click_skip`、`bottom_container_skip`、`tv_dlc_draw_card_skip`

其中 `tv_skip`（0x7f094a32）**不在**开屏 layout 中，属于其它广告位/旧版本，因此只作降级候选而非主目标。

## 五、真机实测结论

在 Android 16 + vivo 上验证无障碍服务能力：

- ✅ 能正常读取 B 站完整视图树（实测拿到 `action_bar_root`→`drawer`→`content_layout`→`coordinatorLayout`，坐标 [0,0][1080,2400]）
- ✅ 能覆盖 B 站 6 种 Activity：`MainActivityV2`、`UnitedBizDetailsActivity`、`VertexActivity`、`LocalAuthorSpaceActivity` 等
- ✅ 手势能力 `capabilities=33`（含 retrieve window content + perform gestures）
- ⚠️ 该设备当前**不下发开屏广告**（连续 6 次完全冷启动均直接进入首页），因此「命中广告并自动点击」这一步未能在真机上复现

开屏页判定指纹（`splash_button_container` / `splash_interact_view` / `count_down` 等）在没有广告时不会命中，服务实测正确地保持了不动作。
