---
title: "看板即协议：拆解 Hermes Agent 的 Kanban Swarm 多智能体协作"
date: 2026-11-24T19:03:00+08:00
tags: ["kanban", "multi-agent", "hermes-agent"]
author: "clike-peng"
---

上周的典型现场：三个 agent 并行改同一个仓库，各自在收尾消息里报“已完成，测试全绿”。打开 diff 才发现，两个 agent 都重写了 `config.py`，第三个的迁移脚本基于一个已经被删掉的字段名。日志里一条 error 都没有——它们确实都成功了，只是成功在三个互不相交的假设里。

这不是模型能力问题。三个 agent 各自都干得不差，坏掉的是它们之间那层从未存在的协议：谁记状态、谁判断依赖满足没有、失败后由谁重试。下面拆 Hermes Agent 的 Kanban 模式怎么回答这三个问题，以及它在什么规模下才值得用。

先说版本，因为这块最容易传错：仓库里[没有 v0.16.0 这个 tag](https://github.com/NousResearch/hermes-agent/releases/tag/v0.16.0)（直接 404），写作时的最新稳定版是 [v0.23.4](https://github.com/NousResearch/hermes-agent/releases)，Kanban 的引入版本在 releases 里也查不到。所以本文不绑版本号——绑了反而像在说它是某次一次性发布的特性。

## 一、状态不在对话里，而在一行 SQLite 里

Kanban 最容易被误解的一点：它不是进程内 swarm。每个任务是 `~/.hermes/kanban.db` 里的一行记录，每个 worker 是一个独立的 OS 进程，有自己的身份和 session（[官方文档](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban)）。多板隔离也很土：每板一个 SQLite 文件，跨板 link 直接禁止。

为什么重要：状态留在进程内，worker 崩溃就等于状态蒸发，你只能去日志里考古；状态落进数据库行，崩溃只是一个可重放的中断。

推动这套状态机的不是模型，而是一个叫 dispatcher 的长驻轮询循环。它默认每 60 秒跑一轮（`kanban.dispatch_interval_seconds`），每轮固定做四件事：回收过期 claim、回收已崩溃的 worker、把父卡已完成的子卡提升为 ready、原子 claim 一张卡并 spawn 指定 profile（[文档源码](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/website/docs/user-guide/features/kanban.md)）。它默认跑在 gateway 进程内（`dispatch_in_gateway: true`），没有独立服务要运维，代价是也没法单独扩容。

依赖用“父卡闸门”表达：子卡在父卡 done 之前停在 todo，父卡完成后自动提升。同一时刻，一张卡只能被一个 worker claim。这条约束替代了大部分分布式锁，也让“两个 agent 并发写同一个文件”从设计上就难得发生。

workspace 分三型：`scratch`（完成即删）、`dir:<绝对路径>`（保留）、`worktree`（保留）。注意 `dir:` 只接受绝对路径，相对路径在 dispatch 阶段就被拒绝——文档明说这是避免相对 dispatcher 的当前目录解析，属于 confused-deputy 风险。值得抄进自己的系统：任何让调度器解析相对路径的接口，最后都会在目录变化时静默指错。

下面按文档语义复现一轮 tick（不是 Hermes 源码），把“慢但活着”和“真的死了”跑一遍：

```python
"""按官方文档语义复现 Kanban dispatcher 的一轮 tick（玩具实现，不是 Hermes 源码）。
默认值：60s 轮询 / 4h stale 超时 / 1h 心跳窗口 / failure_limit=2。
运行：python dispatcher_tick.py
"""
from dataclasses import dataclass, field

POLL = 60                      # kanban.dispatch_interval_seconds
STALE_TIMEOUT = 4 * 3600       # kanban.dispatch_stale_timeout_seconds
HEARTBEAT_WINDOW = 3600        # 最近 1h 内必须有心跳
FAILURE_LIMIT = 2              # kanban.failure_limit


@dataclass
class Card:
    id: str
    parents: list = field(default_factory=list)
    status: str = "todo"          # todo / ready / running / blocked / done
    claimed_at: int = 0
    heartbeat_at: int = 0
    failures: int = 0


def is_stale(card, now):
    """两个条件必须同时成立：跑太久 且 久未心跳。"""
    if card.status != "running":
        return False
    return (now - card.claimed_at > STALE_TIMEOUT
            and now - card.heartbeat_at > HEARTBEAT_WINDOW)


def tick(cards, now, spawn_ok=True):
    by_id = {c.id: c for c in cards}

    for c in cards:                                  # 1) 回收已死 worker
        if is_stale(c, now):
            c.status, c.claimed_at, c.heartbeat_at = "ready", 0, 0
            print(f"[stale]   {c.id} -> ready（不计入 failures）")

    for c in cards:                                  # 2) 父卡闸门
        if c.status == "todo" and all(by_id[p].status == "done" for p in c.parents):
            c.status = "ready"
            print(f"[promote] {c.id} todo -> ready")

    for c in cards:                                  # 3) 一次一 claim
        if c.status != "ready":
            continue
        if not spawn_ok:
            c.failures += 1
            if c.failures >= FAILURE_LIMIT:
                c.status = "blocked"
                print(f"[block]   {c.id} 连续 {c.failures} 次 spawn 失败 -> blocked")
            continue
        c.status, c.claimed_at, c.heartbeat_at = "running", now, now
        print(f"[spawn]   {c.id} 被 claim，spawn profile=writer")


if __name__ == "__main__":
    now = 4 * 3600 + 120           # 距首次 claim 已 4h02m

    cards = [
        Card("t1", status="done"),
        Card("t2", parents=["t1"], status="running",
             claimed_at=0, heartbeat_at=now - 90),   # 跑得久，但心跳新鲜
        Card("t3", parents=["t2"]),
    ]
    print("--- 第一轮：t2 心跳新鲜，不该被回收 ---")
    tick(cards, now)
    print("状态:", {c.id: c.status for c in cards})

    print("\n--- 第二轮：t2 静默 2 小时，两个条件同时成立 ---")
    cards[1].heartbeat_at = now - 2 * 3600
    tick(cards, now)
    print("状态:", {c.id: c.status for c in cards})

    print("\n--- 第三轮：spawn 反复失败，第 2 次熔断 ---")
    cards[1].status = "ready"
    for _ in range(2):
        tick(cards, now, spawn_ok=False)
    print("状态:", {c.id: c.status for c in cards})
```

跑一遍会看到 t2 在第二轮被回收、又在同一轮被重新 claim；第三轮因连续两次 spawn 失败熔断到 blocked。回收不计失败、spawn 失败才计，这两个计数器分开是最容易被忽略的一笔。

## 二、“我做了”必须能被机器验证

多智能体最贵的 bug 不是算错，是自述。worker 说“文件我写好了”，下游拿到的是一个不存在的路径。Kanban 的做法是把交接变成结构化字段：`kanban_complete(summary, metadata)`，下游子卡读到的是父卡最近一次成功 run 的 summary 加 metadata，重试的 worker 读自己这张卡既有的尝试记录。

为什么重要：共享上下文的成本随 agent 数量平方增长，handoff 的成本只随依赖边线性增长；更关键的是，结构化摘要可以 diff、可以审计，出错时能定位到“从哪一跳开始错”。

光有字段还不够，字段得能被验证。三个机制：

`created_cards` 里每个 id 都会被 kernel 校验真实存在且由本 worker 创建，幻影 id 直接让 completion 失败——“我建了卡片”从声明变成断言。`artifacts` 声明的文件会在 scratch workspace 清理前复制进持久附件存储（单文件上限 25 MB），声明了但文件不存在，任务会留在 in-flight 等你修。`metadata` 则是约定而非 schema：文档只给推荐键名（如 changed_files / verification / dependencies / residual_risk），并明确禁止放密钥和原始日志。

第二条有个现实坑：用 Docker 或 Modal 后端时，attachments 目录要手动挂进沙箱，否则“我附了文件”在容器里是个空目录。

```python
"""校验一次 kanban_complete 的交接：幻影 card id 和缺失 artifact 都必须让 completion 失败。
运行：python verify_handoff.py
"""
import pathlib
import tempfile

TASKS = {"t_a1", "t_b2"}          # 板上真实存在、且由本 worker 创建的卡


def verify(payload, workspace):
    problems = []
    for cid in payload.get("created_cards") or []:
        if cid not in TASKS:
            problems.append(f"phantom card id: {cid}")
    for rel in payload.get("artifacts") or []:
        if not (pathlib.Path(workspace) / rel).exists():
            problems.append(f"artifact 不存在: {rel}")
    if not (payload.get("summary") or "").strip():
        problems.append("summary 为空：下游读不到任何东西")
    return problems


if __name__ == "__main__":
    ws = tempfile.mkdtemp()
    pathlib.Path(ws, "03-draft.md").write_text("draft", encoding="utf-8")

    good = {"summary": "草稿完成",
            "created_cards": ["t_a1"],
            "artifacts": ["03-draft.md"]}
    bad = {"summary": "草稿完成",
           "created_cards": ["t_a1", "t_x9"],
           "artifacts": ["03-draft.md", "04-review.md"]}

    for name, payload in (("good", good), ("bad", bad)):
        problems = verify(payload, ws)
        print(f"{name}: {'PASS' if not problems else 'FAIL'}")
        for p in problems:
            print("  -", p)
```

还有一条写给拆解者的规则，比任何分工模板都实用：并列的两张卡不得各自发明文件格式。要么在父卡里预先定死（比如 newline-delimited JSON 加一个 `version` 字段），要么就等着合并时写转换层。所以拆解者的核心产出不是分工表，是替所有 worker 提前做掉共享决策——命名、schema、文件布局。

## 三、人的位置是阻塞点，不是审批流

只要 worker 会卡住，就得有人处理。多数框架只给一个笼统的 failed，然后人的收件箱被淹。Kanban 把 block 按原因分成四类：`dependency`（等别的卡）、`needs_input`（要人决定）、`capability`（硬墙：没权限、没凭据）、`transient`（可能自己好的抖动）。

为什么重要：分类的意义是决定“谁叫醒谁”。没有父卡的 `dependency` 会被改记成 `needs_input` 并带上 `rekind_reason: no_open_parent`，假等待被系统主动识别出来；`transient` 则不该去敲人。人的时间只花在真正需要判断的地方。

防抖也是机制而不是判断：同一原因反复 block，达到 `BLOCK_RECURRENCE_LIMIT`（默认 2）就自动升级进 `triage`，而且是确定性的数据库守卫——任务正文里写“请勿升级”也没用，模型也没机会给自己开后门。

回收侧同样有区分度。stale 判定要求运行超过 `dispatch_stale_timeout_seconds`（默认 4 小时）并且最近 1 小时没有心跳，两个条件同时成立才 SIGTERM 回 ready，且这次回收不计入失败计数——这是 dispatcher 侧的存在性检测，不算 worker 的过错。连续 spawn 失败是另一回事，`failure_limit` 默认 2 次就 block，防止对着一个不存在的 profile 反复冲撞。

异常退出语义也补齐了：普通失败 exit 1，限流、5xx、超时 exit 75（`EX_TEMPFAIL`），配置性终错 exit 78（`EX_CONFIG`）；provider 凭据被吊销这类终错首次就 sticky block，不耗重试预算；worker 正常退出却没调终止性 board 工具，走独立的协议违规预算（默认连续 3 次）。

review 则是同一张卡内的评审，不是 block：`request_review` 把卡移进 review 列，评审者用 `request_changes` 打回原实现者，不消耗 block 循环计数——否则一次正常的两轮评审就会被误判成卡循环而升级 triage。

最后是 orchestrator 的三条约束：不做实现、先发现真实存在的 profile、替所有 worker 做共享决策。文档还建议把它的工具集限制到 `kanban` / `gateway` / `memory`，让它物理上无法动手。这是少见的“用权限而非提示词约束 agent”，比在 prompt 里写十遍“别自己改代码”可靠。

## 四、分界线与账单

该用：任务之间有真实依赖、单个子任务要跑几十分钟以上、失败需要归因到具体一跳、或者中途确实需要人介入判断。这些条件下，60 秒的调度粒度相对任务时长可以忽略。

不该用：一张卡一个人十分钟能干完的事，或者子任务之间需要高频共享中间状态（比如一起调同一个算法）。为后者上板，你只是把函数调用换成了数据库往返。

还有一条硬边界：Kanban 是 single-host by design，没有跨主机协调原语，崩溃检测假设 PID 就在本机。想跨机器共享一块板，你得自己造。

三笔成本。延迟：状态推进粒度是 60 秒一轮，加上 worker 启动与首 token 时间，交接越细、串行跳数越多。token：官方文档没有任何量化数据（单卡耗时、回收次数、并发上限都没有），别信“省 X%”；能确定的是上下文不再随 agent 数增长，但每次交接都要重装卡片正文与父卡摘要。人工值守：人从“全程盯”变成“只在 blocked 卡上出现”是净收益，但瓶颈也从最慢的 agent 转移到最慢的人。

一条安全提醒，官方写在 security model 里：dashboard 的插件 REST 面按设计不鉴权，只有绑 localhost 才安全。用 `--host 0.0.0.0` 把看板暴露出去，等于公开任务正文、评论和文件路径，而且别人能改派和归档。

和 CrewAI、LangGraph 的差别不在快慢，在状态放哪：[CrewAI 的 hierarchical process](https://docs.crewai.com/en/concepts/processes) 靠 `manager_llm` / `manager_agent` 在同一个 crew 内分派与验收，任务不预先绑定 agent；[LangGraph](https://docs.langchain.com/oss/python/langgraph/workflows-agents) 把流程表达成 state graph，状态存在图的 checkpoint 里。Kanban 的赌注更朴素：把状态放进一个所有人都能读、可以 diff 的数据库，比放进任何模型的工作记忆都可靠。

回到开头那三个 agent。它们缺的不是更强的模型，是一张谁都能查、失败能重放的板。
