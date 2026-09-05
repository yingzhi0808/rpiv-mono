# 本地 fork 说明（单仓库形态）

这是 `rpiv-mono`  monorepo 里 `packages/rpiv-ask-user-question` 的本地 fork，上游是 <https://github.com/juicesharp/rpiv-mono>（MIT）。2026-09-05 之前它是一份独立的 vendor 副本（npm 包基线 + vendor 分支同步），当天合并进 monorepo 单仓库，本文是合并后的形态。旧仓库的分支与标签已收进本仓库 `refs/vendor/` 命名空间：`refs/vendor/heads/main` 是旧运行时线终点，`refs/vendor/heads/upstream` 是 vendor 分支，`refs/vendor/tags/upstream/*` 是各基线标签；旧同步流程的文档在这些提交的版本里可查。

fork 的唯一目的：让外部程序能直接提交问卷答案，不必模拟按键。herdr 里 pi 作为执行者跑在独立 pane 中，守望方原先只能靠 `pane send-keys` 发方向键和回车，按错不报错、残缺答案会被当成真实用户答案送给模型。

## 一个仓库，两个分支，各司其职

仓库在 `~/workspace/rpiv-mono`，远端 `origin` 是上游、`fork` 是自己的 GitHub fork（SSH；HTTPS 推送会被 OAuth 的 workflow scope 拦住）。

| 分支 | 内容 | 检出位置 |
| --- | --- | --- |
| `runtime` | 上游 main + 本 fork 改动 + 本文档。**pi 实际加载的就是它** | 固定 worktree `~/workspace/rpiv-runtime`，`~/.pi/agent/settings.json` 指向其中的 `packages/rpiv-ask-user-question` |
| `feat/programmatic-answer-event` | 上游 main + 本 fork 改动（不含本文档），推上游用 | 主 checkout，PR #207 的 head |

两条分支的功能代码保持逐行一致；差别只有本文档在不在。不再给 `package.json` 加 `-herdr.1` 版本号后缀——上游每次发版都改版本号行，后缀会让每次同步都冲突一次；fork 身份由分支名和本文档承载。

**worktree 纪律：`~/workspace/rpiv-runtime` 永远停在 `runtime` 分支。** 切走分支，pi 的魔改就静默消失（问卷还在，程序化应答通道没了），而 pi 启动不会报错。需要在这个 worktree 里看别的分支时，看完切回 `runtime` 再走。

**绝不在 runtime worktree 里跑 monorepo 测试套件（含 commit/push 触发的 hook）。** 2026-09-05 实测：coverage 全量跑里的 git fixture 测试会在 worktree 上下文里把共享 `.git` 当 fixture 重建——`core.bare` 被翻成 true、`runtime` 分支被移到 fixture 的 seed 历史，主 checkout 当场变裸库。对象库与其余引用未损，靠 worktree 的 HEAD reflog 找回提交、`core.bare` 翻回 false、worktree 内 `reset --hard` 复原。根因未定位到上游具体哪一行，只确定了触发条件是「worktree 内跑套件」：同一套件在主 checkout 跑过三次均无此事。跑测试去主 checkout 跑。

## 改了什么

五个文件，除文档外全是新增，没有删改上游既有逻辑：

| 文件 | 改动 |
| --- | --- |
| `events.ts` | 新增入站事件 `rpiv:ask-user:answer`（`ASK_USER_ANSWER_EVENT`）与回执事件 `rpiv:ask-user:answer-result`，以及 `ExternalAnswerEntry` 等 payload 类型 |
| `state/questionnaire-session.ts` | `QuestionnaireSession` 新增公开方法 `answerExternal(entries)`，把「题号 + 选项号」补全成 `QuestionAnswer[]` 后调既有的 `this.done(...)` |
| `ask-user-question.ts` | 扩展作用域新增 `activeSession` 引用（经 `makeSessionFactory` 的 config 传入，在 `sessionRef.current = session` 旁边同步赋值）并注册入站事件监听；`finally` 里与 blocked 事件同处清空 |
| `index.ts` | 把新增的常量与类型加进 `./events` 的公开导出 |
| `docs/tool-schema.md` | 事件契约一节补两个新事件，以及 2.1.0 就有但漏写的 `rpiv:ask-user:blocked` |

设计约束（改的时候别破坏）：

- **全量提交**。每道题都必须给答案，任一题缺失或索引越界就整体拒绝并回 `ok:false` + 原因。部分应用的答案送到模型那里，与真实用户答案无法区分，比拒绝更糟。
- **回执必发**。接受与否都 emit `rpiv:ask-user:answer-result`，因为事件 payload 按上游的稳定性约定必须能 JSON 序列化，不能塞回调。
- **只覆盖 TUI 路径**。RPC/ACP 主机走 `runRpcQuestionnaire`，不经过 `QuestionnaireSession`，那时 `activeSession` 为 null，入站事件会被拒绝并说明原因。herdr 场景永远是 TUI。

## 跟上游同步

vendor 分支、npm pack、rsync 那一套已退休。同步就是两次 rebase：

```bash
cd ~/workspace/rpiv-mono
git fetch origin

# PR 分支（主 checkout）
git checkout feat/programmatic-answer-event
git rebase origin/main
git push --force-with-lease fork feat/programmatic-answer-event

# runtime 分支（worktree 里做，别动主 checkout 的分支）
git -C ~/workspace/rpiv-runtime rebase origin/main
```

冲突基本必然发生，但基本都是良性的：两边往同一文件末尾追加内容、零重叠，删掉冲突标记两段都留即可。真正需要动脑的是上游重构 `QuestionnaireSession` 的提交路径或 `execute` 的 session 生命周期——2026-09-05 同步 2.9.0 时真实发生过一次：上游把内联 session 工厂提取成 `makeSessionFactory`，解法是给它的 config 加 `activeSession: SessionRef` 字段，在 `sessionRef.current = session` 旁边同步赋值。判断依据是上面三条设计约束。

rebase 完成后必须重跑[验证清单](#验证清单)，全项通过才算同步成功。

## 依赖

worktree 根部 `npm ci`（monorepo 是 npm workspaces）。包内导入经根部 `node_modules` 解析：`@juicesharp/rpiv-config` 指向 workspace 源码，`@earendil-works/pi-tui` 是仓库锁定的版本（当前 0.80.6），与 pi 进程自带的版本（当前 0.84.x）不同——包里用到的只是按键匹配的纯函数与类型，2026-09-05 单仓库形态复测通过。真撞到按键行为诡异，这里是第一嫌疑。

## 验证清单

外层还需要 `~/.pi/agent/extensions/herdr-blocked-bridge.ts` 把 unix socket 桥到事件，以及客户端 `~/.local/bin/herdr-ask`（细节见 hello-world 仓库的 `skills/personal/spawn-pane-agent/SKILL.md`）。在 herdr pane 里起 `pi -a`，让它调 `ask_user_question`，然后 `herdr-ask <pane> answer '…'`，逐项核对：

| 场景 | 期望 |
| --- | --- |
| 单选 `optionIndexes: [1]` | `ok:true`，模型收到第 2 个选项 |
| 多选 `optionIndexes: [0,2]` + `notes` | `ok:true`，模型收到两个标签与 notes |
| 自定义 `text` | `ok:true`，等价于 `Type something.` 行 |
| 索引越界 | `ok:false` 带范围说明，问卷仍停在原处可重提 |
| 单选给多个索引 | `ok:false` |
| 缺题或空 `answers` | `ok:false` 指出缺哪一题 |
| 无活跃问卷时提交 | `ok:false`，`no questionnaire is awaiting input` |
| 进程强杀后残留的死 socket | 客户端报「进程已经没了」，不挂起 |

以上各项在 2026-08-11（vendor 副本基线 2.4.0）、2026-09-05（vendor 副本基线 2.9.0）、2026-09-05（单仓库形态）三次全项实测通过。

想彻底回到上游版本：把 `settings.json` 里那条 packages 改回 `npm:@juicesharp/rpiv-ask-user-question`，删掉 runtime worktree 即可。

值得继续给上游推：PR #207 已开。作者在 `events.ts` 的稳定性约定里明确写了「listeners forward them across process or network boundaries」，跨进程驱动本就在他的设计视野内。
