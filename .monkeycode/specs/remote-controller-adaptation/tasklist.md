# 遥控器适配实施任务清单

## 阶段一：存储与设置（0.5 天）

- [ ] 1.1 `lib/utils/storage_key.dart` 新增 `autoResumeLastVideo` 常量
- [ ] 1.2 `lib/utils/storage_pref.dart` 新增 `autoResumeLastVideo` getter
- [ ] 1.3 `lib/pages/setting/models/video_settings.dart` 新增"自动播放上次视频"开关项
- [ ] 1.4 验证设置项在 UI 中正常显示、开关状态正确持久化

## 阶段二：自动播放上次视频（1 天）

- [ ] 2.1 `lib/main.dart` 或首页 Controller 中实现启动检查逻辑
  - 检查 `autoResumeLastVideo` 开关
  - 检查登录态
  - 获取最近一条历史记录
- [ ] 2.2 实现跳转视频播放页逻辑，携带 `bvid`、`cid`、`resumePosition` 参数
- [ ] 2.3 `lib/pages/video/controller.dart` 处理 `resumePosition` 参数
- [ ] 2.4 在播放器就绪后 seek 到指定进度
- [ ] 2.5 边界情况处理：
  - 无登录态时正常进入首页
  - 无历史记录时正常进入首页
  - 历史记录视频已失效时的异常处理

## 阶段三：页面 Focus 导航适配（1.5 天）

### 3.1 首页
- [ ] 3.1.1 视频卡片包裹 `InkWell` + `canRequestFocus: true`
- [ ] 3.1.2 分类 Tab 支持方向键导航
- [ ] 3.1.3 焦点视觉反馈（高亮边框/背景色）

### 3.2 历史记录页
- [ ] 3.2.1 历史视频卡片支持 Focus
- [ ] 3.2.2 确认键打开视频

### 3.3 设置页
- [ ] 3.3.1 设置项 ListTile 支持 Focus
- [ ] 3.3.2 开关控件可聚焦并切换
- [ ] 3.3.3 弹窗选项对话框支持方向键选择

### 3.4 全局
- [ ] 3.4.1 `lib/main.dart` 配置全局 `Shortcuts` 和 `FocusTraversalGroup`
- [ ] 3.4.2 新建 `lib/common/widgets/focus_border.dart` 通用焦点组件

## 阶段四：视频播放页遥控器增强（1 天）

- [ ] 4.1 `lib/pages/video/widgets/player_focus.dart` 扩展：
  - 遥控器确认键（`LogicalKeyboardKey.enter` / `select`）
  - 遥控器返回键（`LogicalKeyboardKey.goBack`）
- [ ] 4.2 `lib/common/widgets/back_detector.dart` 增加 `goBack` 键处理
- [ ] 4.3 全屏模式下控制器显示/隐藏
- [ ] 4.4 非全屏模式下简介区、推荐列表的焦点导航

## 阶段五：联调测试（1 天）

- [ ] 5.1 Android TV 模拟器端到端测试
- [ ] 5.2 修复焦点导航路径问题
- [ ] 5.3 修复自动播放边界情况
- [ ] 5.4 性能和响应时间验证
