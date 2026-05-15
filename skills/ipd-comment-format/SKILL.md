---
name: ipd-comment-format
description: |
  通过 mcp-issue-service 的 M_saveComment 给 IPD 问题(https://ipd.mioffice.cn/...)贴评论时的格式规范。换行只认 HTML <br>,不认 \n;参数只要 issueId(数字)+ issId("ISS-..."),不需要 tempId。常见踩坑、修复方法、参考模板都在这里。
  触发场景:
  - 用户说"在 JIRA / IPD 问题下面追加评论"
  - 用户给 ISS-* 问题编码 + 评论模板
  - 看到 mcp__mcp-issue-service__M_saveComment 调用前
inclusion: manual
---

# IPD 评论格式 Skill

通过 `mcp__mcp-issue-service__M_saveComment` 给小米 IPD 问题协作平台贴评论时,**HTML 换行 + 必填参数 + 排错** 的 cheatsheet。

## TL;DR

```python
mcp__mcp-issue-service__M_saveComment(
  issueId=302235,                      # 必填,从 https://ipd.mioffice.cn/.../item/<id> 提
  issId="ISS-202605-00014645A",       # 必填,问题编码
  content="第一行<br><br>第二段<br>段内换行",  # 用 <br>,不用 \n
  # 不需要 tempId
)
```

## 1. 必填参数

| 参数 | 类型 | 来源 |
|---|---|---|
| `issueId` | int | IPD 问题详情页 URL `https://ipd.mioffice.cn/issue-collaboration-platform/issue_info/item/<id>` 里的 `<id>`,纯数字 |
| `issId` | str | 问题编码 `ISS-YYYYMM-NNNNNNNNX`,在多维表格 / IPD UI 标题旁可见 |
| `content` | str | 评论正文 |

**`tempId` 不需要传**(实测可省)。

## 2. 换行规则(最重要)

IPD 把 content 存成 `[{"type":"text","text":"<原文>"}]` 单段结构,**渲染时不把 `\n` 转 `<br>`**。结果就是整段评论挤在一行,惨不忍睹。

### 正确做法

| 想要 | 用 |
|---|---|
| 行内换行 | `<br>` 或 `<br/>` |
| 段落分隔 | `<br><br>` |
| 行首缩进 | 全角空格 `　`(`<br>` 后 leading 半角 space 在 HTML 里折叠) |
| 列表项 | 用 `1. xxx<br>2. xxx<br>` 即可,不需要 `<ul><li>`(测过,IPD 不渲染 ul/li) |

### 错误示例(都试过,都不 work)

| 错误格式 | 为啥不行 |
|---|---|
| `"行1\n行2"` | `\n` 被当文本字符,UI 显示 `行1\n行2` 或 `行1行2` |
| `"行1\\n行2"` | 字面双反斜杠 + n,渲染出来是 `行1\n行2` |
| `[{"type":"text","text":"段1"},{"type":"text","text":"段2"}]` | API 把整个字符串当一段存,数组结构丢失 |
| `<p>段1</p><p>段2</p>` | `<p>` 不渲染 |
| `<ul><li>项1</li></ul>` | `<ul>` `<li>` 不渲染 |

### 实测验证步骤

如果不确定某种格式 work 不,贴一条小测试 + `M_getCommentList` 看返回 + 再去 IPD UI 肉眼看渲染:

```python
mcp__mcp-issue-service__M_saveComment(issueId=..., issId=..., content="[换行测试] 行1<br>行2")
# 然后 IPD UI 上看到「行1」换行「行2」=> 格式 OK
```

返回的 `M_getCommentList` 里 content 字段是 `[{"type":"text","text":"<原文>"}]`,但**单看 API 返回判断不了渲染**,必须实地肉眼校验 UI。

## 3. 完整模板(修复失败评论)

```
(可选 prefix)<br><br>问题:<className><br><br>change:<时间>, gerrit <NUM> patchset <PS> cherry-pick 进 monkey 验证<br><br>验证结果:<具体描述,实例数,stuck rate>。本轮 leak 报告中持续出现 <className>。<br><br>GC 路径:<br>GC ROOT → <step 1><br>　　　 ↓ <step 2><br>　　　 ↓ <step 3><br>　　　 ↓ ... → <className><br><br>原因:<br><1-3 句根因总结><br><br>建议修复方案:<br>1. <方案 1><br>2. <方案 2><br>3. <方案 3><br><br>详细分析:https://feishu.cn/docx/<token>
```

注意 GC 路径的缩进用全角空格 `　　　` 三个,跟 ↓ 箭头对齐;每步一个 `<br>`。

## 4. 排错

### Session not found

```
Streamable HTTP error: Error POSTing to endpoint:
{"code":-32002,"message":"Session not found: <uuid>"}
```

mcp-issue-service 的 server 端 session 过期。处理:

```bash
# 1. CLI 重 add(token 在 ~/.claude.json 里)
claude mcp remove mcp-issue-service -s user
claude mcp add --scope user --transport http mcp-issue-service \
  "http://one.mi.com/hubs-server/mcp/streamableHttp/mcp-issue-service" \
  --header "x-authenticated-user: <你的小米邮箱前缀>" \
  --header "X-Supplychain-Token: <你的 IPD token>"

# 2. 但是 Claude Code 当前会话 cache 着旧 session id
#    必须用 /mcp 命令手动 reconnect:
#    在终端里输入 /mcp,选 mcp-issue-service,reconnect
```

reconnect 后立刻重试 `M_saveComment` 就 OK。

### 没有 update / delete API

`M_saveComment` 只能 **追加**,不能改也不能删。贴错的话只能再贴一条 `(排版更正,以本条为准 — 前一条请忽略)<br><br>...`。

### 富文本不工作

IPD UI 对 HTML 支持很有限。**只确认 `<br>` 和裸 URL 文本(自动识别为链接)能用**。其他 markdown / HTML(粗体、斜体、列表、表格)都不要试。复杂表达靠 `<br>` + 文字描述 + 飞书 docx 链接。

## 5. 配套 memory

如果使用方有持久 memory 系统(类似 `~/.claude/projects/.../memory/`),把 `<br>` 这个事实写进 reference memory 一份,跨会话立刻可用。参考 `~/.claude/projects/-home-mi-performance-memory/memory/reference_ipd_comment_format.md`。

## 6. 实战来源

- 2026-05-15 桌面 monkey 闭环验证,3 条修复失败评论先用 `\n` 贴失败,再切 `<br>` 成功
- 验证 issue:ISS-202605-00014645A / ISS-202605-00014652A / ISS-202605-00017045A
- 最初的 A/B/C 测试只有 A(`<br>`)work
