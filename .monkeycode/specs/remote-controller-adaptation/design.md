# 遥控器适配技术设计文档

## 1. 概述

本方案为 PiliPlus Flutter 应用实现完整的 Android 遥控器适配，涵盖方向键导航、确认键操作、返回键处理，以及新增的"自动播放上次视频"功能。方案基于 Flutter 内置的 Focus 系统扩展现有键盘事件处理能力。

## 2. 架构设计

### 2.1 现有代码分析

- **PlayerFocus** (`lib/pages/video/widgets/player_focus.dart`): 已实现视频播放页的部分键盘事件处理（方向键调音量/快进快退、空格暂停、F 全屏等），但仅聚焦播放器区域，不处理页面级焦点导航
- **BackDetector** (`lib/common/widgets/back_detector.dart`): 已处理 Escape 键返回，可扩展处理遥控器返回键
- **存储系统**: 使用 Hive CE，`GStorage.video` 和 `GStorage.setting` 两个 Box 分别存储视频设置和通用设置
- **自动播放**: 已有 `SettingBoxKey.autoPlayEnable` 用于多P视频自动连播，需新增独立存储键

### 2.2 技术方案

```
┌─────────────────────────────────────────────────────────────┐
│                     App 启动流程                              │
├─────────────────────────────────────────────────────────────┤
│  1. 检查 autoResumeLastVideo 开关                             │
│  2. 若开启 + 有登录态 + 有历史记录 → 跳转视频播放页            │
│  3. 否则 → 正常进入首页                                       │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                   遥控器焦点导航架构                           │
├─────────────────────────────────────────────────────────────┤
│  MaterialApp                                                │
│  └── Shortcuts (全局键盘映射)                                 │
│      └── Actions (全局动作处理)                               │
│          └── FocusScope (焦点树根)                            │
│              ├── HomePage (Focus + 视频网格)                  │
│              ├── VideoDetailPage (PlayerFocus + 简介区)       │
│              ├── HistoryPage (列表 Focus)                     │
│              └── SettingPage (列表 Focus)                     │
└─────────────────────────────────────────────────────────────┘
```

## 3. 详细设计

### 3.1 新增存储键

**文件**: `lib/utils/storage_key.dart`

在 `SettingBoxKey` 类中新增：

```dart
static const String autoResumeLastVideo = 'autoResumeLastVideo';
```

### 3.2 Pref 访问器

**文件**: `lib/utils/storage_pref.dart`

```dart
static bool get autoResumeLastVideo =>
    _setting.get(SettingBoxKey.autoResumeLastVideo, defaultValue: false);
```

### 3.3 设置页面新增选项

**文件**: `lib/pages/setting/models/video_settings.dart`

在视频设置列表中新增开关项：

```dart
const SwitchModel(
  title: '自动播放上次视频',
  subtitle: '打开应用时自动恢复上次观看的视频',
  leading: Icon(Icons.autorenew_outlined),
  setKey: SettingBoxKey.autoResumeLastVideo,
  defaultVal: false,
),
```

### 3.4 App 启动时自动恢复播放

**文件**: `lib/main.dart`

在 App 初始化完成后、首页加载前，检查是否需要恢复上次播放：

```dart
// 在 main() 的 runApp 前或首页 controller 的 onInit 中
Future<void> _tryResumeLastVideo() async {
  if (!Pref.autoResumeLastVideo) return;
  if (!Accounts.hasLogin) return;

  final history = await UserHttp.historyList(
    account: Accounts.current,
    type: null,
    max: null,
    viewAt: null,
  );

  if (history case Success(:final response)) {
    final lastItem = response.list?.firstOrNull;
    if (lastItem != null) {
      final historyData = lastItem.history;
      // 跳转到视频播放页，携带播放进度
      Get.offNamed(
        Routes.VIDEO_DETAIL,
        arguments: {
          'bvid': historyData.bvid,
          'cid': historyData.cid,
          'resumePosition': historyData.progress, // 恢复进度（秒）
        },
      );
    }
  }
}
```

### 3.5 视频播放页支持恢复进度

**文件**: `lib/pages/video/controller.dart`

在 `VideoDetailController` 中处理 `resumePosition` 参数：

```dart
// 在 initVideo() 或 initPlayer() 中
if (args['resumePosition'] case int position when position > 0) {
  // 播放器初始化完成后 seek 到指定位置
  plPlayerController.player?.seek(Duration(seconds: position));
}
```

### 3.6 全局遥控器焦点导航

#### 3.6.1 方案选型

采用 Flutter 的 **Focus + FocusTraversalPolicy** 方案，而非手动监听 RawKeyboard。理由：
- Flutter Focus 系统原生支持方向键导航
- 自动处理焦点树遍历
- 与现有 `PlayerFocus` 组件兼容

#### 3.6.2 全局配置

**文件**: `lib/main.dart`

在 `MaterialApp` 层配置全局 Shortcuts 和 Actions：

```dart
MaterialApp(
  shortcuts: {
    ...DefaultShortcuts().shortcuts,
    // 遥控器返回键映射
    LogicalKeySet(LogicalKeyboardKey.goBack): const ActivateIntent(),
  },
  builder: (context, child) {
    return FocusTraversalGroup(
      policy: ReadingOrderTraversalPolicy(),
      child: child!,
    );
  },
  // ...
)
```

### 3.7 首页遥控器适配

**文件**: `lib/pages/home/view.dart`

在首页视频网格列表中，为每个视频卡片包裹 `Focus` 和 `FocusTraversalOrder`：

```dart
Focus(
  onFocusChange: (hasFocus) {
    if (hasFocus) {
      // 焦点时显示高亮边框
    }
  },
  child: GestureDetector(
    onTap: () => _openVideo(video),
    child: KeyboardListener(
      focusNode: FocusNode(),
      onKeyEvent: (event) {
        if (event is KeyDownEvent && event.logicalKey == LogicalKeyboardKey.enter) {
          _openVideo(video);
        }
      },
      child: VideoCard(video: video),
    ),
  ),
)
```

更推荐的方式是使用 `InkWell` + `focusNode` + `canRequestFocus`：

```dart
InkWell(
  focusNode: FocusNode(),
  canRequestFocus: true,
  onFocusChange: (hasFocus) => _updateFocusStyle(hasFocus),
  onTap: () => _openVideo(video),
  child: VideoCard(video: video),
)
```

### 3.8 视频播放页遥控器增强

**文件**: `lib/pages/video/widgets/player_focus.dart`

在现有 `_handleKey` 方法中扩展遥控器按键处理：

```dart
bool _handleKey(KeyEvent event) {
  // ... 现有逻辑 ...

  // 遥控器确认键
  if (key == LogicalKeyboardKey.enter || key == LogicalKeyboardKey.select) {
    if (event is KeyDownEvent) {
      if (isFullScreen) {
        // 全屏时：显示/隐藏控制器
        plPlayerController.toggleControls();
      } else {
        // 非全屏时：如果焦点不在播放器，不处理
      }
    }
    return true;
  }

  // 遥控器返回键 (映射为 goBack 或 escape)
  if (key == LogicalKeyboardKey.goBack || key == LogicalKeyboardKey.escape) {
    if (event is KeyDownEvent) {
      if (isFullScreen) {
        plPlayerController.triggerFullScreen(status: false);
      } else {
        Get.back();
      }
    }
    return true;
  }

  // ... 现有逻辑 ...
}
```

### 3.9 设置页遥控器适配

**文件**: `lib/pages/setting/video_setting.dart`

ListView 中的每个设置项使用 `ListTile` + `focusNode`：

```dart
ListTile(
  focusNode: FocusNode(),
  onTap: () => _handleSettingTap(item),
  trailing: item.trailing,
  // ...
)
```

对于 `SwitchModel`，确保开关本身可通过方向键聚焦并切换：

```dart
Switch(
  value: value,
  onChanged: (v) => onChanged(v),
  focusNode: FocusNode(),
)
```

### 3.10 历史记录页遥控器适配

**文件**: `lib/pages/history/view.dart`

历史列表中的每个视频卡片添加 `canRequestFocus`：

```dart
InkWell(
  canRequestFocus: true,
  focusNode: FocusNode(),
  onTap: () => _openVideo(item),
  child: HistoryCard(item: item),
)
```

### 3.11 焦点视觉反馈

为所有可聚焦控件定义统一的焦点样式：

**文件**: `lib/common/widgets/focus_border.dart` (新建)

```dart
class FocusBorder extends StatelessWidget {
  const FocusBorder({
    super.key,
    required this.child,
    this.focusNode,
    this.borderWidth = 3.0,
    this.borderColor,
    this.borderRadius = 8.0,
  });

  @override
  Widget build(BuildContext context) {
    return Focus(
      focusNode: focusNode,
      onFocusChange: (hasFocus) {
        // 触发动画
      },
      child: AnimatedContainer(
        duration: const Duration(milliseconds: 150),
        decoration: BoxDecoration(
          border: Border.all(
            color: hasFocus ? (borderColor ?? Theme.of(context).colorScheme.primary) : Colors.transparent,
            width: hasFocus ? borderWidth : 0,
          ),
          borderRadius: BorderRadius.circular(borderRadius),
        ),
        child: child,
      ),
    );
  }
}
```

## 4. 改动文件清单

| 文件 | 改动类型 | 说明 |
|------|---------|------|
| `lib/utils/storage_key.dart` | 新增常量 | 添加 `autoResumeLastVideo` 存储键 |
| `lib/utils/storage_pref.dart` | 新增 getter | 添加 `autoResumeLastVideo` 访问器 |
| `lib/pages/setting/models/video_settings.dart` | 新增设置项 | 添加"自动播放上次视频"开关 |
| `lib/main.dart` | 修改 | App 启动时检查并恢复上次播放；配置全局 Shortcuts |
| `lib/pages/video/controller.dart` | 修改 | 支持 `resumePosition` 参数恢复播放进度 |
| `lib/pages/video/widgets/player_focus.dart` | 扩展 | 增加遥控器确认键、返回键处理 |
| `lib/pages/home/view.dart` | 修改 | 视频卡片支持 Focus 导航 |
| `lib/pages/history/view.dart` | 修改 | 历史记录卡片支持 Focus 导航 |
| `lib/pages/setting/video_setting.dart` | 修改 | 设置项支持 Focus 导航 |
| `lib/common/widgets/focus_border.dart` | 新建 | 通用焦点视觉反馈组件 |
| `lib/common/widgets/back_detector.dart` | 扩展 | 增加 `LogicalKeyboardKey.goBack` 处理 |

## 5. 关键技术点

### 5.1 播放器恢复进度

media_kit 播放器支持 `seek()` 方法。需要在播放器完全初始化后再执行 seek，否则可能无效。建议在 `PlPlayerController` 的播放器状态变为 `playing` 或 `ready` 后执行。

### 5.2 历史记录获取

使用已有的 `UserHttp.historyList` API 获取观看历史。第一条记录即为最近观看的视频。历史记录模型包含 `bvid`、`cid`、`progress`（播放进度秒数）。

### 5.3 遥控器按键映射

Android TV 遥控器标准按键：
- D-Pad Up/Down/Left/Right → `LogicalKeyboardKey.arrowUp/Down/Left/Right`
- D-Pad Center (OK) → `LogicalKeyboardKey.enter` 或 `LogicalKeyboardKey.select`
- Back → `LogicalKeyboardKey.goBack` (Android 系统层已处理，但需在全屏播放时拦截)
- Menu → `LogicalKeyboardKey.contextMenu`

### 5.4 焦点遍历策略

使用 `ReadingOrderTraversalPolicy` 按阅读顺序（从左到右、从上到下）遍历焦点。对于网格布局（如视频列表），可自定义 `FocusOrder` 或使用 `WidgetOrderTraversalPolicy`。

## 6. 风险与应对

| 风险 | 影响 | 应对策略 |
|------|------|---------|
| 播放器 seek 时机不当导致失败 | 无法恢复播放进度 | 监听播放器状态，在 `ready` 事件后 seek |
| 焦点导航在复杂布局中错乱 | 用户体验差 | 使用 `FocusTraversalGroup` 分组管理焦点树 |
| 无登录态或无历史记录时崩溃 | App 启动异常 | 增加空值检查和异常处理 |
| 部分遥控器按键码不标准 | 部分按键不响应 | 增加按键日志调试能力，后续迭代补充 |

## 7. 测试策略

### 7.1 手动测试

- 使用 Android TV 模拟器或真机连接遥控器
- 验证各页面方向键导航路径正确
- 验证确认键触发预期操作
- 验证返回键行为符合预期
- 验证自动播放上次视频功能（开启/关闭两种状态）

### 7.2 自动化测试

- 单元测试：`storage_pref.dart` 新增 getter 的默认值测试
- Widget 测试：`FocusBorder` 组件的焦点样式变化测试

## 8. 实施计划

1. **阶段一**: 新增存储键、Pref 访问器、设置页选项（预计 0.5 天）
2. **阶段二**: 实现自动播放上次视频功能（启动恢复逻辑 + 播放器 seek）（预计 1 天）
3. **阶段三**: 首页、历史页、设置页的 Focus 导航适配（预计 1.5 天）
4. **阶段四**: 视频播放页遥控器增强 + 焦点视觉反馈组件（预计 1 天）
5. **阶段五**: 联调测试、Bug 修复（预计 1 天）

总计约 5 天。
