# crew

Claude Code 的多角色执行任务 skill。输入 `/crew <任务>`，调度员按任务难度把活派给三个子代理，每个角色只做自己的事。另带一个无人值守运行脚本 `bin/crew-run`，实时输出进度和心跳；还有一个给 [Hermes Agent](https://github.com/NousResearch/hermes-agent) 用的 skill，让 Hermes 把编码任务交给 crew（见「给 Hermes 用」）。

| 角色 | 文件 | 只做 |
|---|---|---|
| 调度员 | `skills/crew/SKILL.md` | 派活、传话、判断、汇报，不碰代码 |
| 规划员 | `agents/crew-planner.md` | 读代码，出改动方案和验收标准 |
| 开发员 | `agents/crew-coder.md` | 按方案改代码 |
| 验证员 | `agents/crew-verifier.md` | 跑验收命令、审查改动，给出结论 |

流程是规划 → 开发 → 验证。简单任务跳过规划；验证不通过最多返工 2 轮，第 1 轮把问题清单发回给原来的开发员接着改（它记得自己读过和改过什么），每次改完都新派一个验证员复验，并带上之前各轮的问题清单让它逐条复查。

- 只读任务（调研、排查、解释、评审）只派规划员，全程不改文件；发现的问题写进遗留问题，附上可以直接下发的修复任务。分不清是不是只读的，按只读处理。
- 需求有歧义、要你拍板时，规划员把问题写成选择题（推荐项在前），调度员停下来问你：会话里直接弹出选项问；命令行运行时汇报「待确认」，答复后用 `crew-run --resume` 接着原来的会话跑。有稳妥做法的小事不问，自己定下来写进汇报。
- 本地查不到、又非查清不可的事实（生产数据、线上配置、别的系统里的记录）也按「待确认」停下，但和要你拍板的问题分开列：这类是请启动 crew 的一方去查的，不是让你选的。
- 开始前调度员会记下工作区里已有的改动，告诉每个角色那是你自己的，不要动。
- 除非任务明确要求，全程不暂存、不提交、不推送。

## 分支与发布规范（2026-11-10 起强制执行）

- **只有两个长期分支：`dev`（开发）、`main`（生产）。** 所有开发直接在 `dev` 上提交。除非用户明确要求，禁止新建任何分支（功能/修复/热修/发布分支、worktree 分支、`cc/*`、`codex/*` 等），也不开 PR。远端已有的其他分支是旧流程遗留，不再使用。
- **`dev` 必须始终包含 `main` 的全部提交（领先或持平）。** 不允许 `main` 有而 `dev` 没有的提交。`main` 只能通过合入 `dev` 前进。紧急修复如果直接提交到 `main`，推送后要立刻把 `main` 合回 `dev` 并推送。禁止 force-push `main` 和 `dev`。
- **完成一个阶段性内容就必须提交并推送 `origin/dev`**，前提是本地检查通过，且只提交本任务的文件。不留未提交的改动。
- **发布流程**：推送 `dev` → 部署测试环境 → 测试环境验收 → 把 `dev` 合入 `main` 并推送 → 部署生产 → 生产验收。
- **上生产审批**：
  - 重要修改 / 大型改动（数据库结构或数据迁移、支付与资金、鉴权、架构调整、大范围重构、发布机制变更）：测试验收后必须经用户同意才能上生产。
  - 中小型改动：评估不会影响数据的改动，以及修 bug、修问题，测试环境验收通过后可直接上生产。
  - 紧急止血 / 恶性 bug：可以直接上生产处理，事后补验收、同步 `dev`，并告知用户。
- **本仓库环境**：测试 = 暂无测试环境：以本地检查和本地运行验收代替（按「安装」一节把 `dev` 的 `skills/`、`agents/` 装到本机，在试验项目里跑 `/crew` 或 `bin/crew-run`）；生产 = 没有部署流程：`main` 即正式版本，使用者按「安装」一节克隆 `main` 后复制到 `~/.claude/`（或项目的 `.claude/`）。
- 本节约束的是开发本仓库。README 开头「除非任务明确要求，全程不暂存、不提交、不推送」说的是 crew 在目标项目里执行任务时的行为，不受本节影响。

## 安装

装到用户级目录，所有项目都能用：

```bash
git clone -b main https://github.com/a19ba14d/crew.git && cd crew
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
- 回答待确认的问题：`crew-run --resume <日志目录> "答复"`，见「待确认：答复后接着跑」。
- 也可以直接用 `claude --dangerously-skip-permissions -p "/crew 给登录接口加手机号格式校验"`，但中间看不到进度，要等跑完才输出最终汇报。遇到待确认时，在同一个目录里用 `claude --dangerously-skip-permissions -p -c "/crew 续：答复"` 接着最近的那次会话跑。

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
| `--log-dir 目录` | `~/.crew/runs` | 日志根目录；续跑时默认放在原来那次旁边 |
| `--permission-mode 模式` | 不设 | 不跳过权限检查，改用这个模式，比如 `auto` |
| `--resume 日志目录` | | 回答待确认的问题，接着那次运行的会话跑，任务描述换成你的答复 |
| `-- 参数…` | | `--` 之后的参数原样传给 claude，比如 `-- --model sonnet` |

三个时间设成 0 表示关掉。

每次运行在日志根目录下建一个 `<项目>-<时间>/`：

| 文件 | 内容 |
|---|---|
| `run.log` | 和终端一样的进度日志，最后附完整汇报 |
| `handoffs/` | 调度员和各角色之间每次交接的原文：派活、返工、汇报，按顺序编号，比如 `01-派给开发员.md`、`02-开发员汇报.md` |
| `result.md` | 调度员的最终汇报 |
| `status.json` | 运行中每秒更新：`state`、`current`（谁在做什么）、`verdict`、用时、费用（`cost_usd` 是这次的，`session_cost_usd` 是整个会话累计的，续跑时两者才不同）；待确认时还有 `questions`、`evidence` 和 `resume`（接着跑的命令）。给脚本、监控或 Hermes 读 |
| `questions.json` | 待确认、有问题要你拍板时才有：问题和选项，格式可以直接交给 Hermes 的 `clarify` |
| `evidence.json` | 待确认、有事实要先查清时才有：要查什么、可能的几种结果。由启动 crew 的一方去查，不是给用户选的 |
| `events.jsonl` | claude 的原始事件流，排查问题用 |
| `stderr.log` | claude 的错误输出 |
| `baseline.diff` | 开始前工作区已有改动时才有：已跟踪文件相对 HEAD 的改动，万一被改掉可以对照找回 |

退出码按最终汇报的结果行定，脚本里可以据此判断要不要人工介入。结果行必须是「结果：」后面只跟一个结果词，带了别的字（比如「通过率 0%」「通过（部分）」）一律按不规范处理，不去猜；claude 自己异常退出时，不管汇报里写的是什么都按出错处理。

| 退出码 | 含义 |
|---|---|
| 0 | 通过，或只读任务已回答 |
| 1 | 未通过、无法验证，或待确认。待确认不是失败：答复后用 `--resume` 接着跑，不要重跑 |
| 2 | 运行出错（包括 claude 异常退出、结果行不规范）、超时、找不到 /crew，或这个目录已经有 crew 在跑 |
| 130 | 手动中断（Ctrl-C 或 kill） |

注意：

- 返工时续用开发员要靠会话记录，不要加 `-- --no-session-persistence`。
- 工作区开始前就有没提交的改动时，crew-run 会提示一行，并把改动存成 `baseline.diff`。各角色会被告知不要动这些改动，但最稳妥的还是先提交或 stash 再跑。
- 同一个目录同时只能跑一个 crew-run，第二个会直接退出（退出码 2），免得两个调度员改同一个工作区。
- Ctrl-C、kill 或超时会终止 claude 和它启动的所有进程：先发 TERM，1 秒内没退的发 KILL。收到停止信号时先把 status.json 记为 interrupted 再清理，Hermes 这类进程管理器停后台进程时只给 2 秒。claude 正常结束后，它启动的进程 2 秒内还没退的（比如忘了关的测试服务）也会一并清理，并提示一行。
- Claude Code 的 Bash 工具会把每条命令放进单独的进程组，所以 crew-run 每 3 秒顺着进程树记一次这些进程组。管不到的只有一种：不到 3 秒就结束的命令在后台留下的进程（比如 `nohup 服务 &`），它已经脱离进程树又没被记到，要自己清理。
- 结束行里的金额是 claude 按 API 价格估算的费用；用订阅时只作参考，实际消耗的是订阅额度。

### 待确认：答复后接着跑

需求有歧义时，调度员停下来汇报「待确认」，问题写成选择题，推荐项在前。crew-run 把问题存成 `questions.json`，结尾提示接着跑的命令。下面是一次真实运行，省略了部分进度行（汇报的小标题和结尾的提示行按现在的格式写）：

```
$ crew-run --permission-mode auto "给 total 加会员 9 折和满 100 减 10 两种优惠"
18:30:29 启动   项目 shop，权限模式 auto，日志 ~/.crew/runs/shop-20261109-183029
18:30:34 调度员 模型 claude-opus-5-5
18:30:35 调度员 Bash: git status --short
18:30:48 调度员 → 规划员（sonnet / high）
18:30:54 规划员 Bash: cat price.py test_price.py .gitignore
18:31:11 规划员 → 调度员：难度 简单：只改一个函数，另加几个测试，没有外部依赖，改动 2 处
18:31:42 ♥      已运行 1m12s，调度员在思考（已 30s）
…
18:35:07 规划员 → 调度员：难度 简单：只改一个函数，另加测试。浮点和舍入细节要处理，但范围小，改动 2 处，待确认 2 项

- **结果**：待确认
- **要你拍板**：
  1. **两种优惠能不能叠加？**
     - **叠加（推荐）**：会员打 9 折，同时还能享受满减。推荐的理由是任务原文写的是「加两种优惠」，叠加最贴近字面。
     - **只取更优的一种**：每单只给会员折扣和满减里省得更多的那一个。
     - **互斥，会员只打折**：会员不享受满减，只有非会员能满减。
  2. **满 100 的门槛按什么金额判断？** 只有第 1 题选「叠加」时才需要回答。
     - **折后金额（推荐）**：会员打完折后满 100 才减。推荐的理由是，用户看到的价格就是判断依据。
     - **折前金额**：会员原价满 100 就减，打完折低于 100 也能减。

这两题决定每单收多少钱，属于定价规则，所以停下来请你定。目前还没有改任何文件。

（最后是一段 json 问题数据，crew-run 存成了 questions.json）

18:35:33 结束   待确认，用时 5m03s，$1.23，日志 ~/.crew/runs/shop-20261109-183029
18:35:33 提示   这不是失败，不要重跑：crew 停下来等答复，会话和已经做的改动都留着
18:35:33 提示   有 2 个问题要用户拍板，见 questions.json
18:35:33 提示   答复后接着跑：crew-run --resume /Users/me/.crew/runs/shop-20261109-183029 "答复"
```

答复写在命令最后，比如两题都选推荐项：

```
$ crew-run --resume /Users/me/.crew/runs/shop-20261109-183029 "叠加；满 100 按折后金额算"
18:36:31 续跑   接着 ~/.crew/runs/shop-20261109-183029 的会话，项目 shop，权限模式 auto，日志 ~/.crew/runs/shop-20261109-183631
18:37:28 调度员 收到你的答复：两种优惠叠加，满 100 按折后金额判断，和推荐选项一致。按流程，我把答复补进任务，连同上一版方案重新派规划员定稿，这次不算返工轮次。
18:37:51 调度员 → 规划员（sonnet / medium）
18:38:08 规划员 → 调度员：难度 简单：只改一个函数，另加测试，改动 2 处
18:38:42 调度员 → 开发员（sonnet / low）
18:39:07 开发员 → 调度员：已改动 2 处，拿不准 1 项
18:39:39 调度员 → 验证员（sonnet / medium）
18:40:00 验证员 → 调度员：结论：通过

（这里输出调度员的完整最终汇报，同时存进 result.md）

18:40:48 结束   通过，用时 4m16s，这次 $0.86，会话累计 $2.09，日志 ~/.crew/runs/shop-20261109-183631
```

- 续跑接着原来的 Claude Code 会话：调度员记得之前的方案、开始前的工作区和自己改过的文件，不用从头规划，也不会把自己的改动当成你的。
- 续跑会自己回到原来的项目目录，沿用原来的权限设置和 `--` 之后的参数，日志放在原来那次旁边。
- 续跑的结束行把这次的费用和整个会话累计的分开写：Claude Code 续跑时报的是会话累计费用，crew-run 减掉了之前那几轮的。`status.json` 里分别是 `cost_usd` 和 `session_cost_usd`。
- 会话记录在 Claude Code 里默认保留 30 天，这期间都能接着跑。
- `questions.json` 的格式是 `{"questions": [{"question": "…", "choices": ["推荐项", "…"]}]}`：最多 4 个问题，每题 2–4 个选项，推荐的在第一个；没有选项的题不带 `choices`，可以多选的带 `"multi_select": true`。有候选答案但没有推荐项的题也不带 `choices`，候选答案写在问题里：Hermes 的 `clarify` 总会把第一个选项标成推荐。
- 待确认里还可能有要先查清的事实，比如「生产接口现在返回什么」。它们存在 `evidence.json` 里，格式是 `{"evidence": [{"question": "…", "choices": ["可能的结果", "…"]}]}`，`choices` 不分推荐。结尾的提示会多一句「有 N 件事要先查清，见 evidence.json」。查清以后和拍板的答复写在一起传给 `--resume`。两类分开存，是为了让 Hermes 这样的调用方只把 `questions.json` 拿去问你，`evidence.json` 自己去查。

## 给 Hermes 用

`hermes/crew/SKILL.md` 是给 [Hermes Agent](https://github.com/NousResearch/hermes-agent) 用的 skill，告诉它什么时候把编码任务交给 crew、怎么后台运行、各个退出码怎么处理，以及遇到待确认时怎么用 `clarify` 问你、再用 `crew-run --resume` 接着跑。

安装：

1. 按上面的「安装」把 skill 和 agents 装进 Claude Code，并把 `bin` 加进 PATH。
2. 在 `~/.hermes/config.yaml` 里把仓库的 `hermes` 目录加进外部 skill 目录：

   ```yaml
   skills:
     external_dirs:
       - /path/to/crew/hermes
   ```

   只指到 `hermes` 这一层。指向仓库根目录的话，Claude Code 用的 `skills/crew/SKILL.md` 也会被 Hermes 扫成同名的 skill。
3. 新开一个 Hermes 会话，`hermes skills list` 里能看到 crew 就装好了。

Hermes 这边的做法：

- 用 `terminal` 后台运行（`background=true, notify=true`），结束时收到通知。前台最长 600 秒，不够用。
- 读 `status.json` 和 `result.md` 向你汇报。待确认时把 `questions.json` 交给 `clarify` 问你，`evidence.json` 里的事实它自己去查，都齐了再续跑。它不替你回答问题；问不到你时（超时没答、定时任务里），就把问题和续跑命令汇报出来等你。
- 待确认的退出码是 1，Hermes 的后台任务通知只看退出码，会显示成「后台任务失败」并提示可以重新运行。这不是失败，不要重跑；crew-run 结尾的提示行和 skill 里都写明了这一点。
- 你让 Hermes 停掉 crew 时，crew-run 在 2 秒内连同 claude 启动的进程一起收尾，赶在 Hermes 强杀之前。
- 前提是 Hermes 的 terminal 用 local 后端，crew-run 和项目在同一台机器上。
- Hermes 不把外部 skill 目录当只读：它用 `skill_manage` 改 crew 的 skill 时，会直接改到这个仓库里的文件。改动在 git 里看得到，不想要就 `git restore hermes/` 还原。

## 改默认设置

- 调度员的模型和思考程度：`skills/crew/SKILL.md` 开头的 `model:`（默认 `claude-opus-5-5`）和 `effort:`（默认 `max`）。
- 各角色按难度用的流程、模型和思考程度：`skills/crew/SKILL.md` 里「按难度分配」那张表。
