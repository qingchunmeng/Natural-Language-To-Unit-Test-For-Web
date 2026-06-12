# Natural-Language-To-Unit-Test-For-Web

将自然语言描述转化为 Web 页面单元测试 case 的 Claude Code skill 项目。

## 项目结构

```
.
├── CLAUDE.md                        # 项目级上下文
├── README.md                        # 项目说明
├── LICENSE                          # MIT License
├── xmind/                           # XMind 测试用例示例文件
└── .claude/
    └── skills/
        ├── nl2unit-test/            # 自然语言 → 单元测试
        │   ├── skill.json
        │   └── instructions.md
        └── xmind2unit-test/         # XMind → 单元测试（通用引擎）
            ├── skill.json
            └── instructions.md
```

## 可用 Skill

### /nl2unit-test
将用户的自然语言描述转化为 Web 页面单元测试 case。
触发：用户输入自然语言测试描述。

### /xmind2unit-test
通用 XMind → UI 单元测试转化引擎。解析任意业务的 XMind 思维导图，
通过 OCR + 图像识别关联 UI 截图和设计稿，将语义化测试描述映射为
可执行的 Web 测试代码。

触发：用户提供 `.xmind` 文件路径。
支持：Vue 3 / React / 纯 Playwright E2E。
核心能力：
  - 自动解析 XMind 内容（content.json）
  - 读取 XMind 内嵌的需求文档 URL
  - OCR + 图像识别分析 UI 截图和 MasterGo 设计稿
  - 语义化描述 → UI 元素选择器的通用映射
  - 优先级过滤 + 4 个用户确认关卡

## 开发规范

- skill 命名使用 kebab-case，如 `nl2unit-test`
- skill 描述使用简体中文
- `instructions.md` 遵循 TRIGGER/SKIP 约定，采用 `## 行为` 结构化描述工作流
- 修改 skill 后在 Claude Code 中重新加载即可生效
