# SKILLS

这是我的 AI Skill 仓库，用来集中保存skill。每个 skill 都以独立目录维护，目录名与 skill 的 `name` 保持一致。

## Skill 列表

| Skill | 说明 | 适用场景 | 路径 |
| --- | --- | --- | --- |
| `tech-doc-generator` | 根据需求生成中文 Java 后端技术设计文档，遇到不明确设计决策时先向用户确认。 | Java、Spring Boot、DDD 分层架构下的模块技术设计文档生成。 | [`tech-doc-generator/`](./tech-doc-generator/) |

## 目录规范

每个 skill 推荐使用以下结构：

```text
skill-name/
├── SKILL.md
├── agents/
│   └── openai.yaml
├── references/
├── scripts/
└── assets/
```

只有 `SKILL.md` 是必需文件，其他目录按需添加。不要在单个 skill 目录里放无关的 README、安装说明、变更日志等文件，保持 skill 自包含且轻量。

## 新增 Skill 流程

1. 在仓库根目录新建一个与 skill `name` 一致的目录。
2. 在该目录下创建 `SKILL.md`，并确保 frontmatter 至少包含 `name` 和 `description`。
3. 如有详细参考资料，放入 `references/`；如有可执行工具，放入 `scripts/`；如有模板或资源文件，放入 `assets/`。
4. 更新本 README 的「Skill 列表」。
5. 提交前运行 skill 校验脚本，确认结构合法。

## 命名约定

- 目录名使用小写字母、数字和连字符。
- `SKILL.md` frontmatter 中的 `name` 与目录名保持一致。
- `description` 写清楚 skill 做什么，以及哪些用户请求会触发它。
