# Natural-Language-To-Unit-Test-For-Web

针对 Web 页面将自然语言转化为单元测试 case 的 Claude Code skill。

## 安装

将本项目克隆到本地后，在 Claude Code 中自动发现并加载该 skill。

```bash
git clone <repo-url>
cd Natural-Language-To-Unit-Test-For-Web
# 在此目录下启动 Claude Code 即可使用 /nl2unit-test
```

## 使用

在 Claude Code 中输入 `/nl2unit-test`，然后用自然语言描述你的测试场景。例如：

> 帮我给用户列表页面生成测试，包括：列表正常渲染、分页点击、搜索过滤、加载状态和空数据展示

Skill 会自动识别项目的前端框架和测试框架，并生成符合项目规范的测试代码。

## Skill 结构

```
.claude/skills/nl2unit-test/
├── skill.json        # skill 元数据和配置
└── instructions.md   # skill 行为指令
```
