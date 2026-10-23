# DYYY: 抖音 40.7.0 首页视频、评论毛玻璃与主页视频详情布局异常

- **Date**: 2026-10-24
- **Scope**: iOS 抖音 40.7.0，首页 Feed、评论区、个人主页视频详情
- **Severity**: P1
- **Affected systems**: `DYYY.xm`、`DYYYUtils.m`、`DYYYSettingsHelper.m`、`DYYYSettingViewController.m`、`DYYYSettings.xm`、`AwemeHeaders.h`

## 一、Bug 现象

- 触发条件:
  - 开启 DYYY 首页全屏，在抖音首页上下切换视频。
  - 打开任意视频评论区。
  - 从个人主页点进视频，再上下切换视频。
- 用户观察:
  - 首页视频没有完整沉浸到状态栏和底栏之间，画面整体偏上并留有底部空隙。
  - 评论昵称、时间和正文背后先出现白色框，停留一段时间后才消失；部分版本最终仍保留黑色文字或白色底。
  - 个人主页视频切换后只显示半屏，下方残留一条大黑条或上一视频的一部分。
  - 评论毛玻璃透明度调节不精细，旧版调到高透明度仍不是真正全透明；曾出现 80% 时直接变黑。
- 不影响的范围:
  - 普通视频播放、下载、设置页导航和评论操作逻辑不在本次修改范围内。

### 复现与成功标准

- 最小复现:
  1. 自签注入 DYYY 40.7.0-18 或更早版本。
  2. 进入首页 Feed，观察视频上下边界。
  3. 打开评论区，观察评论文字背景和毛玻璃。
  4. 进入个人主页，点开视频并上下切换。
- 成功标准:
  - 首页视频从状态栏覆盖到底栏上沿，底栏之外的画面没有额外空白。
  - 评论列表首帧开始，昵称、时间、正文、回复展开入口均没有白色或黑色底板。
  - 透明度输入 `0` 到 `100` 连续有效，`100` 时评论面板不残留 `UIVisualEffectView` 或 tint 层。
  - 个人主页视频切换后保持完整竖屏画布，底部没有黑条和上一视频残留。

## 二、Bug 产生的原因

```text
首页视频尺寸异常
  └─ Proximate cause: Feed 表格被按“底栏上方内容高度”布局
      └─ Underlying cause: 40.7.0 中 AWEFeedTableView 仍保留旧的底部预留高度
          └─ Root cause: DYYY 只修改了 cell/contentView，没有在表格最终布局阶段统一恢复宿主全高

个人主页视频详情黑条
  └─ Proximate cause: 视频画面仍按 A spect-fit 的方式缩在容器内
      └─ Underlying cause: TTPlayerView.scaleType 被错误设置为 1
          └─ Root cause: 对 40.7.0 TTPlayerView 缩放枚举理解相反；二进制行为表明 0 才是 aspect-fill，1 是 aspect-fit

评论毛玻璃丢失与文字底板闪现
  └─ Proximate cause: 评论 Swift virtual view 在列表布局后重新写入背景色或富文本背景属性
      └─ Underlying cause: 旧清理名单只覆盖部分 Label/Fold 类，没有覆盖 UserName/Time/Text/VirtualView
          └─ Root cause: 清理仅依赖延迟异步执行，首帧和后续复用 cell 之间仍存在时间窗

透明度调节不精确
  └─ Proximate cause: UI 新百分比与旧 0-1 值混用，映射不统一
      └─ Underlying cause: 模糊层、tint 层和对“透明度”的语义没有一起更新
          └─ Root cause: 100% 时仍保留模糊视图，并且 tint 强度在深色背景下会快速累积成黑色
```

### Layer 1: Feed 与视频详情画布

- `AWEFeedTableView` 是 Feed 视频页的实际列表宿主。即使 cell/contentView 被拉高，列表自身仍可能保留底栏预留高度。
- `TTPlayerView` 的 `scaleType` 是真实抖音播放器属性。对 `AwemeCore` 的静态反汇编确认 `0` 对应 `aspectFill`，`1` 对应 `aspectFit`。旧实现强制设置为 `1`，只会让视频更小并继续保留黑边。
- 个人主页详情页还依赖 `AWEAwemeDetailTableViewCell` 和 `AWEAwemeBaseViewController` 的播放器 frame 回调，仅修改 `TTPlayerView` 不足以覆盖切换视频的时序。

### Layer 2: 评论 Swift virtual view

- 40.7.0 的真实评论标签不是普通 `UILabel` 的单一实现，实际路径包括：
  - `CommentCellTimeVirtualView`
  - `CommentCellUserNameComponent...UserNameVirtualView`
  - `CommentCellTextComponent...CommentLabelVirtualView`
  - `_TtCV...CommentVirtualLabel`
- 这些类会通过 `backgroundColor`、富文本 `NSBackgroundColorAttributeName`、`YYTextBorder` 或 `YYTextBorderArray` 显示底板。
- `AWEListKitMagicCollectionView.layoutSubviews` 是列表完成布局的关键时点；只在打开评论后延迟清理，会留下首帧白框。

### Layer 3 (root): 透明度的单一数据源

- 新设置键是 `DYYYCommentBlurTransparencyPercent`，旧安装仍可能保留 `DYYYCommentBlurTransparent` 的 0-1 值。
- 设置值必须从 UI 写入、发布通知、评论控制器接收，再统一换算为 `0.0...1.0` 后才可同时驱动 blur、tint 和清底。
- `100%` 必须是“无模糊层”，而不是让模糊视图以极低 alpha 继续存在。

## 三、Bug 排查 + 修复思路

### 1. Phase 1 - 自顶向下定位

| 层级 | 文件:行 | 关键值 / 调用 |
|---|---|---|
| 首页列表 | `DYYY.xm:7759` | `AWEFeedTableView.layoutSubviews` 将高度恢复为 `self.superview.bounds.height` |
| 首页 cell | `DYYY.xm:7694` | `viewDidLayoutSubviews` 处理 Feed cell 内容高度 |
| 播放器 | `DYYY.xm:8487` | `TTPlayerView.layoutSubviews` 保存/恢复原始 `scaleType` |
| 播放器枚举 | `AwemeHeaders.h:1533` | `scaleType`、`setScaleType:`、`updateVideoFrame` |
| 主页详情 | `DYYY.xm:8441` | `AWEAwemeDetailTableViewCell.playerNotCoverBottomBarEnable` 返回 `NO` |
| 主页详情 frame | `DYYY.xm:8451` | `adjustVideoFrameIfNeededWithPanelType:` 调用全屏 frame |
| 评论清理入口 | `DYYY.xm:7106` | `AWEListKitMagicCollectionView.layoutSubviews` 同步清理 |
| 评论列表类判断 | `DYYYUtils.m:1207` | 补充 `VirtualView`、`UserName`、`Time`、`Content` |
| 透明度读取 | `DYYYUtils.m:1192` | 新百分比键优先，兼容旧 0-1 键 |
| 模糊应用 | `DYYYUtils.m:1287` | `applyCommentPanelBlurToRootViewController:transparency:` |
| 清理根因 | `DYYYUtils.m:1561` | `clearCommentBlurBackgroundsNowInView:` 同步清理 |
| 设置写入 | `DYYYSettingsHelper.m:20` | 写入后主线程发布透明度变化通知 |
| 设置 UI | `DYYYSettingViewController.m:158` | 评论区毛玻璃开关和百分比输入项 |

Phase 1 出口的命名假设:

- H1: 首页视频的底部空隙由 `AWEFeedTableView` 的旧高度约束造成，需要恢复宿主全高。
- H2: 个人主页视频详情被错误使用 `scaleType=1`，并且 detail cell 的 frame 回调没有覆盖切换过程。
- H3: 评论首帧白框来自 Swift virtual view 在 `layoutSubviews` 后重写背景/富文本属性。
- H4: 透明度问题来自新百分比键、旧 0-1 键及 blur/tint 映射不一致。

### 2. Phase 2 - 全链路校验

本次没有在 Windows 环境伪造真机日志，也没有把临时日志带入发布包。对 40.7.0 的二进制定位和实际包验证如下：

- `AwemeCore` 静态符号/反汇编确认：
  - `-[TTPlayerView scaledRectFrom:ratio:scaleType:]`
  - `-[TTPlayerView setScaleType:]`
  - `-[TTPlayerView updateVideoFrame]`
- 真实评论 Swift 类名从 `AwemeCore` 中确认，覆盖昵称、时间、正文和 virtual label。
- 设置到应用的静态链路确认：
  1. `DYYYSettingViewController` 写入百分比字符串。
  2. `DYYYSettingsHelper setUserDefaults:forKey:` 发布 `DYYY_COMMENT_BLUR_TRANSPARENCY_DID_CHANGE_NOTIFICATION`。
  3. `AWECommentContainerViewController` 在 `viewWillAppear:` 注册、在 `viewDidDisappear:` 移除观察者。
  4. `dyyyApplyCommentBlurIfNeeded` 读取 `commentBlurTransparencyPercent` 并调用 blur 方法。
- 发布前调试前缀清理命中数为 0。本次修复没有新增运行时调试前缀。

在没有 USB 真机控制台的情况下，不能把“CI 编译成功”等同于“截图中问题已消失”。因此本报告只把编译、Mach-O 和包结构列为已验证，把 UI 首帧和切换动画列为待真机验证项。

### 3. Phase 3 - 分层根因

| 层 | 证据 |
|---|---|
| Feed 高度 | cell 内容高度与 `AWEFeedTableView` 宿主高度分开控制，旧代码仅处理前者 |
| 播放器缩放 | `AwemeCore` 反汇编显示 `scaleType=0` 为填充、`scaleType=1` 为适应 |
| 评论标签 | 真实类名包含 `VirtualView`、`UserName`、`Time`、`Content`，旧名单覆盖不足 |
| 首帧闪现 | 旧清理是延迟异步执行，列表 `layoutSubviews` 期间没有同步清理 |
| 透明度 | 新 UI 使用 0-100，旧值使用 0-1，必须有兼容归一化和 `100%` 的移除分支 |

### 4. 方案选型

| 候选 | 评估 |
|---|---|
| 继续只放大评论列表背景 | 不能覆盖 Swift virtual view 的富文本背景和 cell 复用，放弃 |
| 使用固定延时清底 | 首帧仍会闪，且不同机型布局耗时不同，放弃 |
| 在列表布局完成时同步清理，并保留一次主线程兜底 | 能覆盖首帧和后续异步配置，保留 |
| 继续使用 `scaleType=1` | 与真实 40.7.0 二进制行为相反，放弃 |
| 在宿主、detail cell 和播放器三层同时校正画布 | 能覆盖首页、详情页和切换时序，采用 |
| `100%` 仅把 alpha 设为 0 | 仍可能留下透明模糊层和 tint 累加，放弃 |
| `100%` 直接移除 blur 和 tint | 语义明确且能消除黑色叠层，采用 |

## 四、修复方案

### 改动 1: 首页 Feed 宿主高度

`DYYY.xm`:

```objc
%hook AWEFeedTableView
- (void)layoutSubviews {
    %orig;
    if (!DYYYGetBool(@"DYYYEnableFullScreen") || !self.superview) {
        return;
    }
    CGFloat targetHeight = CGRectGetHeight(self.superview.bounds);
    CGRect frame = self.frame;
    if (targetHeight > 0 && fabs(CGRectGetHeight(frame) - targetHeight) > 0.5) {
        frame.size.height = targetHeight;
        self.frame = frame;
    }
}
%end
```

理由: 在列表最终布局阶段覆盖旧版预留高度，避免只修改 cell 后底部仍留空。

### 改动 2: 评论 Swift virtual view 与首帧清理

`DYYY.xm` 在 `AWEListKitMagicCollectionView.layoutSubviews` 完成后同步调用清理；`DYYYUtils.m` 扩展类名匹配并清理 `NSAttributedString` 背景/边框属性。

```objc
BOOL isCommentPanelText = isCommentPanelListClass &&
                          ([className containsString:@"Label"] ||
                           [className containsString:@"Fold"] ||
                           [className containsString:@"VirtualView"] ||
                           [className containsString:@"UserName"] ||
                           [className containsString:@"Time"] ||
                           [className containsString:@"Content"]);
```

```objc
+ (void)clearCommentBlurBackgroundsNowInView:(UIView *)view {
    [self clearCommentBlurBackgroundsInView:view depth:0];
}
```

理由: 同时覆盖普通 view 的 `backgroundColor`、评论富文本背景和 cell 复用，避免“刚打开先白框，再慢慢消失”。

### 改动 3: 百分比透明度和全透明分支

`DYYYUtils.m` 将百分比归一化为 `0.0...1.0`：

```objc
CGFloat normalizedTransparency = MIN(MAX(userTransparency, 0.0), 1.0);
BOOL fullyTransparent = normalizedTransparency >= 0.9995;
UIBlurEffect *blurEffect = fullyTransparent ? nil : [UIBlurEffect effectWithStyle:blurStyle];
if (fullyTransparent) {
    blurView.effect = nil;
    blurView.hidden = YES;
    [blurView removeFromSuperview];
    [self clearCommentBlurBackgroundsInView:panelView depth:0];
    return;
}
```

理由: `100%` 语义为完全透明，不能保留任何模糊或 tint 叠层；低透明度时 blur alpha 为 `1 - transparency`，tint 使用低于线性的衰减避免 80% 时变黑。

`DYYYSettingViewController.m` 和 `DYYYSettings.xm` 使用 0-100 百分比输入；`DYYYUtils.m` 优先读取新键，并在缺失时把旧 0-1 键转换成百分比。

### 改动 4: 主页视频详情全屏 frame

`DYYY.xm` 在 `AWEAwemeDetailTableViewCell` 中关闭底栏覆盖限制，并在 `adjustVideoFrameIfNeededWithPanelType:` 后再次应用全屏 frame；`TTPlayerView` 改为 `scaleType=0` 并调用 `updateVideoFrame`。

```objc
- (BOOL)playerNotCoverBottomBarEnable {
    if (DYYYGetBool(@"DYYYEnableFullScreen") &&
        DYYYGetBool(DYYY_PROFILE_DETAIL_HIDE_BOTTOM_BG_KEY) &&
        (self.homepageUser != nil || [DYYYUtils isProfileVideoDetailContext:self]) &&
        ![DYYYUtils isCommentPanelPresentedInContext:self]) {
        return NO;
    }
    return %orig;
}
```

理由: 40.7.0 的详情页黑条不是单纯背景色问题，而是播放器画布仍使用 aspect-fit 的尺寸。必须在 detail cell frame、播放器容器和 `TTPlayerView` 三层一起校正。

### 不动什么 / 兼容性说明

- 没有删除普通视频播放、下载和设置页逻辑。
- 旧 `DYYYCommentBlurTransparent` 仍会读取并转换为百分比，避免老用户升级后重置配置。
- 所有新增的缩放 hook 都受 `DYYYEnableFullScreen` 开关保护；关闭后恢复抖音原始 `scaleType`。
- 评论清理只在评论面板上下文或已展示评论面板的上下文中执行，不影响普通 UICollectionView。

## 五、验证结果

### 已完成

| 检查项 | 结果 |
|---|---|
| CI 构建 | GitHub Actions run `40013415370` 成功，来源提交 `6835dcc` |
| 版本 | `control` 与 `DYYYConstants.h` 均为 `40.7.0-19` |
| dylib 版本 | 三个 deb 内 dylib 均能匹配到 `40.7.0-19`，每个命中 2 次 |
| Mach-O | 三个 deb 内 `DYYY.dylib` 均为 fat Mach-O，包含 `ARM64 subtype 0` 与 `ARM64 subtype 2147483650`（arm64/arm64e） |
| Rootful 载荷 | `Library/MobileSubstrate/DynamicLibraries/DYYY.dylib` |
| Rootless 载荷 | `var/jb/Library/MobileSubstrate/DynamicLibraries/DYYY.dylib` |
| RootHide 载荷 | `Library/MobileSubstrate/DynamicLibraries/DYYY.dylib` |
| 自签包 | `DYYY-40.7.0-19.selfsign.zip` 根目录只含 `19DYYY.dylib` 和 `DYYY.plist` |
| 自签过滤器 | plist 可解析，Bundles 为 `com.ss.iphone.ugc.Aweme` 和 `com.ss.iphone.ugc.aweme.lite` |
| 调试日志 | 专用调试前缀扫描命中 0 |

### 待真机验证

| 指标 | 修复前 | 本次预期 |
|---|---|---|
| 首页 Feed 底栏间隙 | 视频整体抬升，底部有空隙 | 表格与宿主同高 |
| 评论首帧文字底板 | 昵称、时间、正文先白后消失 | 列表布局完成即清理，不应出现白色底板 |
| 评论区毛玻璃 | 背景丢失或文字变黑 | 评论上面板保留 blur，文字背景清空 |
| 透明度 80% | 直接变黑 | 低强度 blur，不出现黑色叠层 |
| 透明度 100% | 仍不是全透明 | 移除 blur/tint，完全透明 |
| 个人主页视频详情 | 半屏、下方黑条或上一视频残留 | 全宽竖屏画布，无底部黑条 |

当前机器没有连接 iOS 真机，无法在此会话中通过 `Console.app`/设备日志复现并截图确认首帧时序。上述 UI 项应在自签安装后按复现步骤确认；CI 成功和包结构检查不能替代真机结果。

## 六、改动文件清单

| 文件 | 改动 |
|---|---|
| `DYYY.xm` | Feed 表格全高、评论列表同步清理、主页详情画布与 `TTPlayerView scaleType=0` |
| `DYYYUtils.h/.m` | Swift virtual view 识别、同步背景清理、百分比透明度与全透明移除 |
| `DYYYSettingsHelper.m` | 透明度设置变化通知 |
| `DYYYSettingViewController.m` | 百分比输入 UI |
| `DYYYSettings.xm` | 设置页百分比文案与标识 |
| `AwemeHeaders.h` | 40.7.0 播放器/detail cell 接口声明 |
| `DYYYConstants.h` | 版本和透明度配置键 |
| `control` | Debian 包版本 `40.7.0-19` |

## 七、后续建议

1. **真机安装验证** — 自签注入 `DYYY-40.7.0-19.selfsign.zip` 后，按首页、评论区、个人主页视频详情三条路径逐项确认，并记录设备型号/iOS 版本。
2. **首帧回归截图** — 对评论区打开后的第 1 帧、布局完成帧和滚动复用帧各留一张截图，防止异步背景再次回归。
3. **保留旧值兼容测试** — 用仍保存 `DYYYCommentBlurTransparent=0.8` 的旧配置启动一次，确认百分比迁移不会覆盖用户值。
4. **版本命名** — 后续同版本修复继续使用 `40.7.0-20`、`40.7.0-21` 修订号，不再回到 `2.2-*` 主版本。
