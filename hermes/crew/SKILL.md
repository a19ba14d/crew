---
name: crew
description: 用 Claude Code 分规划、开发、验证三步无人值守完成编码任务。
version: 1.0.0
author: arvin (@a19ba14d)
platforms: [macos, linux]
metadata:
  hermes:
    tags: [Coding-Agent, Claude, Multi-Agent, Automation]
    category: autonomous-ai-agents
    related_skills: [claude-code]
---

# Crew Skill

crew 在 Claude Code 里由一个调度员带规划员、开发员、验证员三个子代理，按「规划 → 开发 → 验证」无人值守地完成编码任务，验证不通过会自动返工，最后给出带结果词的汇报。它只改工作区，不提交、不推送。遇到需要用户拍板的问题，或者它在本地查不到的事实，会停下交回给你：拍板的你去问用户，查不到的你自己去查，拿到答复后接着原来的会话跑。

## When to Use

- 用户要在本地 git 仓库里修 bug、加功能、改配置或文档，希望改完有人验证，不用盯着过程。
- 用户要调研、排查、解释或评审代码：crew 按只读任务处理，只给结论和修复建议，不改文件。
- 不适合：一两行、改哪里已经说清楚的小改动，或者要和用户来回交互的工作，用 `claude-code` skill 更快。
- 同一个目录同一时间只能跑一个 crew。

## Prerequisites

- Claude Code 已安装并登录。
- crew 已装进 Claude Code：`~/.claude/skills/crew/` 和 `~/.claude/agents/crew-*.md`，或者装在项目的 `.claude/` 下。装法见 crew 仓库的 README。
- crew 仓库的 `bin/` 在 PATH 里，`crew-run` 能直接调用。它只需要 Python 3。
- `terminal` 用 local 后端：crew-run 要和项目在同一台机器上。

## How to Run

一律后台跑，结束时收通知：

```
terminal(command=<下面的命令>, workdir=<项目根目录>, background=true, notify=true)
```

任务描述用带引号的 heredoc 传，反引号、`$`、引号都不会被 shell 处理。结束标记必须顶格：

```bash
crew-run "$(cat <<'TASK'
给登录接口加手机号格式校验，不合法时返回 400
TASK
)"
```

回答待确认的问题时，用 `status.json` 里的 `resume` 命令接着跑，答复同样用 heredoc 传，同样后台跑：

```bash
crew-run --resume /Users/me/.crew/runs/shop-20261109-213907 "$(cat <<'ANSWER'
1. 校验规则：按 11 位大陆手机号
2. 不合法时的提示：沿用现有的 ErrInvalidParam
ANSWER
)"
```

- 不要前台跑：crew 一次常常要几分钟到几十分钟，超过前台上限会被转成后台。
- 不要设 `heartbeat`：每跳一次都要你多跑一轮；crew-run 自己会定时打心跳。
- 默认跳过 Claude Code 的权限检查，执行中不会被拦。用户没说信任这个仓库时，加 `--permission-mode auto`（写在任务描述前面）。
- crew-run 的第一行和结束行都会打印日志目录（「日志 ~/.crew/runs/<项目>-<时间>」），后面的步骤都用这个目录。

## Quick Reference

| 退出码 | 含义 | 接着做 |
|---|---|---|
| 0 | 通过，或只读任务已回答 | 读 `result.md`，向用户汇报 |
| 1，结果是待确认 | crew 停下来等答复，不是失败 | 不要重跑。读 `evidence.json`、`questions.json` → 自己查证、`clarify` 问用户 → `crew-run --resume` |
| 1，未通过或无法验证 | 返工 2 轮还没过，或者缺环境 | 读 `result.md`，照实汇报 |
| 2 | 运行出错、超时、找不到 /crew、结果行不规范，或这个目录已有 crew 在跑 | 读 `run.log` 末尾，照实汇报 |
| 130 | 被停止 | 告诉用户已停止 |

日志目录里的文件：

| 文件 | 内容 |
|---|---|
| `status.json` | `state`、`verdict`、`exit_code`、`current`（谁在做什么）、`questions`、`evidence`、`resume`（接着跑的命令） |
| `result.md` | 调度员的最终汇报 |
| `questions.json` | 待确认、有问题要用户拍板时才有，内容就是 `clarify` 的 `questions` 参数 |
| `evidence.json` | 待确认、有事实要你去查清时才有：`{"evidence": [{"question", "choices"?, "multi_select"?}]}`，`choices` 是可能查到的几种结果，不分推荐。不是给用户选的 |
| `run.log` | 完整进度日志 |

## Procedure

1. `workdir` 用项目根目录。工作区有没提交的改动也能跑，crew 会当成用户的，不去动。
2. 按 How to Run 后台启动，告诉用户已经开始，并给出日志目录。
3. 用户问进度时，用 `process` 看最新输出，或用 `read_file` 读 `status.json` 里的 `current`。
4. 收到结束通知后，用 `read_file` 读日志目录下的 `status.json` 和 `result.md`，按退出码处理。
5. 待确认。这不是失败，不要重跑：重跑会从头再来，还会撞上这次已经做的改动。
   1. 有 `evidence.json` 时先查证。里面是 crew 在本地查不到的事实，比如生产数据、线上配置、别的系统里的记录。用你自己的工具去查，只做只读的查询；怎么查、每种结果意味着什么，写在 `result.md` 的「要先查清」里。这些不要用 `clarify` 做成选择题问用户。你也查不到的（没有权限，或者要用户手里的截图、订单号），才向用户要这份材料，要的是材料，不是让他猜答案。
   2. 有 `questions.json` 时，用 `read_file` 读出来，把里面的 `questions` 原样传给 `clarify`，一次问完。查证的结果会影响某个问题怎么问时，先查证再问。
   3. 都齐了以后，把「问题 → 用户的答复」和「要查的事 → 查到的结果和依据」逐条写成一段话，按 How to Run 里的续跑命令接着跑。续跑接着原来的会话，调度员记得之前的方案和自己改过的文件，不用重新规划。
   4. 两个文件都没有（问题数据不规范）时，从 `result.md` 里把问题整理出来，照上面的分法处理。
6. 向用户汇报：结果、改了哪些文件、怎么验证的、遗留问题。crew 不提交，要不要提交问用户。

## Pitfalls

- 待确认的退出码是 1。Hermes 的后台任务通知只看退出码，会把它显示成「后台任务失败」，还提示可以重新运行。以 `status.json` 的 `verdict` 为准，不要照通知说的重跑；用户看到这条通知来问时，告诉他是在等答复。
- 不要替用户回答 `questions.json` 里的问题。能按稳妥做法定的，crew 已经自己定了，剩下的是产品取舍；只有用户在对话里明确说过「你来定」才自己答。
- `evidence.json` 里的正相反：那是要你查的事实，不要推给用户去选。查到什么就答什么，附上依据；查不到就照实说查不到、缺什么，不要挑一个看起来合理的结果当答案。
- `clarify` 超时、被取消，或者当前环境没有 `clarify`（定时任务、`delegate_task` 派出的子代理）：把问题和 `resume` 命令汇报出来，然后停下。会话保留 30 天，用户之后答复了再续跑。
- 续跑只认原来的日志目录：crew-run 会自己回到原来的项目目录，沿用原来的权限设置。
- 同一个目录不要同时跑两个：第二个会直接退出（退出码 2），等前一个结束或先停掉。
- 停止用 `process` 的 kill：crew-run 会在 2 秒内连同 Claude Code 启动的进程一起停掉，`status.json` 记为 `interrupted`。
- 不要提交或推送 crew 的改动，除非用户要求。

## Verification

- `status.json` 的 `state` 不是 `running`，`exit_code` 和结束通知里的一致。
- 结果是通过时，在项目目录用 `terminal` 跑 `git status --short`，改动的文件和 `result.md` 里的「改动文件」对得上。
- 续跑后，新日志目录的 `status.json` 里 `resumed_from` 指向原来那次。
