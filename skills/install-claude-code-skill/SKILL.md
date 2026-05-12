---
name: install-claude-code-skill
description: |
  Install a community Claude Code skill from GitHub while avoiding common pitfalls.
  Use when: (1) 用户说"装 XX skill / install this skill / 从 <repo> clone 一个 skill",
  (2) 给定一个 skill 名字要找对应仓库,(3) 多个同名仓库要挑主流版本,
  (4) 改动 ~/.claude/skills/ 目录(搬文件、做备份、加软链),
  (5) 评估一个 skill 默认推荐的 UserPromptSubmit / Stop hook 要不要装。
  Covers: 按 star 找主流仓库、skill 热加载发现机制、备份不能放 skills 目录内、
  软链兼容、hook 副作用评估。
author: Claude Code
version: 1.0.0
date: 2026-05-08
---

# 安装社区 Claude Code Skill

## Problem

从 GitHub 装 Claude Code skill 看起来是 `git clone` 一步到位,但实际常踩 4 个坑:
1. **装错 fork**:搜出来的第一个仓库不一定是主流,社区 fork 和原作的 star 数可能差 3 个数量级
2. **备份污染 skills 列表**:在 `~/.claude/skills/` 下做 `mv X X.bak` 会导致备份目录本身被当成一个新 skill 注册
3. **默认 hook 未必适合**:README 推荐装的 UserPromptSubmit hook 会在每一轮对话注入上下文,对琐事会话是噪音
4. **不知道生效机制**:不清楚要不要 `/exit` 重启、软链会不会被扫

这个 skill 把这些坑的应对全部写清楚,把安装变成可复制流程。

## Context / Trigger Conditions

触发场景(任一命中即用):

- 用户说:"帮我装 XX skill"、"clone 这个 skill 仓库"、"install a Claude Code skill"
- 用户给出一个不带作者前缀的 skill 名字(如 "装 Claudeception"),需要先锁定主流仓库
- 用户已经在 `~/.claude/skills/` 下操作,要做备份 / 改名 / 迁移
- README 要求写入 `~/.claude/settings.json` 的 hook,要判断是否必要

## Solution

### Step 1 · 按 star 找主流仓库

同一个 skill 名字下往往有原作 + 多个 fork。**不要看搜索排名**(搜索引擎/WebSearch 返回的第一个往往不是主流)。用 GitHub API 按 star 排序:

```bash
curl -s "https://api.github.com/search/repositories?q=<skill-name>&sort=stars&order=desc&per_page=5" \
  | jq -r '.items[] | "\(.stargazers_count) ⭐  \(.full_name)  — \(.description)"'
```

挑 star 数最高、描述对得上的那个。差 2 个数量级以上(如 2.3k vs 6)基本就是原作 vs fork 的区别。

### Step 2 · 确认 `~/.claude/skills/` 的形态

```bash
ls -la ~/.claude/skills
readlink -f ~/.claude/skills
```

三种可能:
- 普通目录:直接往里 clone
- 顶层软链(如 → `~/Documents/AISkill/skills`):clone 会顺着软链落到实体位置,git 仓库追踪也在实体端,没问题
- 不存在:`mkdir -p ~/.claude/skills`

### Step 3 · 克隆到 skills 目录

```bash
git clone --depth 1 https://github.com/<owner>/<Repo>.git ~/.claude/skills/<skill-dir-name>
```

注意 `<skill-dir-name>` 习惯用 **小写 kebab-case**,即便原仓库名是 PascalCase(如 `blader/Claudeception` → 落成 `claudeception`)。Claude Code 用目录名在 available-skills 列表里做 key,SKILL.md 的 `name:` 字段是逻辑名,两者一致更好排查。

### Step 4 · 如果要覆盖旧版,**备份必须挪出 skills 目录**

这是本 skill 存在的核心原因。以下做法是**错的**:

```bash
# ✗ 错:备份留在 skills 目录里,会被当成新 skill 扫到
mv ~/.claude/skills/foo ~/.claude/skills/foo.bak
git clone ... ~/.claude/skills/foo
```

正确做法:

```bash
# ✓ 对:备份搬到 skills 目录外
mv ~/.claude/skills/foo ~/.claude/foo.bak  # 或放 /tmp、~/backup 等
git clone ... ~/.claude/skills/foo
```

**扫描规则**:Claude Code 把 `~/.claude/skills/*/SKILL.md` 每个含 SKILL.md 的直接子目录注册为一个 skill。目录名任意(包括 `foo.bak`、`foo-old` 这类)都会被登记。验证无误后再 `rm -rf` 备份。

### Step 5 · 评估 hook 要不要装

不少 skill(如 claudeception)会推荐装一个 `UserPromptSubmit` 或 `Stop` hook 做"强制评估"。在改 `~/.claude/settings.json` 之前:

```bash
# 读 hook 脚本全文,看它往 stdout 输出什么
cat ~/.claude/skills/<name>/scripts/*.sh
```

判断原则:

| 脚本行为 | 推荐 |
|---|---|
| 只在特定条件下(如命令失败)输出 | 装 |
| 每次 UserPromptSubmit 都往 stdout 灌一段"MANDATORY / CRITICAL"文本 | **不装**,靠 skill description 匹配 + 显式短语触发就够了 |
| 有网络请求 / 写文件 / 调 API | 逐行审完再决定,别盲装 |

**记住**:hook 不是必需的。skill 仍可被 description 语义匹配触发,或用 `/skill-name` 显式触发。先不装 hook,有明确诉求再加。

### Step 6 · 验证已装好

**无需 `/exit` 重启** —— Claude Code 的 skill 发现是热加载的,下次交互时注入的 available-skills 列表会自动包含新 skill。验证方式:

- 看下一轮 system reminder 里的 skill 列表,应出现新 skill,描述对得上
- 显式试:说"/<new-skill-name>" 或一句能命中 description 触发词的话

如果列表没出现,再看:
- SKILL.md frontmatter 是否有 `name:` 和 `description:` 两个字段(缺一不可)
- 目录结构是否是 `~/.claude/skills/<name>/SKILL.md`(不是 `~/.claude/skills/<name>.md`)

## Verification

装完走一遍:
```bash
# 1. 看扫描路径正确
find -L ~/.claude/skills -maxdepth 2 -name 'SKILL.md'

# 2. 新 skill 的 frontmatter 合规
head -15 ~/.claude/skills/<new>/SKILL.md

# 3. 确认没有散落的备份目录在 skills 内
ls ~/.claude/skills | grep -E '\.(bak|old|backup)$' && echo "⚠️ 发现备份,挪出去" || echo "✓ 干净"
```

下一轮交互看 available-skills 列表确认注册成功。

## Example · 装 Claudeception(2026-05-08 实录)

**目标**:装 `Claudeception`(一个从会话中提炼 skill 的 meta-skill)。

1. **搜主流仓库**
   ```bash
   curl -s "https://api.github.com/search/repositories?q=claudeception&sort=stars&order=desc&per_page=3"
   ```
   → `blader/Claudeception`(2.3k stars,原作)vs `abhattacherjee/claudeception`(6 stars,fork)
   → 选 **blader**

2. **克隆**
   ```bash
   git clone --depth 1 https://github.com/blader/Claudeception.git ~/.claude/skills/claudeception
   ```

3. **Hook 评估**:查看 `scripts/claudeception-activator.sh`,发现每次 `UserPromptSubmit` 都输出 "MANDATORY SKILL EVALUATION REQUIRED" 文本 → 对日常琐事会话是噪音,**决定不装**

4. **验证**:下一轮 system reminder 的 skills 列表里看到 `claudeception`,描述是 blader 版原文,完成

**踩坑复盘**(被本 skill 固化):
- 初次搜索错装了 `abhattacherjee` 版 → 后来用 star 排序才看清
- 做 `mv ~/.claude/skills/claudeception ~/.claude/skills/claudeception.bak`,结果 `claudeception.bak` 自己也被注册成一个 skill → 改成搬到 `~/.claude/` 才干净

## Notes

- **软链是一等公民**:`~/.claude/skills` 本身可以是软链(如 → `~/Documents/AISkill/skills`),`~/.claude/skills/<name>` 也可以各自是软链。扫描时用 `find -L`(跟符号链接)可以看到全部。早期怀疑"软链不被扫描"是误判 —— 真正原因是 frontmatter 缺 `description` 或扫描时机问题。

- **热加载不需要重启**:Claude Code 每轮对话扫描 skills 目录,改 frontmatter / 加删子目录下一轮就生效。但**缓存的 description 不一定马上刷新**,若连续几轮看到的还是旧列表,`/exit` 重开一次即可。

- **skill 命名冲突**:两个 skill 的 `name:` 字段相同时,行为未定义。如果是备份/迁移场景,建议把旧的那个重命名 `name:` 字段或移出 skills 目录。

- **MCP ≠ skill**:MCP server(如 drawio)装在 `~/.claude.json`,命令是 `claude mcp add ...`;skill 装在 `~/.claude/skills/<name>/SKILL.md`,靠目录发现。两者触发机制和安装位置都不同,不要混淆。

## References

- [blader/Claudeception](https://github.com/blader/Claudeception) —— 自动从会话提炼 skill 的 meta-skill
- [anthropics/skills](https://github.com/anthropics/skills) —— Anthropic 官方 skill 仓库 + 模板
- [obra/superpowers · writing-skills](https://github.com/obra/superpowers/blob/main/skills/writing-skills/SKILL.md) —— TDD 风格创建 skill 的方法论
