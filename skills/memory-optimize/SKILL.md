---
name: memory-optimize
description: |
  降低 Android app 运行时内存占用(RSS / PSS / Native Heap / Dart heap / GPU 等)。
  Use when: (1) 用户说"内存优化 / 降内存 / 减少 RAM / 内存降不下来 / XX 进程吃内存",
  (2) `dumpsys meminfo <pkg>` 任一分类偏高:Native Heap > 50MB、Code > 80MB、
  Graphics > 40MB、Private Other 过大,(3) smaps 里 `[anon:scudo:primary]` /
  `[bare anon]`(Dart VM heap)稳态不下降,(4) 长期驻留进程(桌面/SystemUI/服务)
  做内存治理,(5) Flutter + Rust + C++ 混合 app 调优。
  内容:各内存区域归属与可控性速查、已验证的优化方案(scudo purge / 触发 Dart
  compact 等)、失败路径黑名单(避免重走)、定位工具命令模板。
author: Claude Code
version: 1.0.0
date: 2026-05-11
---

# Android App 内存优化

> **结论先行**:不同内存区域归**不同 allocator** 管,降它们的手段**完全不同**。
> 先用 smaps 定位主要区域,再挑对应方案,**不要乱试**(有些方案会反效果)。

## 1. 内存分布 Mental Model

`dumpsys meminfo` 和 smaps 对应关系:

| dumpsys 分类 | smaps 中的名字 | 归谁管 | 可控性 | 推荐方案 |
|---|---|---|---|---|
| **Native Heap** | `[anon:scudo:primary]` + `[anon:scudo:secondary]` | libc / scudo allocator | ✅ 高 | **方案 A** |
| (Private Other) | `[anon]`(bare,无名匿名)VSS 很大 RSS 少 | Dart VM heap | ⚠️ 中 | **方案 B**,谨慎 |
| **Code** | `.so mmap` / `.apk mmap` / `.dex mmap` | 内核 mmap | ❌ 低(除非平台) | 不推荐应用层改 |
| **Graphics** | `/dev/kgsl-3d0` + `EGL mtrack` + `GL mtrack` | GPU 驱动 | ⚠️ 中 | **方案 C**(半有效) |
| **.ttf mmap** | 字体文件 mmap | mmap | ❌ 低 | — |
| **Stack** | `[anon:stack_and_tls:*]` | 线程栈 | ⚠️ 中 | 减少线程数 |
| **Unknown** | 其他带名字 anon 段 | 各组件 | 视情况 | — |

**关键认知**:
- **`ui.Image` 的 pixel 数据主要在 GPU**(Flutter Picture→SkImage 走 GPU rasterize),CPU heap 几乎不占。清 Dart ImageCache / 自建 IconCache 对 CPU PSS 影响小
- **Dart VM heap 只能往下降、不能压缩到比 live 对象更小**,Dart VM 不主动缩堆
- **.so 代码段 Private Clean 降不下去**,除非让 zygote 预加载分摊成 Shared

## 2. 方案 A:scudo `mallopt(M_PURGE)` 主动回收 ⭐

### 原理

Android 11+ 默认 allocator = **LLVM scudo**。scudo 把 free 掉的 slab **retain 在 free list 里不还内核**,为了分配复用性能。结果:开机期峰值分配留下的空闲 slab **长期留在 Native Heap PSS**。

两个 `mallopt` 命令组合:
- `mallopt(M_DECAY_TIME, 0)` —— 改策略,今后 free 立即 `madvise(DONTNEED)` 还内核
- `mallopt(M_PURGE, 0)` —— 立刻扫一遍 free list,把历史囤积也还了

### Dart / Flutter 版(FFI)

```dart
import 'dart:async';
import 'dart:ffi';
import 'dart:io';

int _mallopt(int cmd, int value) {
  try {
    final fn = DynamicLibrary.process()
        .lookupFunction<Int32 Function(Int32, Int32), int Function(int, int)>('mallopt');
    return fn(cmd, value);
  } catch (_) { return -1; }
}

String _status() {
  final p = <String>[];
  for (final l in File('/proc/self/status').readAsLinesSync()) {
    if (l.startsWith(RegExp(r'(VmRSS|RssAnon|RssFile|VmSwap):'))) {
      final s = l.split(RegExp(r'\s+'));
      p.add('${s[0].replaceAll(':', '')}=${s[1]}kB');
    }
  }
  return p.join(' ');
}

void scheduleMemoryPurge() {
  _mallopt(-100, 0); // M_DECAY_TIME=0
  Timer(const Duration(seconds: 30), () {
    void tick(String tag) async {
      final before = _status();
      try { PaintingBinding.instance.handleMemoryPressure(); } catch (_) {}  // 方案 B 联动
      final r = _mallopt(-101, 0); // M_PURGE
      await Future.delayed(const Duration(milliseconds: 200));
      print('[MemPurge] $tag ret=$r | BEFORE { $before } | AFTER { ${_status()} }');
    }
    tick('first');
    Timer.periodic(const Duration(minutes: 5), (_) => tick('periodic'));
  });
}
```

**调用点**:入口函数 `runApp()` 之后调一次 `scheduleMemoryPurge()`。

### Kotlin/Java 版(JNI)

```cpp
// native.cpp
#include <jni.h>
#include <malloc.h>
extern "C" JNIEXPORT jint JNICALL
Java_com_example_MemUtil_malloptPurge(JNIEnv*, jclass) {
    mallopt(-100, 0);
    return mallopt(-101, 0);
}
```

```kotlin
object MemUtil {
    init { System.loadLibrary("myjni") }
    external fun malloptPurge(): Int
}
```

### 实测收益

典型降幅:**scudo:primary -15 ~ -25 MB,scudo:secondary -5 ~ -10 MB**。稳态 Native Heap 可降 15-30%。

## 3. 方案 B:`PaintingBinding.handleMemoryPressure()` 降 Dart heap + Skia cache

### 原理

Flutter 的 `PaintingBinding.handleMemoryPressure()`:
1. 清空 **Flutter `ImageCache`**(释放 decoded Image 引用)
2. 通过 engine 的 system channel 发送 `memoryPressure` 消息给 C++ engine
3. engine 内部调用 Dart VM 的 **compact + GC**(不是公开 API,内部路径)
4. 部分 Skia raster cache 被 purge

### 使用

```dart
// 启动 30 秒后首次触发(与方案 A 同一 Timer 即可)
Timer(const Duration(seconds: 30), () {
  PaintingBinding.instance.handleMemoryPressure();
});
```

### 实测收益

- 启动峰值期(Dart heap ~68MB)触发 → heap 降到 ~41MB(**-27MB**)
- 稳态期触发 → 降 0-5MB(GC 已自然做过)
- **和方案 A 配合** 在同一个 tick 里连调,一次性清 CPU + GPU cache

### ⚠️ 注意

- 仅在 **Flutter Framework layer 之上** 调(有 `PaintingBinding` 实例的环境)
- 不保证触发 Dart VM GC(取决于 engine 实现;release 版 `libhyper_os_flutter.so` 不导出 `Dart_NotifyLowMemory`,FFI 调不到 VM API)
- **`onTrimMemory` 路径更彻底**:若 app 被 Android 系统发到 `TRIM_MEMORY_RUNNING_CRITICAL`,engine 走完整路径清得最干净。但 foreground app 一般收不到;手动触发要改 Activity(桌面/系统 app 可能 Activity 源码不在应用仓库)

## 4. 方案 C:GPU / Graphics 相关

| 目标 | 手段 | 有效性 |
|---|---|---|
| EGL mtrack 大 | 降 `imageCache.maximumSizeBytes`(Flutter `PaintingBinding.instance.imageCache`) | 中 |
| kgsl-3d0 大 | 减少 `RepaintBoundary` 数量、合并 Layer | 中(需审查 widget 树) |
| GPU driver 基础开销 | 无法(Vulkan/Adreno 驱动本身 ~20MB 固定开销) | ❌ |
| Widget 截图 snapshot(`boundary.toImage`) | 用完立即 `image.dispose()` | 高 |

## 5. ❌ 失败方案黑名单(已验证无效或反效果)

**不要重走这些路径**:

| 手段 | 结果 | 原因 |
|---|---|---|
| 分配大量 garbage 诱发 Dart GC | 🔴 反效果,Dart heap 永久扩 | Dart old-gen 扩了不缩;实测 55MB→387MB |
| FFI 调 `Dart_NotifyLowMemory` | 🔴 symbol 未导出 | HyperOS Flutter engine release 版 strip 了所有 `Dart_*` embedder API |
| DartVmArgs `--old_gen_heap_size=N` | 🔴 反效果,内存反涨 | meta-data 注册导致额外 bootstrap 开销,net 负收益 |
| `android:hasCode="true"` + 自定义 Application | ⚠️ 测量噪声,净效果 ~1MB | 启动路径变化让某些 .so 从 shared 变 private,抵消 |
| MethodChannel 触发 MainActivity.onTrimMemory | 🔴 channel 不通 | 桌面真实 Activity 可能不是 MainActivity,channel handler 没被注册到真 engine |
| `mallopt(M_PURGE)` 不配合 `M_DECAY_TIME=0` | ⚠️ 只有一次性降,很快又涨回 | scudo 持续囤积,需要改策略才持续低位 |
| 清 Dart 层 Icon cache(IconImageCache / IconCache / BigIconCache 等) | 🔴 对 native heap PSS 几乎无影响 | `ui.Image` GPU-backed,CPU heap 不存 pixel |
| 频繁(< 1 分钟)调 `mallopt(M_PURGE)` | ⚠️ CPU 浪费 | 每次遍历 free list + 多次 madvise syscall |
| 每次 dump 都读 `/proc/self/smaps` | 🔴 测量污染 | 15 万行文本读取导致大量分配,secondary VMA 数爆涨,**扰动数据** |
| 外部 `am send-trim-memory RUNNING_CRITICAL` | ⚠️ 一次性有效(-17MB Dart heap) | foreground 进程不能重复 set 同级 level,首次有效 |

## 6. 定位工具速查

### 快速全景(单条命令)

```bash
PID=$(adb shell pidof com.example.app)
adb shell "cat /proc/$PID/smaps" | python3 -c '
import re, sys
H=re.compile(r"^[0-9a-f]+-[0-9a-f]+")
agg={}; cur=None
for l in sys.stdin:
    if H.match(l):
        if cur:
            n = cur["n"] or "[bare-anon]"
            if "/" in n and not n.startswith("["): n = n.rsplit("/",1)[-1]
            agg.setdefault(n,{"pss":0,"vma":0})
            agg[n]["pss"] += cur["p"]; agg[n]["vma"] += 1
        s = l.strip().split()
        cur = {"n": s[-1] if len(s)>5 else "", "p": 0}
    elif cur and l.startswith("Pss:"):
        cur["p"] = int(l.split()[1])
tot = sum(v["pss"] for v in agg.values())
for n,v in sorted(agg.items(), key=lambda x:-x[1]["pss"])[:15]:
    print(f"{n[:40]:<40} {v[\"vma\"]:>4} VMA  {v[\"pss\"]/1024:>7.2f} MB")
print(f"TOTAL PSS = {tot/1024:.1f} MB")
'
```

### 各分类

```bash
# Android 层视角
adb shell dumpsys meminfo com.example.app

# 进程内实时(在自己代码里读)
cat /proc/self/status | grep -E "VmRSS|RssAnon|RssFile|VmSwap|VmSize"

# scudo 专项
adb shell "cat /proc/$PID/smaps" | awk '
  /^[0-9a-f]+.*scudo:primary]/   { p=1; s=0; next }
  /^[0-9a-f]+.*scudo:secondary]/ { s=1; p=0; next }
  /^[0-9a-f]+/ { p=0; s=0 }
  p && /^Pss:/ { pri += $2 }
  s && /^Pss:/ { sec += $2 }
  END { print "primary="pri/1024"MB secondary="sec/1024"MB" }'
```

### 调用栈级(需要 symbols)

```bash
# heapprofd(需要 app 是 profileable 或 debuggable)
# manifest 加:<profileable android:shell="true"/>
adb shell am force-stop com.example.app
python3 /path/to/heap_profile -n com.example.app -d 90000 -i 10 --all-heaps
# 输出 /tmp/<id>/raw-trace,用 trace_processor SQL 分析
```

**限制**:`heap_profile` 只抓 `malloc/calloc/new`,抓不到 mmap 内存(`.so` / 匿名 arena / Dart VM heap);被 strip 的 .so 内部函数无法 resolve(只能看到 leaf `malloc` + 模块名)。

## 7. 真实案例:HyperLauncher Flutter 桌面

| 指标 | 基线 | 应用方案 A + B 后 | 变化 |
|---|---|---|---|
| TOTAL PSS | 239 MB | **197 MB** | **↓ 42 MB (17.5%)** |
| Native Heap (scudo) | 85 MB | 55 MB | ↓ 30 MB |
| Dart heap (bare anon) | 55 MB | 40 MB | ↓ 15 MB |

**代码量**:~30 行 Dart(一个 `scheduleMemoryPurge` 函数),无原生改动。

## References

- [Android bionic libc `malloc.h`(M_PURGE / M_PURGE_ALL / M_DECAY_TIME 官方定义)](https://android.googlesource.com/platform/bionic/+/refs/heads/main/libc/include/malloc.h)
- AOSP 自身先例:设备 logcat 能看到 `BluetoothServiceJni: ... trimNativeHeapNative: mallopt(M_PURGE_ALL) ret=1`,AOSP Bluetooth 模块也这么 trim,间隔 ~5 分钟
- [LLVM scudo allocator 设计文档](https://source.android.com/docs/security/test/scudo)
- Perfetto heap profiling:https://perfetto.dev/docs/data-sources/native-heap-profiler

## 何时**不**用本 skill 的方案

- Android < 11:scudo 非默认,`M_PURGE` 可能不支持
- 高频 malloc 热路径(音视频编解码):周期 purge 有 CPU 开销
- 瓶颈不在 native heap(比如 Dart heap / GPU texture / .so 代码段)—— 先用 §6 工具定位,再挑对应方案
- app 已换非 scudo allocator(mimalloc / jemalloc 等):mallopt 私有 cmd 不兼容

---

## ✍️ 后续沉淀:新方案模板

这份 skill 是**持续沉淀**的内存优化档案库。新方案验证成功后,按以下模板加一节:

```
## 方案 X:<手段名>
### 原理
### 使用(代码片段)
### 实测收益
### 注意 / 限制
```

**失败方案**也沉淀到 §5 黑名单,标注原因,避免后续重走。
