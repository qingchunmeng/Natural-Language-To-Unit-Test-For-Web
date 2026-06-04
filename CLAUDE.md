# Natural-Language-To-Unit-Test-For-Web

将自然语言描述转化为 Web 页面单元测试 case 的 Claude Code skill 项目。

## 项目结构

```
.
├── CLAUDE.md                        # 项目级上下文
├── README.md                        # 项目说明
├── LICENSE                          # MIT License
└── .claude/
    └── skills/
        └── nl2unit-test/
            ├── skill.json           # skill 元数据
            └── instructions.md      # skill 指令
```

## 开发规范

- skill 命名使用 kebab-case，如 `nl2unit-test`
- skill 描述使用简体中文
- `instructions.md` 遵循 TRIGGER/SKIP 约定，采用 `## 行为` 结构化描述工作流
- 修改 skill 后在 Claude Code 中重新加载即可生效
