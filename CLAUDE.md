# AISkill

Kiro 和 Claude Code 共用的自定义 skills 仓库。这里**不是一个运行时工程**——没有构建、没有测试、没有依赖。所有内容都是 Markdown + YAML frontmatter，由各 AI 工具在用户调用 `/<skill-name>` 时加载。

## 使用方式

本仓库只是 skill **源文件**的集中存放点，实际被 Kiro / Claude Code 读取的位置通过**软链接**指向这里。这样一次修改、多处生效，也方便跨机器同步（仓库跟 git 走即可）。典型软链：

```bash
# Claude Code（用户级）
ln -s /Users/xiamin/Code/AISkill/skills ~/.claude/skills

# Kiro 同理，链到它的 skills 目录
```

## 目录结构

```
skills/
├── SKILL.md              # 根 skill（工作大助手），路由到各子 skill
├── codeSkill/SKILL.md    # 代码解释/优化（Rust、Dart/Flutter 偏性能）
├── ReviewSkill/SKILL.md  # 代码 review
└── WriteDoc/SKILL.md     # 飞书文档编写规范
```

每个 `SKILL.md` 的 YAML frontmatter 声明 `name` / `description` / `inclusion`。`inclusion: always` 代表全局加载。

## 工作流

- **改 skill** = 直接编辑对应的 `SKILL.md`
- **加新 skill** = 新建 `skills/<NewSkill>/SKILL.md`，frontmatter 填好，并在根 `skills/SKILL.md` 里加一条路由
- **发布** = `git push origin main`（SSH 已配好，无需额外凭证）

## 语言约定

Skill 内容全部用**中文**编写，面向中文用户。PR 标题、commit message 也遵循这个习惯。
