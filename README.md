# crew

Claude Code 的多角色执行任务 skill。输入 `/crew <任务>`，调度员按任务难度把活派给三个子代理，每个角色只做自己的事。另带一个无人值守运行脚本 `bin/crew-run`，实时输出进度和心跳。

| 角色 | 文件 | 只做 |
|---|---|---|
| 调度员 | `skills/crew/SKILL.md` | 派活、传话、判断、汇报，不碰代码 |
| 规划员 | `agents/crew-planner.md` | 读代码，出改动方案和验收标准 |
| 开发员 | `agents/crew-coder.md` | 按方案改代码 |
| 验证员 | `agents/crew-verifier.md` | 跑验收命令、审查改动，给出结论 |

流程是规划 → 开发 → 验证。简单任务跳过规划；验证不通过最多返工 2 轮，第 1 轮把问题清单发回给原来的开发员接着改（它记得自己读过和改过什么），每次改完都新派一个验证员复验，并带上之前各轮的问题清单让它逐条复查。

- 只读任务（调研、排查、解释、评审）只派规划员，全程不改文件；发现的问题写进遗留问题，附上可以直接下发的修复任务。分不清是不是只读的，按只读处理。
- 开始前调度员会记下工作区里已有的改动，告诉每个角色那是你自己的，不要动。
- 除非任务明确要求，全程不暂存、不提交、不推送。

## 安装

装到用户级目录，所有项目都能用：

```bash
git clone https://github.com/a19ba14d/crew.git && cd crew
mkdir -p ~/.claude/skills ~/.claude/agents
cp -R skills/crew ~/.claude/skills/
cp agents/*.md ~/.claude/agents/
```

只给某个项目用，就把 `skills/` 和 `agents/` 复制到那个项目的 `.claude/` 下。装完新开一个会话。

要用 `crew-run`，在仓库目录里再执行一次，把 `bin` 加进 PATH。它只需要 Python 3，不用装别的依赖：

```bash
echo "export PATH=\"$(pwd)/bin:\$PATH\"" >> ~/.zshrc && source ~/.zshrc
```

## 用法

- 会话里：`/crew 给登录接口加手机号格式校验`
- 命令行无人值守：`crew-run "给登录接口加手机号格式校验"`，实时显示每个角色在做什么，见下一节。
- 也可以直接用 `claude --dangerously-skip-permissions -p "/crew 给登录接口加手机号格式校验"`，但中间看不到进度，要等跑完才输出最终汇报。

命令行这两种都会跳过所有权限检查，执行中不会被拦，只在你信任的项目里用。

## crew-run：实时日志和心跳

`claude -p` 默认跑完才输出，中间可能几分钟甚至几十分钟没有动静。`crew-run` 在外面包一层：读 claude 的实时事件流，翻译成一行一行的进度；终端安静超过 30 秒就打一行心跳，说明谁在做什么、做了多久。

```
$ crew-run --permission-mode auto "把 calc.py 里 add 函数的 return a - b 改成 return a + b，改完 python3 -m unittest 要全部通过"
21:29:16 启动   项目 calc-demo，权限模式 auto，日志 ~/.crew/runs/calc-demo-20261107-212916
21:29:33 调度员 模型 claude-opus-5-5
21:29:33 调度员 判断为简单任务：改哪里、改成什么都写清楚了，只改一处。跳过规划，先派开发员（sonnet / low）。
21:29:36 调度员 → 开发员（sonnet / low）
21:29:39 开发员 Edit: calc.py
21:29:41 开发员 Bash: python3 -m py_compile calc.py && echo ok
21:29:44 开发员 → 调度员：已改动 1 处
21:29:51 调度员 开发完成：开发员改了 calc.py 里的一行，没有计划外改动。接着派验证员（sonnet / low）做全量验证。
21:29:56 调度员 → 验证员（sonnet / low）
21:29:59 验证员 Bash: git diff && git status --short && git ls-files && python3 -m unittest 2>&1 | tail -15
21:30:03 验证员 → 调度员：结论：不通过，问题 1 个
21:30:34 ♥      已运行 1m17s，调度员在思考（已 30s）
21:30:48 调度员 → 开发员（续用上一轮的开发员）
21:30:50 开发员 Bash: cat calc.py test_calc.py
21:30:51 调度员 问题清单已经发给开发员（续用原来的 sonnet / low），等它修完汇报，再新派验证员全量复验。
21:30:53 开发员 Bash: sed -i '' 's/total = 1/total = 0/' calc.py && python3 -m unittest 2>&1 | tail -3
21:30:56 开发员 → 调度员：已改动 1 处
21:31:01 调度员 返工完成：开发员查到 add_all 的累加初值是 total = 1，已改成 total = 0，并说测试全部通过。现在新派一个验证员全量复验（sonnet / low）。
21:31:09 调度员 → 验证员（sonnet / low）
21:31:13 验证员 Bash: git diff && git status --short && python3 -m unittest 2>&1 | tail -5; cat test*.py | head -30
21:31:16 验证员 → 调度员：结论：通过

（这里输出调度员的完整最终汇报，同时存进 result.md）

21:31:29 结束   通过，用时 2m12s，$0.56，日志 ~/.crew/runs/calc-demo-20261107-212916
```

| 选项 | 默认 | 作用 |
|---|---|---|
| `--heartbeat 秒` | 30 | 终端多久没有输出就打一行心跳 |
| `--stall 分钟` | 10 | 多久没有任何事件就提示可能卡住，只提示不终止 |
| `--timeout 分钟` | 60 | 总时长上限，超过就终止 claude 和它启动的所有进程 |
| `--log-dir 目录` | `~/.crew/runs` | 日志根目录 |
| `--permission-mode 模式` | 不设 | 不跳过权限检查，改用这个模式，比如 `auto` |
| `-- 参数…` | | `--` 之后的参数原样传给 claude，比如 `-- --model sonnet` |

三个时间设成 0 表示关掉。

每次运行在日志根目录下建一个 `<项目>-<时间>/`：

| 文件 | 内容 |
|---|---|
| `run.log` | 和终端一样的进度日志，最后附完整汇报 |
| `handoffs/` | 调度员和各角色之间每次交接的原文：派活、返工、汇报，按顺序编号，比如 `01-派给开发员.md`、`02-开发员汇报.md` |
| `result.md` | 调度员的最终汇报 |
| `status.json` | 运行中每秒更新：`state`、`current`（谁在做什么）、`verdict`、用时、费用，给脚本或监控读 |
| `events.jsonl` | claude 的原始事件流，排查问题用 |
| `stderr.log` | claude 的错误输出 |
| `baseline.diff` | 开始前工作区已有改动时才有：已跟踪文件相对 HEAD 的改动，万一被改掉可以对照找回 |

退出码按最终汇报的结果行定，脚本里可以据此判断要不要人工介入。结果行必须是「结果：」后面只跟一个结果词，带了别的字（比如「通过率 0%」「通过（部分）」）一律按不规范处理，不去猜；claude 自己异常退出时，不管汇报里写的是什么都按出错处理。

| 退出码 | 含义 |
|---|---|
| 0 | 通过，或只读任务已回答 |
| 1 | 未通过、无法验证，或有问题等你确认（把答复补进任务描述再跑一次） |
| 2 | 运行出错（包括 claude 异常退出、结果行不规范）、超时，或找不到 /crew |
| 130 | 手动中断（Ctrl-C 或 kill） |

注意：

- 返工时续用开发员要靠会话记录，不要加 `-- --no-session-persistence`。
- 工作区开始前就有没提交的改动时，crew-run 会提示一行，并把改动存成 `baseline.diff`。各角色会被告知不要动这些改动，但最稳妥的还是先提交或 stash 再跑。
- Ctrl-C、kill 或超时会终止 claude 和它启动的所有进程：先发 TERM，10 秒内没退的发 KILL。claude 正常结束后，它启动的进程 2 秒内还没退的（比如忘了关的测试服务）也会一并清理，并提示一行。
- Claude Code 的 Bash 工具会把每条命令放进单独的进程组，所以 crew-run 每 3 秒顺着进程树记一次这些进程组。管不到的只有一种：不到 3 秒就结束的命令在后台留下的进程（比如 `nohup 服务 &`），它已经脱离进程树又没被记到，要自己清理。
- 结束行里的金额是 claude 按 API 价格估算的费用；用订阅时只作参考，实际消耗的是订阅额度。

## 改默认设置

- 调度员的模型和思考程度：`skills/crew/SKILL.md` 开头的 `model:`（默认 `claude-opus-5-5`）和 `effort:`（默认 `max`）。
- 各角色按难度用的流程、模型和思考程度：`skills/crew/SKILL.md` 里「按难度分配」那张表。
