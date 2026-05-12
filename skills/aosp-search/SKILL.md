---
name: aosp-search
description: 在线查找 AOSP（Android 开源）源码、回答"哪个进程/文件/函数用了 XX"类问题时使用。强制：任何论断必须带 Gitiles URL + 文件路径 + 行号 + 代码片段作为证据，禁止凭记忆列清单。TRIGGER：用户说"查 AOSP / 在线源码 / android 源码里 / 哪个文件用了 XX / 哪个进程调了 XX / 去 cs.android.com 搜 / 去 android source 看 / XX 在 Android 里怎么实现的"。
---

# AOSP 在线源码调研

## 总原则

**没有证据的论断等于编造。** 说"X 进程/文件用了 Y"之前，必须能贴出：① Gitiles URL、② 文件路径、③ 行号、④ 代码片段。用户质疑具体文件时（"X.cpp 里有吗？"），不管多有信心也先 fetch 再答。默认措辞是"我还没找到"，不是"没有"。

## 工具选型（实测）

### ✅ 可用：直接抓单个源文件（Gitiles）

URL 模板：
```
https://android.googlesource.com/platform/<project>/+/refs/heads/main/<path>
```

示例：
```
https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/jni/com_android_internal_os_Zygote.cpp
```

### ⚠️ 关键陷阱：WebFetch 会静默截断大文件

实测 SurfaceFlinger.cpp（约 1 万行）、AndroidRuntime.cpp 抓回来只是片段。

**结论：大文件上 "未发现 XX" 是 FALSE NEGATIVE，不能作为"此文件无 XX"的证据。** 只有 < 2000 行的小/中文件 "未发现" 才可采信。

### ❌ 不可用（实测失败）

| 工具 | 失败模式 |
|---|---|
| cs.android.com/search | SPA 需 JS，WebFetch 拿不到结果 |
| grep.app | 单请求可用；并发立刻 429 |
| DuckDuckGo /html | 60s 超时 |
| Bing | 重定向 cn.bing.com 返回完全不相关内容 |
| aospxref.com | ECONNREFUSED |
| searchcode.com | 返回工具说明页而非结果 |
| sourcegraph.com | JS 密集，WebFetch 看不到结果 |
| android-review.googlesource.com/q/ | HTML 空 |
| github.com/search | 代码搜索需登录 |

## WebFetch prompt 模板

抓单文件搜字符串时，用以下模板，不要偷懒简化：

```
在整个页面源码里精确搜索 "<KEYWORD>" 字符串（大小写敏感）。
出现就列：完整代码行 + 行号 + 前后 2-3 行上下文；
可能有多处，全部列出，不要省略。
如果通篇没有，明确说"未发现 <KEYWORD>"。
不要推测，不要补充你没看到的内容。
```

缺任一条，下游 LLM 都可能漏、截、脑补。

## 搜不到时的兜底

不要凭记忆硬答。两条：

1. **让用户本地 grep**（AOSP 树通常在手边），给现成命令：

   ```bash
   cd <AOSP_ROOT>
   grep -rn --include='*.cpp' --include='*.cc' --include='*.c' \
       -E '[^a-zA-Z_]<KEYWORD>\s*\(' \
       frameworks/ system/ external/ packages/ 2>/dev/null \
       | grep -v '/tests/' | grep -v '<impl_path>'
   ```

2. **坦白工具限制**——给三张清单：

   - **已验证**（带 URL 证据）
   - **已验证不含**（也带证据，仅限小文件）
   - **盲区 / 未覆盖**（明说，不掩饰）

## 输出格式

每条论断必须同时给：

- Gitiles URL（用户可点开核对）
- 文件路径（相对 AOSP 根）
- 行号 + 调用片段

示例：

> `frameworks/base/core/jni/com_android_internal_os_Zygote.cpp` 第 506 行：
> `mallopt(M_DECAY_TIME, 1);`
> 来源：https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/jni/com_android_internal_os_Zygote.cpp

## 常见坑复盘

- **不要说"AOSP 里没有 XX.cpp"**——除非已 fetch 过目录树。默认说"我没找到"，让用户提供正确路径比错否定更省事。
- **不要用 `frameworks/av/services/mediaserver/main_mediaserver.cpp` 这种"看起来对"的路径硬试**——实测 404。拿不准时先抓目录 index。
- **初回答前不许列"凭印象认识的几个候选"**。列就必须全部 fetch 过、带 URL。凭记忆的列表只会被用户一个个推翻，浪费双方时间。
