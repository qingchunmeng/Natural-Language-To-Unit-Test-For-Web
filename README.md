# Natural-Language-To-Unit-Test-For-Web

针对 Web 页面将自然语言转化为单元测试 case 的 Claude Code skill。

## 安装

将本项目克隆到本地后，执行以下命令将 skill 安装到系统级 `.claude/skills` 目录（全局可用）：

```bash
# 安装 /nl2unit-test
mkdir -p ~/.claude/skills/nl2unit-test && cp .claude/skills/nl2unit-test/SKILL.md ~/.claude/skills/nl2unit-test/SKILL.md

# 安装 /xmind2unit-test
mkdir -p ~/.claude/skills/xmind2unit-test && cp .claude/skills/xmind2unit-test/SKILL.md ~/.claude/skills/xmind2unit-test/SKILL.md
```

安装后，在任意项目中启动 Claude Code 即可使用 `/nl2unit-test` 和 `/xmind2unit-test`。

## 使用

### /nl2unit-test — 自然语言 → 单元测试

输入自然语言描述测试场景即可：

> 帮我给用户列表页面生成测试，包括：列表正常渲染、分页点击、搜索过滤、加载状态和空数据展示

### /xmind2unit-test — XMind → 单元测试

提供 `.xmind` 文件路径，Skill 自动解析思维导图中的测试用例并生成 Playwright E2E 测试代码：

```
/xmind2unit-test xmind/理科精准学430.xmind
```

支持优先级过滤、多框架（Vue/React/纯 Playwright）、OCR 截图分析和数据 mock 模板化。

## Skill 结构

```
.claude/skills/
├── nl2unit-test/
│   └── SKILL.md          # 自然语言 → 单元测试
└── xmind2unit-test/
    └── SKILL.md          # XMind → 单元测试（通用引擎）
```
