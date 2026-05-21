---
layout: private_post
title: "Android 应用异常日志采集与上报体系"
---

> 本文梳理 Android 应用开发中异常日志的采集、持久化、上报、服务端处理及触达开发者的完整链路。

---

## 一、异常采集 — 在用户设备上捕获异常

### 1.1 全局未捕获异常（Java/Kotlin 崩溃）

通过 `Thread.UncaughtExceptionHandler` 捕获所有未处理的崩溃：

```java
public class CrashHandler implements Thread.UncaughtExceptionHandler {

    private Thread.UncaughtExceptionHandler mDefaultHandler;

    public void init() {
        mDefaultHandler = Thread.getDefaultUncaughtExceptionHandler();
        Thread.setDefaultUncaughtExceptionHandler(this);
    }

    @Override
    public void uncaughtException(Thread t, Throwable e) {
        // 1. 收集设备信息 + 堆栈
        String log = collectCrashInfo(e);
        // 2. 写入本地文件
        saveCrashToFile(log);
        // 3. 交给默认处理器（弹出"应用停止运行"）
        if (mDefaultHandler != null) {
            mDefaultHandler.uncaughtException(t, e);
        }
    }

    private String collectCrashInfo(Throwable e) {
        StringBuilder sb = new StringBuilder();
        sb.append("Brand: ").append(Build.BRAND).append("\n");
        sb.append("Model: ").append(Build.MODEL).append("\n");
        sb.append("SDK: ").append(Build.VERSION.SDK_INT).append("\n");
        StringWriter sw = new StringWriter();
        e.printStackTrace(new PrintWriter(sw));
        sb.append(sw.toString());
        return sb.toString();
    }
}
```

在 `Application.onCreate()` 中初始化：

```java
public class MyApp extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
        new CrashHandler().init();
    }
}
```

### 1.2 ANR 监控

通过 Watchdog 方式检测主线程是否卡死：

```java
private final Handler mMainHandler = new Handler(Looper.getMainLooper());
private final Handler mWatchHandler = new Handler(handlerThread.getLooper());

public void startWatch() {
    final long CHECK_INTERVAL = 5000;
    mWatchHandler.postDelayed(new Runnable() {
        @Override
        public void run() {
            final boolean[] responded = {false};
            mMainHandler.post(() -> responded[0] = true);
            mWatchHandler.postDelayed(() -> {
                if (!responded[0]) {
                    String stack = getMainThreadStack();
                    saveAnrLog(stack);
                }
            }, CHECK_INTERVAL);
            mWatchHandler.postDelayed(this, CHECK_INTERVAL);
        }
    }, CHECK_INTERVAL);
}

private String getMainThreadStack() {
    StackTraceElement[] stack = Looper.getMainLooper().getThread().getStackTrace();
    StringBuilder sb = new StringBuilder("ANR detected:\n");
    for (StackTraceElement e : stack) {
        sb.append("  at ").append(e).append("\n");
    }
    return sb.toString();
}
```

原理：向主线程投递消息，在子线程定时检查是否响应，超时则 dump 主线程堆栈。

### 1.3 Native Crash 采集

Java 层无法捕获 C/C++ 崩溃（SIGSEGV、SIGABRT 等），需要注册信号处理器：

```c
#include <signal.h>

void signalHandler(int sig, siginfo_t *info, void *context) {
    // 写入崩溃信息到文件
    // 调用原有 handler
}

void registerNativeCrashHandler() {
    struct sigaction sa;
    sa.sa_sigaction = signalHandler;
    sa.sa_flags = SA_SIGINFO;
    sigaction(SIGSEGV, &sa, NULL);
    sigaction(SIGABRT, &sa, NULL);
}
```

实际项目中推荐使用成熟方案（如 xCrash），自己实现成本较高。

### 1.4 业务层主动捕获

对关键业务逻辑主动 try-catch 并记录：

```java
try {
    riskyOperation();
} catch (Exception e) {
    Log.e(TAG, "操作失败", e);
    LogCollector.report(e);
}
```

---

## 二、Android 日志系统基础

### 2.1 系统原生日志架构

```
App 进程                    系统侧
┌─────────┐           ┌──────────────┐
│ Log.d() │ ───────►  │  /dev/log/*  │  内核环形缓冲区（logd 守护进程）
└─────────┘  write()  │              │
                      └──────┬───────┘
                             │ read()
                      ┌──────▼───────┐
                      │   logcat     │  读取 & 过滤
                      └──────────────┘
```

**日志级别**：`V(verbose) < D(debug) < I(info) < W(warning) < E(error) < WTF(fatal)`

**缓冲区类型**：

| Buffer | 用途 |
|--------|------|
| main | 应用日志（Log.x） |
| system | 系统框架日志（Slog.x） |
| crash | 崩溃日志 |
| events | 结构化事件（EventLog） |
| radio | 通信相关 |

logcat 是环形缓冲区，容量有限，日志会被覆盖，所以线上环境需要持久化方案。

### 2.2 应用层日志框架的典型架构

```
┌────────────────────────────────────────┐
│            统一日志接口 Logger           │
├──────┬──────┬───────┬─────────────────┤
│Logcat│ File │Console│  Remote/Upload  │
│输出  │ 输出  │ 输出  │    上报通道      │
└──────┴──────┴───────┴─────────────────┘
```

常见封装：统一 TAG、自动附加调用位置、按 BuildConfig.DEBUG 控制输出级别。

### 2.3 文件日志持久化策略

```
写入策略
├── 直接写文件         简单，但频繁 I/O 影响性能
├── 内存缓冲 + 定时刷盘  平衡性能与实时性
└── mmap 映射写入       最优解，进程崩溃也不丢日志

文件管理
├── 按天/按大小滚动     log_2026-03-19.txt → log_2026-03-20.txt
├── 自动清理过期文件     保留最近 7 天
├── 压缩 + 加密         减少磁盘占用，保护隐私
└── 总大小上限          防止撑满磁盘
```

**mmap 是高性能日志方案的核心**：通过内存映射文件写入，无需 flush，即使进程崩溃已写入的数据也不会丢失。微信 xlog、美团 Logan 均采用此方案。

### 2.4 主流日志框架对比

| 框架 | 特点 |
|------|------|
| **Timber** | Jake Wharton 出品，轻量，Tree 机制可插拔 |
| **Logger** | 美观的格式化输出，带线程/方法信息 |
| **XLog** | 爱奇艺，支持文件写入、压缩、加密 |
| **Logan**（美团） | 高性能文件日志，mmap 写入 |
| **Mars-xlog**（微信） | C++ 实现，mmap + 压缩，性能极高 |

---

## 三、异常信息从用户设备到开发者的完整链路

### 3.1 全链路概览

```
用户设备                        网络                    服务端                   开发者
┌──────────┐              ┌──────────┐          ┌──────────────┐         ┌──────────┐
│ 异常发生  │              │          │          │              │         │          │
│    ▼      │              │          │          │              │         │          │
│ 捕获异常  │              │          │          │              │         │          │
│    ▼      │              │          │          │              │         │          │
│ 采集信息  │──── 上报 ───►│  HTTPS   │────►     │ 接收 & 解析   │         │          │
│    ▼      │              │          │          │    ▼         │         │          │
│ 本地持久化│              │          │          │ 聚合 & 归因   │──通知──►│ 分析定位  │
│(下次上报) │              │          │          │    ▼         │         │    ▼     │
└──────────┘              └──────────┘          │ 告警 & 展示   │         │ 修复发版  │
                                                └──────────────┘         └──────────┘
```

### 3.2 采集的信息内容

一条完整的崩溃报告包含：

| 类别 | 字段 | 说明 |
|------|------|------|
| **异常本身** | exceptionType, message, stackTrace | 崩溃类型、描述、完整调用栈 |
| **设备信息** | brand, model, sdkVersion, romVersion | 机型、系统版本 |
| **运行时状态** | totalMemory, availableMemory | 崩溃时内存状况 |
| **应用信息** | appVersion, versionCode, channel | 版本号、渠道 |
| **上下文** | currentActivity, threadName, isBackground | 当前页面、线程、前后台 |
| **行为路径** | breadcrumbs | 用户操作轨迹（面包屑） |
| **业务字段** | userId, orderId 等自定义信息 | 辅助定位 |

### 3.3 上报策略

**核心原则：先存后传** — 崩溃瞬间网络不可靠，必须先写本地文件。

```
上报时机
├── 下次启动时上报     最可靠，崩溃日志的主要方式
├── 实时上报           严重错误立即发送
├── 定时批量上报       非致命异常，攒一批一起发
└── WiFi 时上报        大文件（Native dump）省流量
```

**可靠性保障**：

| 策略 | 说明 |
|------|------|
| 先存后传 | 崩溃时只写文件，不做网络请求 |
| 失败重试 | 指数退避，最多重试 3 次 |
| 去重 | 同一堆栈不重复上报（本地 hash 去重） |
| 采样率 | 非致命异常按比例上报（如 10%） |
| 流量控制 | 单日上报上限，防止异常风暴 |
| 数据压缩 | gzip 压缩，减少用户流量消耗 |

上报实现示例：

```java
public class CrashUploader {
    // App 启动时调用，检查是否有待上报的崩溃
    public static void uploadPendingCrashes() {
        File crashDir = new File(context.getFilesDir(), "crash");
        File[] files = crashDir.listFiles();
        if (files == null) return;

        for (File file : files) {
            try {
                String json = readFileContent(file);
                boolean success = uploadToServer(json);
                if (success) {
                    file.delete();  // 上报成功才删除
                }
            } catch (Exception e) {
                // 上报失败，下次再试
            }
        }
    }

    private static boolean uploadToServer(String json) throws IOException {
        HttpURLConnection conn = (HttpURLConnection)
            new URL("https://crash.example.com/api/report").openConnection();
        conn.setRequestMethod("POST");
        conn.setRequestProperty("Content-Type", "application/json");
        conn.setRequestProperty("Content-Encoding", "gzip");
        conn.setDoOutput(true);

        GZIPOutputStream gzip = new GZIPOutputStream(conn.getOutputStream());
        gzip.write(json.getBytes());
        gzip.close();

        return conn.getResponseCode() == 200;
    }
}
```

### 3.4 服务端处理

```
客户端上报
    │
    ▼
┌──────────┐    ┌───────────┐    ┌──────────────┐    ┌──────────────┐
│ API 网关  │───►│ 消息队列   │───►│  数据处理服务  │───►│  数据存储     │
│ (Nginx)  │    │ (Kafka)   │    │              │    │ (ES/MySQL/   │
└──────────┘    └───────────┘    │ · 符号化还原   │    │  ClickHouse) │
                                 │ · 堆栈聚合    │    └──────┬───────┘
                                 │ · 指标计算    │           │
                                 └──────────────┘           ▼
                                                     ┌─────────────┐
                                                     │  管理后台/看板 │
                                                     │  · 崩溃列表   │
                                                     │  · 趋势图表   │
                                                     │  · 告警通知   │
                                                     └─────────────┘
```

#### 堆栈符号化还原（关键步骤）

线上 APK 经过 ProGuard/R8 混淆，上报的堆栈不可读：

```
# 混淆后
java.lang.NullPointerException
    at a.b.c.d(SourceFile:12)

# 用 mapping.txt 还原后
java.lang.NullPointerException
    at com.example.pay.PaymentManager.processOrder(PaymentManager.java:127)
```

**每次发版必须保存 mapping.txt 并上传到崩溃平台。**

#### 崩溃聚合

按堆栈 top N 帧的 hash 聚合，将成千上万条上报归类为若干个 Issue：

```
聚合前：10000 条上报
    ▼
聚合后：
  Issue #1  NullPointer@PaymentManager:127   (8500 次, 影响 3200 用户)
  Issue #2  OOM@ImageLoader:45              (1200 次, 影响 800 用户)
  Issue #3  IndexOutOfBounds@ListAdapter:67  (300 次, 影响 150 用户)
```

### 3.5 告警与触达开发者

**告警规则**：

| 条件 | 动作 |
|------|------|
| 新增崩溃 | 新版本出现从未见过的崩溃 → 立即通知 |
| 崩溃率突增 | 崩溃率超过阈值（如 0.1%） → 紧急告警 |
| 影响面扩大 | 某个已知问题影响用户数激增 → 升级告警 |
| 特定版本/机型 | 新版本崩溃率明显高于旧版 → 通知负责人 |

**通知渠道**：企业微信/飞书/钉钉机器人、邮件、短信（P0 级别）、电话（极端情况）。

**开发者在崩溃面板上看到的信息**：

- 符号化还原后的完整堆栈
- 影响版本、影响用户数、发生次数、崩溃率
- 设备/系统版本分布
- 用户行为路径（面包屑）
- 自定义业务字段（userId、orderId 等）

---

## 四、日志分级策略（线上 vs 开发）

| 级别 | 何时记录 | 写文件 | 上报 |
|------|---------|--------|------|
| **DEBUG** | 开发调试细节 | 否 | 否 |
| **INFO** | 关键业务节点（登录、支付、页面切换） | 是 | 否 |
| **WARNING** | 异常但可恢复（网络重试、降级） | 是 | 按采样率 |
| **ERROR** | 业务异常（接口失败、数据解析错误） | 是 | 是 |
| **FATAL** | 崩溃 / ANR | 是 | 立即上报 |

---

## 五、项目推荐方案

| 项目规模 | 推荐方案 |
|---------|---------|
| 小型项目 | Timber + 自定义 FileTree |
| 中型项目 | Timber + Logan/XLog（mmap 文件日志）+ Firebase Crashlytics |
| 大型项目 | 自研日志 SDK（mmap 写入 + 多通道 + 结构化日志 + 链路追踪）+ 服务端 ELK/ClickHouse |

**成熟崩溃平台对比**：

| 方案 | 类型 | 符号化 | 聚合 | 告警 | 成本 |
|------|------|--------|------|------|------|
| Firebase Crashlytics | SaaS | 自动 | 自动 | 支持 | 免费 |
| Bugly（腾讯） | SaaS | 自动 | 自动 | 支持 | 免费 |
| Sentry | 开源/SaaS | 自动 | 自动 | 丰富 | 免费/付费 |
| xCrash（爱奇艺） | 开源 | 需自建 | 需自建 | 需自建 | 免费 |

---

## 六、一条崩溃日志的完整生命周期

```
1. 用户操作触发崩溃（如 NPE）
2. UncaughtExceptionHandler 捕获异常
3. 采集堆栈 + 设备信息 + 行为路径 → 写入本地文件
4. 下次启动 → 读取崩溃文件 → gzip 压缩 → HTTPS 上报
5. 服务端接收 → Kafka 缓冲 → 消费处理
6. mapping.txt 符号化还原 → 按堆栈指纹聚合归类
7. 触发告警规则 → 飞书/企微机器人通知开发者
8. 开发者在看板查看堆栈 + 设备分布 + 用户路径
9. 定位问题 → 修复 → 发版 → 观察崩溃率回落
```

---

## 七、核心要点总结

1. **采集全面性**：Java 崩溃 + Native 崩溃 + ANR + 业务异常，四类都要覆盖
2. **先存后传**：崩溃时只写本地文件，下次启动再上报，保证数据不丢
3. **mmap 写入**：高性能日志持久化的核心技术，进程崩溃也不丢数据
4. **符号化还原**：每次发版必须保存 mapping.txt，否则线上堆栈无法阅读
5. **聚合归因**：服务端按堆栈指纹聚合，将海量上报归类为有限的 Issue
6. **实时告警**：新崩溃、崩溃率突增等关键事件自动通知开发者