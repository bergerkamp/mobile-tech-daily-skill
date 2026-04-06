# Mobile Tech Daily — 知识主题库

## Android

### 启动优化
- Application.onCreate() 异步初始化：用 `InitializerProvider` / App Startup 替代同步阻塞，减少 cold start 时间
- Baseline Profile：AOT 预编译热路径，首帧渲染可提升 30%-40%，Compose 项目尤其有效
- ContentProvider 数量优化：每个 ContentProvider 在进程启动时同步 onCreate，合并或懒加载可缩短 TTI
- `StartupTracing`（Jetpack Macrobenchmark）精确测量 cold/warm/hot start

### 内存
- `SparseArray` vs `HashMap`：key 为 int 时，SparseArray 节省装箱开销，内存减少约 30%
- Bitmap 内存：`BitmapFactory.Options.inSampleSize` + `inBitmap` 复用，避免 OOM 和 GC 抖动
- LeakCanary 原理：弱引用 + ReferenceQueue 检测泄漏，核心在 `ObjectWatcher`
- 大对象（>= 85KB）直接进入 Large Object Heap，GC 不压缩，长期持有会碎片化

### 网络
- OkHttp 连接池：默认 5 连接 / 5 分钟空闲，`ConnectionPool(10, 5, TimeUnit.MINUTES)` 可调
- HTTP/3 (QUIC)：Cronet 支持 IETF QUIC，0-RTT 恢复连接；弱网下比 TCP 快 20%-40%
- DNS 预解析：`OkHttpClient.dns()` 自定义，结合 HTTPDNS 规避运营商劫持
- Certificate Transparency：Android 7+ 系统 Trust Store 默认开启，自签证书需显式信任

### 渲染
- Compose Recomposition：`remember { }` 缓存状态，避免 lambda 捕获导致无效重组
- View 过度绘制：GPU Overdraw 工具定位，clipRect() 减少无效绘制区域
- RecyclerView `setHasFixedSize(true)`：跳过 measure/layout 重计算，列表滚动更流畅
- 主线程 16ms 预算：`Choreographer.FrameCallback` 监控帧耗时，超过即掉帧

### 包体积
- R8 全模式 vs ProGuard：R8 fullMode 额外做内联、去无用参数，包体积可再减 5%-15%
- ABI 拆分：仅打包 `arm64-v8a`，去掉 `x86/x86_64`，节省约 30% so 大小
- `shrinkResources true`：配合 `minifyEnabled true` 删除未引用资源，注意动态加载的资源要保留
- WebP 替换 PNG：无损 WebP 体积减少约 25%，有损可达 60%；API 14+ 全面支持

### 编译 & 构建
- Kotlin Incremental 编译：`kapt` 改用 `ksp`，全量编译速度提升 2-4x
- Gradle Configuration Cache：缓存配置阶段结果，大型项目二次构建减少 60% 耗时
- `--rerun-tasks` vs `--no-build-cache`：前者强制重跑已缓存任务，调试构建脚本时用
- Dex 文件数量：超过 65536 方法触发 MultiDex，`d8` 比 `dx` 编译更快且体积更小

### 安全
- 不要把 API Key 硬编码在代码里：反编译即可提取，应走服务端代理
- Root 检测：`/system/xbin/su` 路径检测 + `Build.TAGS` 包含 "test-keys"，可被 Magisk 绕过，建议结合 SafetyNet / Play Integrity
- Android Keystore：私钥存在 TEE（硬件安全区），永不离开设备，即使 root 也无法导出
- `FLAG_SECURE`：防止 Activity 被截图录屏，金融/支付页面必加

### 调试
- `adb shell am start -W`：测量 cold start TotalTime，CI 监控启动回归
- Perfetto：比 Systrace 更强大，支持 SQL 查询 trace，可分析跨进程调用链
- `strict mode`：`StrictMode.setThreadPolicy` 检测主线程 IO，开发阶段必开
- `dumpsys meminfo <package>`：查看 PSS/RSS/USS，定位内存问题比 Profiler 更快

---

## iOS

### 启动优化
- `+load` vs `+initialize`：`+load` 在 dylib 加载时同步执行，启动期间堆积多个 `+load` 严重拖慢冷启；`+initialize` 懒加载，优先使用
- 动态库数量：每个 dylib 增加约 1-2ms 启动耗时，超过 6 个开始显著影响；合并或改用静态库
- Prewarm（iOS 15+）：系统在后台预热 App，`DYLD_COLUMN` 中 `procStartAbsTime` 前有预热时间，测启动时需排除
- MetricKit：线上冷/热启动时间采集，比 Instruments 更接近用户真实体验

### 内存
- AutoreleasePool 时机：默认 RunLoop 迭代末尾排干，循环内大量创建临时对象要手动包 `@autoreleasepool {}`
- `NSCache` vs `NSDictionary`：NSCache 支持内存压力自动淘汰，不需要手动管理内存警告
- Instruments Allocations：Persistent Bytes 持续增长 = 内存泄漏；Generation Analysis 对比两个时间点的存活对象
- Tagged Pointer：小整数、短字符串直接编码在指针里，无堆分配，ARM64 高位存 tag

### 网络
- URLSession `waitsForConnectivity = true`：网络不可用时等待而非立即失败，适合非实时请求
- HTTP/2 多路复用：URLSession 默认对同 Host 复用连接；手动设置 `httpMaximumConnectionsPerHost` 避免竞争
- Background URLSession：App 进入后台后系统接管下载，进度通过 delegate 回调恢复
- Certificate Pinning：`URLSession(_:didReceive:completionHandler:)` 中验证公钥 hash，防中间人

### 渲染
- `drawRect:` 触发 offscreen rendering：Core Graphics 绘制在 CPU，避免在高频 cell 中使用
- `CALayer.shouldRasterize`：将图层光栅化缓存为 bitmap，适合静态内容；动态内容反而更慢
- 圆角性能：`layer.cornerRadius` + `masksToBounds = true` 触发离屏渲染；用预裁剪图片或 `UIBezierPath` 替代
- `preferredFramesPerSecond`（iOS 15+ ProMotion）：自适应 120Hz，使用 CADisplayLink 时需设置 `preferredFrameRateRange`

### 包体积
- App Thinning：Bitcode + Slicing + On-Demand Resources，用户下载包远小于上传包
- Swift 代码大小：避免 `@_specialize` 过度特化，每个特化版本都会增加二进制体积
- 图片资源：使用 Asset Catalog，自动按设备分发 @1x/@2x/@3x；HEIF 格式比 PNG 小约 50%
- Instruments Link Map：分析各模块体积占比，定位最大贡献者

### 安全
- Keychain：比 UserDefaults 安全，支持 `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`（越狱设备无法迁移）
- App Transport Security：强制 HTTPS，`NSExceptionDomains` 白名单要写明原因，审核时 Apple 会检查
- Jailbreak 检测：`/Applications/Cydia.app` 路径检测 + `canOpenURL: cydia://` + `fork()` 返回值，越狱设备均可触发
- `Entitlements`：过度申请权限（如 push 但从不使用）会被 App Store 审核拒绝

---

## HarmonyOS

### ArkTS & 渲染
- `@State` 仅触发当前组件重新渲染；`@Observed` + `@ObjectLink` 跨组件同步嵌套对象状态
- `LazyForEach` vs `ForEach`：列表数据量 > 50 时用 LazyForEach，按需加载 item，减少首屏构建时间
- `@Reusable`：标记可复用组件，类似 Android RecyclerView ViewHolder，避免重复创建节点
- Column/Row 过度嵌套：ArkUI 布局树超过 5 层时性能下降明显，用 RelativeContainer 减少层级

### 网络
- `@ohos.net.http`：系统 HTTP 客户端，API 9+ 支持 HTTP/2；API 26+ 支持 QUIC（需华为系统层开启）
- `PowerStart` 机制：华为 QUIC 在网络切换时比 TCP 快 200-400ms 完成连接恢复（0-RTT）
- `PowerCC` 拥塞控制：结合硬件信号强度动态调整发送窗口，弱网下比 BBR 表现更稳定
- 长连接管理：HarmonyOS 系统级长连接（HMS Push 通道）在 App 后台时由系统代持，节省电量

### 性能
- `TaskPool` vs `Worker`：TaskPool 自动管理线程生命周期，适合短任务；Worker 适合长驻后台计算
- `Sendable` 对象：跨线程零拷贝传递，避免序列化开销，比 Worker 消息传递快约 3x
- `hiTraceMeter`：精确打点测量关键路径耗时，替代手写 Date.now() 差值
- Native 侧 NAPI：频繁调用 TS↔C++ 互操作时，批量化接口调用减少 JNI 桥接开销

### 安全
- `HUKS`（华为通用密钥库）：私钥存 TEE，接口与 Android Keystore 类似；API 9+ 可用
- 权限动态申请：`abilityAccessCtrl.requestPermissionsFromUser`，首次拒绝后需引导用户去设置页
- 数据沙箱：App 数据默认在 `Context.filesDir`，跨 App 访问需声明 `dataShareExtension`

---

## 跨平台

### React Native
- New Architecture（JSI + Fabric + TurboModules）：JS 直接持有 C++ 对象引用，跨桥调用从异步变同步，性能提升 20%-40%
- Hermes：默认启用的 JS 引擎，比 JavaScriptCore 冷启快 30%，内存低 10%；iOS RN 0.70+ 默认开启
- `useNativeDriver: true`：动画计算下沉到 Native 线程，避免 JS 线程阻塞导致卡帧

### Flutter
- `const` 构造器：compile-time 常量 widget，重建时跳过 diff，性能优化最低成本手段
- `RepaintBoundary`：隔离重绘区域，列表 item 各自独立，避免滚动时全量重绘
- Platform Channel 批量化：单次调用携带多个数据比多次单独调用快，减少序列化次数

### 通用
- 网络请求去重：同一接口并发调用时合并为一个请求（request coalescing），减少服务端压力和客户端等待
- 离线优先设计：先读缓存呈现 UI，后台刷新后局部更新，用户感知延迟 < 100ms
- Feature Flag 最佳实践：开关粒度要细（函数级），关闭路径代码覆盖测试，定期清理超过 2 个版本的废弃开关
