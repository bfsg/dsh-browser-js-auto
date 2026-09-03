# Agent Skills

集中存放可复用的 Agent 技能(SKILL)。每项技能是一个自包含的技能包,遵循统一结构与 frontmatter 约定,可被支持该规范的 Agent 运行时直接加载。

## 结构约定

```
skills/
  <skill-name>/
    SKILL.md      # 技能主文档(必需):YAML frontmatter + 触发式 description + 正文
```

frontmatter 字段:`name`(小写连字符)、`x-provider`(来源,如本仓库名)、`x-version`、`description`(以 "Use when…" 描述触发场景,不总结流程)。

## 技能列表

| 技能 | 用途 |
| --- | --- |
| [browser-evaluate-human-actions](skills/browser-evaluate-human-actions/SKILL.md) | 用 browser_evaluate 在页面上下文里模拟真人操作(点搜索框/逐字输入/选联想词/点按钮/下拉选择),含事件序列、helper 与常见坑 |
