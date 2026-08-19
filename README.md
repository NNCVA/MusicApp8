# Local Music Player

按 `本地音乐播放器_PRD.md v2.0.0` 实现的本地优先 Android 音乐播放器。

## 构建基线

本工程按当前指定版本固定：

- Gradle `9.5.0`
- Gradle 运行 JDK：`21`
- Android Gradle Plugin `9.3.0`
- Kotlin / Compose Compiler plugin `2.3.20`
- `compileSdk = 36` / `targetSdk = 36` / `minSdk = 26`
- Android 源码字节码目标保持 Java/Kotlin 17（JDK 21 用于运行 Gradle，并不要求把设备侧字节码目标提升到 21）
- Compose BOM `2026.03.01`
- Navigation 3 `1.0.1`
- Room `2.8.4`
- DataStore `1.2.1`
- Media3 `1.10.1`
- Lifecycle `2.10.0`
- Coroutines `1.11.0`
- Coil 3 `3.5.0`
- ICU4J `77.1`
- jaudiotagger `3.0.1`

AGP 9 使用 Built-in Kotlin，工程**没有**再应用 `org.jetbrains.kotlin.android`。Room 使用 KSP。

应用没有声明 `INTERNET` 或 `POST_NOTIFICATIONS` 权限，核心播放/扫描业务不依赖网络。

## 页面覆盖

PRD 中 13 个路由页面均已有实现与导航入口：

1. 单曲列表
2. 专辑网格
3. 专辑详情
4. 艺术家列表
5. 艺术家详情
6. 文件夹浏览（多级下钻）
7. 歌单主页
8. 歌单详情
9. 播放历史
10. 音乐扫描
11. 设置中心
12. 扫描规则
13. 关于

全局宿主同时包含 Mini Player / Full Player、歌词、播放队列、扫描增量浮窗、顶部消息胶囊、歌曲信息弹层与加入歌单弹层。

13 个路由页面现在分别位于 `ui/screens/<PageName>.kt`，不再集中在单一 `Screens.kt`；纯 Compose 工程不使用 `res/layout/*.xml`。

## 核心实现

- MediaStore 扫描 MP3 / FLAC / WAV / AAC / M4A / OGG / OPUS
- 全盘 / 指定目录扫描；黑名单优先；100KB 短音频过滤；可取消；Flow 增量事件
- 扫描范围增量对账：指定目录扫描不会误删未扫描目录缓存
- 用户隐藏状态跨重新扫描保留
- 扫描阶段提取 120×120 WebP 封面缓存
- 单曲 / 专辑 / 艺术家 / 多级文件夹分类
- ICU4J 中文转写索引：歌名 / 艺术家 / 专辑 / 拼音全拼 / 紧凑拼音 / 首字母搜索
- A-Z 索引、触觉反馈、排序与搜索
- 长按多选：下一首 / 加入队列 / 加入歌单 / 隐藏
- 歌单 CRUD、播放全部、随机播放、批量移除
- 歌单网格模式前 4 首四宫格封面；列表模式最近加入歌曲封面
- 历史记录阈值 `min(30s, 曲目时长 50%)`
- Compact `<600dp` 50% 推移侧栏、Medium `240dp`、Expanded `256dp`
- Mini Player 固定高 80dp 且位于侧栏上层
- Full Player：唱片 / 歌词 / 队列三页 Pager
- Player Sheet 跟手拖动；双向 35% 位置阈值；1000dp/s 速度阈值；Spring 吸附
- 歌词/队列列表滚动到顶部后，剩余下拉手势交给 Player Sheet 收起
- Mini / Full 交叉淡入淡出，不做封面连续缩放形变
- 单 ExoPlayer：淡出 → 切换 → 淡入，不重叠播放两首曲目
- LIST_LOOP / SINGLE_LOOP / SHUFFLE 三模式
- 原始队列 + 稳定随机序列双队列；换轮边界防重复
- 普通加入队列、下一首播放、当前项移除、队列点击跳转
- 队列 / 模式 / 当前曲目 / 进度 Room 持久化与冷启动恢复
- MediaSessionService、Audio Focus、锁屏/蓝牙媒体控制
- BecomingNoisy 监听受 DataStore 开关控制
- 播放异常自动跳过；连续 3 首失败熔断暂停
- 外部 LRC 字符集降级（UTF-8 / GB18030 / Big5 / GB2312）
- ID3v2 `SYLT` 帧级时间轴解析 + `USLT` 纯文本降级
- 歌词源优先级：带时间轴 LRC → SYLT → 其他时间轴嵌入歌词；无时间轴时外部 LRC → USLT / 嵌入文本
- 歌词 ±0.5s 偏移并持久化
- Material You + 4 套预设色 + Light/Dark
- Android 13+ AGSL RuntimeShader；旧系统 / 低电量静态降级；仅 RESUMED 驱动帧循环
- 睡眠定时器：15 分 / 30 分 / 播完当前
- Compact 歌曲信息 Bottom Sheet；宽屏最大 640dp Dialog；按需读取编码/MIME、比特率、采样率、位深、文件大小与路径；仅路径提供复制
- 中文 / English 两套资源；系统 App Language 切换
- 首次无缓存全屏雷达扫描；已有缓存启动时静默后台同步
- 手动扫描雷达、扫描模式切换、全局可折叠增量浮窗与最近新增曲目
- 专辑/艺术家/文件夹搜索；文件夹可点击面包屑与“排除此目录”
- 专辑详情 Compact / Expanded 自适应布局，真实缓存封面优先

## Android 平台约束

PRD 中“**播放期间媒体通知始终可划除，划除即停止服务**”与 Android 前台 `mediaPlayback` 服务约束存在冲突：当 `MediaSessionService` 正在以前台服务播放时，系统要求前台通知存在，Media3 官方也明确说明前台服务运行期间该通知不能被移除。

因此本工程采用平台合法行为：

- 活跃播放：保留系统媒体前台通知；
- 暂停/空队列：服务可退出前台并允许系统移除；
- 从最近任务移除 App 时，若播放器已经暂停或为空则停止空闲服务；活跃播放继续遵循后台媒体播放语义。

这不是 UI 层能够可靠绕过的限制，不能通过伪造“可划除通知”破坏 Android 前台服务规则。

## 已执行的源码级验证

当前隔离执行环境没有 Android SDK / Gradle 安装，因此不能在这里冒充 `assembleDebug` 已通过。已经实际执行：

- XML：全部资源文件格式校验通过
- i18n：`values` / `values-en` 各 225 个 string key，完全一致
- `R.string`：主源码全部引用均存在
- 主 Kotlin 源码：无中文 UI 硬编码
- `QueueManager` 纯 Kotlin smoke test：随机换轮边界、下一首插入、原始队列稳定、快照恢复通过
- ID3v2 lyrics 纯 Kotlin smoke test：SYLT 时间点 + USLT 文本解析通过
- Manifest：无 `INTERNET` / `POST_NOTIFICATIONS`

真实 APK、Lint、设备兼容性和性能红线仍必须在具备 Android SDK 36 的构建机执行。当前容器的真实构建尝试在下载 Gradle 9.5.0 阶段即因 DNS 无法解析官方地址和备用镜像而失败，详见 `docs/BUILD_STATUS.md` / `docs/BUILD_ATTEMPT_CURRENT.log`。

## 构建 APK

要求：

- JDK 21
- Android SDK Platform 36
- Android SDK Build Tools 36.0.0

如果本机已有 Gradle 9.5.0：

```bash
gradle :app:assembleDebug
```

没有 Gradle 时可使用项目附带的 bootstrap：

```bash
./gradlew-bootstrap :app:assembleDebug
```

成功后 APK 位于：

```text
app/build/outputs/apk/debug/app-debug.apk
```

建议直接运行项目附带的一键脚本：

```bash
./build-apk.sh
```

它会依次执行单元测试、`assembleDebug` 和 Lint，并把成功产物复制到 `dist/LocalMusicPlayer-debug.apk`。

## 目录

```text
app/src/main/java/com/vanc/localmusic/
├── data/       Room、DataStore、MediaStore Scanner、Repository
├── lyrics/     LRC、ID3 SYLT/USLT、LyricsEngine
├── playback/   QueueManager、Crossfade、Media3 Service/Controller
├── ui/         Navigation 3、App Shell、主题、页面、全局播放器
├── util/       ICU4J 搜索/排序索引
└── viewmodel/  主状态机与业务编排
```
