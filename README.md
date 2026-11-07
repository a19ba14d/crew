# crew

Claude Code 的多角色执行任务 skill。输入 `/crew <任务>`，调度员按任务难度把活派给三个子代理，每个角色只做自己的事。

| 角色 | 文件 | 只做 |
|---|---|---|
| 调度员 | `skills/crew/SKILL.md` | 派活、传话、判断、汇报，不碰代码 |
| 规划员 | `agents/crew-planner.md` | 读代码，出改动方案和验收标准 |
| 开发员 | `agents/crew-coder.md` | 按方案改代码 |
| 验证员 | `agents/crew-verifier.md` | 跑验收命令、审查改动，给出结论 |

流程是规划 → 开发 → 验证。简单任务跳过规划；验证不通过最多返工 2 轮；全程不提交、不推送。

## 安装

装到用户级目录，所有项目都能用：

```bash
git clone https://github.com/a19ba14d/crew.git && cd crew
mkdir -p ~/.claude/skills ~/.claude/agents
cp -R skills/crew ~/.claude/skills/
cp agents/*.md ~/.claude/agents/
```

只给某个项目用，就把 `skills/` 和 `agents/` 复制到那个项目的 `.claude/` 下。装完新开一个会话。

## 用法

- 会话里：`/crew 给登录接口加手机号格式校验`
- 命令行无人值守：`claude --dangerously-skip-permissions -p "/crew 给登录接口加手机号格式校验"`。这个参数会跳过所有权限检查，执行中不会被拦，只在你信任的项目里用。

## 改默认设置

- 调度员的模型和思考程度：`skills/crew/SKILL.md` 开头的 `model:`（默认 `claude-opus-5-5`）和 `effort:`（默认 `max`）。
- 各角色按难度用的流程、模型和思考程度：`skills/crew/SKILL.md` 里「按难度分配」那张表。
