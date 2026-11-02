# 006 部署

用这份指南安装、配置和运行 dsh-superagent。它涵盖账号、启动检查、升级和排障，建立在 [001 领域模型](001-domain_zh.md) 到 [005 面板](005-panel_zh.md) 之上。

读的时候可以把它分成三截。第一截是机器上长期跑着的东西：专用配置档、智能体账号，以及可选的聊天机器人。第二截是人要亲手做的准备：装依赖、写命令行包装、登录、复制机器配置。第三截是面板里保存的来源、策略、执行和提示词。这些保存走本机的管理接口；保存被拒绝时，响应正文会指出该去文末排障表的哪一行。升级和排障放在后面，因为它们用到前面已经出现过的状态目录、健康码和同一个接口。

## 跑起来的是什么

- **一份专用的 DeepSeek Harness 配置档**，本指南里叫 `superagent`。从网页配置档模板创建它。它加载这个插件包，并在 `127.0.0.1` 上提供面板。把它和你自己工作的那份配置档分开：插件会改整个进程的 `PATH` 和 git 设置，它的进程也需要独立地重启和升级。
- **一个智能体的 GitHub 账号。** 智能体用这个账号评论、推送分支、开拉取请求，经过一个只以该账号登录的 GitHub 命令行。
- **可选：** 一个飞书机器人，用来接聊天里的请求。

插件配置持有状态目录和机器限制。其余一切用面板：来源管仓库和账号，策略管允许名单，执行管工作者目录和命令。

### 权限与隔离

插件包定义 `dsh-superagent-worker` 给执行器，`dsh-superagent-router` 给主 Session。它们出现在 DeepSeek Harness 的预设列表里，但只打算给 dsh-superagent 用。不要给个人会话选它们，也不要停用它们的配置行。应用需要它们来创建和恢复自己的会话。

在网页界面里打开一个空白的受管会话，可能选中配置档的默认预设。dsh-superagent 会在第一轮之前恢复它按角色而定的预设。

两套预设都以 `danger-full-access`、以操作者的操作系统用户身份运行，并读取已订阅线程的消息。账号包装默认选用智能体的凭证；它们并不隔离进程。工作者可以按路径执行程序，并读取操作者的主目录。要隔离的话，用一个只含智能体凭证的专用操作系统账号。

## 前提

- Node.js 22.19+ 或 24+、pnpm、git 2.32+、Python 3.8+ 和 tmux
- GitHub 命令行（`gh`）2.48 或更高。来源的命令行字段必须指向 `gh` 或 `lark-cli` 的包装；宿主依赖它们的子命令和输出格式。
- github.com 上的一个仓库；不支持 GitHub Enterprise Server。要为每张工单造一份检出，就留一份本地克隆，准备命令可以从它添加工作树。那个仓库用 Git LFS 的话，安装 `git-lfs`。
- 给 DeepSeek Harness 配好的一个模型提供方。评估或执行设置没有另点名时，dsh-superagent 用配置档的默认模型。
- 若用飞书：`lark-cli`（以 1.0.90 测过），以及一个带机器人的飞书或 Lark 应用

## 账号

### 智能体的 GitHub 账号

1. 为智能体创建一个 GitHub 用户账号，并给它对该仓库的写权限。GitHub 应用不能当智能体，因为 dsh-superagent 读的是这个账号的通知。
2. 给它一个只用它自己凭证的 GitHub 命令行：

   ```sh
   mkdir -p ~/.local/bin
   cat > ~/.local/bin/gh-agent <<'EOF'
   #!/bin/sh
   GH_CONFIG_DIR="$HOME/.config/gh-agent" exec gh "$@"
   EOF
   chmod +x ~/.local/bin/gh-agent
   gh-agent auth login
   gh-agent api user --jq .login
   ```

   以智能体账号登录；最后一条命令打印它的登录名。dsh-superagent 也用同样的方式读登录名。

### 飞书机器人（可选）

1. 创建一个带机器人能力的自建应用。授予它接收消息事件、读写聊天消息、读取群成员的权限，然后发布应用版本。
2. 给机器人一份它自己的 `lark-cli` 配置，限制在机器人身份上，再加一个包装：

   ```sh
   LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent" lark-cli config init --app-id <cli_…> --app-secret-stdin
   LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent" lark-cli config strict-mode bot --global
   cat > ~/.local/bin/lark-bot <<'EOF'
   #!/bin/sh
   export LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent"
   exec lark-cli "$@"
   EOF
   chmod +x ~/.local/bin/lark-bot
   ```

   第一条命令从标准输入读应用密钥。用一个单独的配置目录：`lark-cli` 每个应用标识只允许一份配置档，你的个人配置里可能已经有该应用的用户登录。

   宿主拒绝一个能以用户身份行动的命令行。它从机器人配置里读应用标识、飞书或 Lark 的品牌，以及机器人的 `open_id`。

   两份配置共用应用密钥的钥匙串条目。要只去掉机器人配置，删掉它的目录；不要跑 `lark-cli config remove`，那会影响到共用的密钥。

## 安装

在一台还没有 `superagent` 配置档的机器上，把这块跑一次：

```sh
npm install -g @deepseek-ai/dsh@0.2.1-rc.3
git clone <this repository> dsh-superagent && cd dsh-superagent
pnpm install && pnpm build
dsh superagent --from-default-profile web --dump-config > /dev/null
dsh plugin --profile superagent add "link:$PWD"
```

上面钉住的版本是公开的；其他版本必须满足 `package.json` 里的 `engines.dsh`。

第四条命令从 DeepSeek Harness 的网页模板创建 `~/.dsh/profiles/superagent`。第五条安装这个插件包。设置 `DSH_HOME` 可以把默认的 `~/.dsh` 目录挪走。

创建配置档的命令只用于全新安装。配置档已经存在时，那条命令会替换它，并删掉它的 `cordis.patch.yml`，包括机器配置。升级时不要用它。

## 机器配置

把 [examples/profile.cordis.patch.yml](../examples/profile.cordis.patch.yml) 复制到 `~/.dsh/profiles/superagent/cordis.patch.yml`，替换那个文件，并换掉每一个 `<placeholder>`。若用飞书，还要在那里启用 `dsh-superagent-feishu` 那一条。`webserver` 那一条设置面板的端口。

| 键 | 是否必需 | 含义 |
| --- | --- | --- |
| `stateDir` | 是 | 绝对路径：数据库、锁、`worker-bin` 里的命令垫片、智能体的 git 配置，以及默认情况下 `work` 里的工单目录。升级前先备份 |
| `promptMaxChars`, `admissionContextMaxPages`, `admissionContextTimeoutMs` | 否 | 已保存提示词配置的最大体积，以及一次研判在多长时间内读多少页线程 |

第一次启动之后，在 **设置 → 部署** 或 DeepSeek Harness 的插件管理器里改机器限制。改动不重启就生效。`stateDir` 不能在那里改。无效的值或未知的配置键会阻止启动；日志会点出那个键。

## 启动与停止

```sh
scripts/dsh-superagent start
scripts/dsh-superagent status
```

`start` 在 tmux 会话 `dsh-superagent` 里跑 `dsh --profile superagent`，别的配置档则是 `dsh-<profile>`。它最多等一分钟，直到运行时就绪或报告一项安装要求。第一次跑时，它打印缺的安装信息。它拒绝重复的实例，包括这个脚本之外启动的。

`stop` 和 `restart` 请求优雅关闭，并最多等执行器五分钟。它们从不强杀进程，也不碰其他 tmux 会话。

`status` 打印进程标识、健康和安装问题。换配置档或端口时，用 `--profile` 和 `--url`，或设置 `DSH_SUPERAGENT_PROFILE` 和 `DSH_SUPERAGENT_URL`。

`status`、`start` 和 `restart` 以四个码之一退出：

| 码 | 含义 |
| --- | --- |
| 0 | 服务正常运行 |
| 1 | 它没在跑、跑了两份、没有应答，或它的运行时报告错误；重启可能修好 |
| 2 | 它在跑，但运行时以外的一部分，例如一个来源或一次策略同步（`sync:<source id>`），报告了错误；重启修不好 |
| 3 | 它在跑，并等着安装；重启修不好 |

一个仍在启动的来源算正常运行。安装没能检查到的飞书机器人也算。

`status` 在 `errors` 下返回失败的部分和它们的细节，在 `setup` 下返回安装要求。`start` 和 `restart` 把那些问题印成行；服务始终起不来时它们以 1 退出。只有故意的 `stop` 才暂停自动恢复。

每个配置档把日志和控制文件存在 `.artifacts/` 下：

- 启动日志：`dsh-superagent-<profile>-<date>-<time>-<pid>.log`，其中进程标识属于这个脚本。最新的 20 份被保留。
- 控制文件：`control/profiles/<profile>/`，装着停止标记、锁和进程标识。

打开日志里印出的地址，然后在侧栏里选择 **Superagent**。

### 每小时检查

在你平时用的那个 DeepSeek Harness 配置档里创建每小时检查，不要建在 `superagent` 里。智能体服务停着的时候，它也必须继续跑。请那个配置档里的一个智能体，用下面这段提示创建每小时日程，并把检出路径填上：

```text
Check dsh-superagent: run `scripts/dsh-superagent --profile superagent status` in <checkout>. Exit 0: do nothing and say nothing. Exit 2: do not restart; tell me each part under `errors` with its detail, unless you already told me about the same ones. Exit 3: do not restart; tell me what `setup` says setup needs, unless you already told me. If it reports manually_stopped: true, leave it stopped. Otherwise (exit 1) run `scripts/dsh-superagent --profile superagent --respect-stop restart`, which exits the same way: 0 means it recovered, so say nothing; 2 and 3 are reported as above; if it exits 1 or fails otherwise, read the newest .artifacts/dsh-superagent-superagent-*.log and the Troubleshooting section of docs/006-deployment.md, fix what you can, and tell me what was wrong and what you did.
```

`--respect-stop` 保留一次故意的停止。只有 `stop` 写这个标记；`start` 清除它。`restart` 不写标记，所以检查可以恢复一次关闭超时的重启。

## 设置

轮询或工作开始之前，安装必须成功。总览列出缺的要求，并链到它们的设置。设置一变，安装再跑一遍。如果你在应用之外修好了什么，例如命令行的认证，选择 **再次检查**。

1. **来源。** 设置 GitHub 仓库（`owner/name`）和智能体命令行，例如 `/home/me/.local/bin/gh-agent`。请求通常经智能体账号的通知到达。一条提及该账号的新议题或评论，在下一次轮询时到达；靠编辑已有议题正文来加提及，不会到达。改发一条新评论。若用飞书，设置机器人命令行、群、每个群是接收全部还是只接收提及，以及是否接收私信。
2. **策略。** 两条示例规则可以改，也可以删。在 **在每个来源上** 之下，加上可以向智能体要求工作的 GitHub 账号和飞书用户；名单上还没有人之前，没有任何请求能成为工单。若用飞书，可以选择一个群，在启动时和 **立即同步** 时从它同步名单。
3. **执行。** 目录根和一条准备命令。要给每张工单一份仓库检出：

   ```sh
   git -C /home/me/src/widgets fetch --no-tags "https://github.com/$DSH_SUPERAGENT_GITHUB_REPOSITORY.git" "${DSH_SUPERAGENT_GITHUB_PR_HEAD:-HEAD}:refs/dsh-superagent/$DSH_SUPERAGENT_TICKET" &&
   git -C /home/me/src/widgets worktree add -b "agent/$DSH_SUPERAGENT_NAME" "$DSH_SUPERAGENT_DIR" "refs/dsh-superagent/$DSH_SUPERAGENT_TICKET"
   ```

   把个人命令行，例如登录成你自己账号的那个，列成被禁止的命令。
4. **提示词。** 设置团队段落：GitHub 的披露行、评审机器人、提交作者和自动合并条件。

### 管理接口

面板用一个本地 HTTP 接口，智能体也可以调用。`SETUP.md` 用它保存设置，服务脚本读取 `GET state`。

把请求发到 `http://127.0.0.1:<port>/orchestrator/api/`，带上 `x-orchestrator-request: 1`。主机必须是 `127.0.0.1:<port>` 或 `localhost:<port>`。缺请求头，或 `Origin` 来自另一个站点，得到 403。

接口保存拥有面板的权限。它们可以改执行环境和安装用的账号字段，这和经主 Session 的工具所作的保存不同。

智能体可以依赖这些路由和字段。`src/app/api.ts` 里的其他路由为面板服务，并跟着面板变。

| 请求 | 应答 |
| --- | --- |
| `GET state` | 每一页都读的东西：`setup`，带 `problem` 和 `feishuProblem`，各自为 `null`，或一个带 `place`、`code`、`values` 和 `detail` 的安装问题，以及安装解析出来之后的 `accounts`；`health`，每个部分一条，见下，带 `state`（`starting`、`ready`、`paused` 或 `error`）、`detail` 和 `at`；`load`，执行器会话的 `running`、工单的 `queued` 和 `maxWorkers`；`paused`；以及 `masterId` |
| `GET settings` | `blocks`：`policy`、`eval` 和 `executor`，各自带 `value` 和 `revision`；`sources`：来源实例，各自带 `id`、`kind`、`installed`、它那一块的 `value`，即 `{"enabled": …, "config": {…}}`，以及 `revision`、它的种类的表单，和它有安装问题时的 `problem`；`policyVersion`；以及 `environment`，执行环境的 `root` 和 `variables` |
| `GET sources` | 每个来源的状态：`state`（`missing`、`off`、`error`、`setup`、`starting`、`paused` 或 `ready`）、它各部分的 `health`、它的安装 `problem`、最后一天的 `stats`、它在那天 `filtered` 的触发，以及它在那天 `dropped` 的消息，最新的丢弃在前，每条带 `channel`、`preview`、若是为某一张工单则带 `ticket`（`id` 和 `title`）、存储里有它的线程时带 `thread`，以及 `dropped.at` 和 `dropped.reason`，即 `channelGone` 或 `channelIgnored` |
| `PUT settings/<block>`，正文 `{"value": …, "revision": n}` | 保存一块：`saved`、这一块新的 `revision`、`policy_changed` 和 `restarting`。`value` 是整块，和 `GET settings` 所持的一样；缺的键取其默认 |
| `POST executor/pause`，正文 `{"paused": true}` 或 `false` | 暂停或恢复处理：`paused` |
| `POST setup/check` | 通过重启 dsh-superagent 再跑一遍安装：`restarting: true`。安装没有要修的东西时被拒绝 |
| `POST policy-sync/<source id>` | 立刻同步那个来源的允许名单：`policy`，即保存下来的来源那一部分 |
| `GET prompts`、`PUT prompts`，正文 `{"config", "baseRevision"}` | 读提示词配置：`current`，带 `revision` 和 `config`；`defaults`；`sections`，会话阅读顺序下的团队段落；`skillName`，团队技能名的模式；`builtin`，团队技能不能占用的名字；以及 `skillPrefix`，会话在其下加载技能名。或保存它：新的 `revision` |
| `POST tickets/<id>/retry`、`POST tickets/<id>/stop` | 重试或停止一张工单，和面板所做的一样 |

健康条目按职责分组：

| 条目 | 报告 |
| --- | --- |
| `runtime` | 应用本身 |
| `models` | 在等模型预算重试的会话 |
| `evals` | 最近一小时里没有后续判断、也没有被忽略的未完成评估 |
| `assessment`, `launch`, `delivery`, `relay`, `notice`, `reply` | [003 执行](003-execution_zh.md) 里描述的存下的工作；送达计的是等待中的工作，回复覆盖渠道消息 |
| `worker` | 执行器会话的事件处理；第一次失败之后才出现 |
| `github`, `feishu`, `history` | 来源轮询、事件和历史补回 |
| `sync:<source id>` | 允许名单的同步 |

一次拒绝带有状态和 `error`，即它的英文文本：

- 400：一个无效的值（`issues`，各自带 `path`）、一次失败的账号检查（`setup`）、一次固定的拒绝（`code` 和 `values`），或正文不是 JSON 或太大。
- 404：某个不存在的东西。
- 409：一个过时的修订。
- 500：其余一切。

这些块：

- `source:github`、`source:feishu`：`{"enabled": true, "config": {…}}`。GitHub 的配置有 `repository` 和 `cli`，以及开关 `observeIssues`、`observePulls`、`observeComments` 和 `observeCi`，默认关，它们接收仓库里每条新议题、新拉取请求和评论，以及默认分支的持续集成；飞书的有 `cli`、`chats`，形如 `[{"chatId": "oc_…", "name": "…", "listen": "mentions"}]`（或 `"all"`）、`directMessages`，以及 `otherChats`，为 `"none"` 或 `"mentions"`。
- `policy`：`{"rules": […], "sources": {"github": {"allowed": [{"id": "<login>"}], "bots": [{"id": "dependabot[bot]"}]}, "feishu": {"allowed": [{"id": "ou_…", "name": "…"}], "syncChat": "oc_…", "admission": {"oc_…": {"unlistedAddressed": "explicit"}}}}}`。`bots` 列出其消息可以成为推断候选的机器人。一行 `admission` 的键，飞书是群标识、`other` 或 `direct`，GitHub 是 `issues`、`pulls`、`comments` 或 `ci`；没设的格取其默认（见 [002 授权](002-authorization_zh.md)）。
- `eval`：`mode`、`timeoutSeconds` 和 `model`。
- `executor`：`model`、`maxWorkers`、`directory`、`prepare` 和 `blockedCommands`。处理暂停是它自己的一项设置 `paused`，由 `POST executor/pause` 来设。

改仓库或命令行会先跑对应的安装检查。检查失败返回 400，细节在 `setup` 里。一次成功的账号变更，或安装尚未完成时的任何保存，返回 `restarting: true`。应用重启期间，接口暂时不可用。

```sh
api=http://127.0.0.1:3082/orchestrator/api
rev=$(curl -s -H 'x-orchestrator-request: 1' $api/settings | jq '.sources[] | select(.id == "github") | .revision')
curl -s -X PUT -H 'x-orchestrator-request: 1' -H 'content-type: application/json' $api/settings/source:github \
  -d "{\"value\": {\"enabled\": true, \"config\": {\"repository\": \"acme/widgets\", \"cli\": \"$HOME/.local/bin/gh-agent\"}}, \"revision\": $rev}"
```

## 安装检查

安装使用 DeepSeek Harness 的子进程服务，和工作者的外壳一样。它去掉名字里含有 KEY、PASSWORD、SECRET 或 TOKEN 的环境变量。

安装核对 GitHub 仓库和智能体的推送权限，写好工作者环境，并检查 git 对 github.com 只用智能体凭证。一次飞书安装失败只停用飞书来源。

安装写下的进程环境：

- **`PATH` 上最前的命令垫片**，在 `<stateDir>/worker-bin`：`gh` 跑智能体的命令行；`lark-cli` 跑机器人的命令行；被禁止的命令失败。垫片在跑它的目标之前，先把自己的目录从 `PATH` 里拿掉。
- **`<stateDir>/worker-gitconfig` 里的 git 配置**：替换操作者的全局和系统 git 配置。它把智能体命令行选为 github.com 的凭证助手，把 GitHub 的 SSH 地址改写成 HTTPS，并定义 Git LFS 过滤器。个人的 git 设置，例如 HTTP 代理，不适用。这里不设提交身份，git 从不为凭证提示，`GH_TOKEN` 和 `GITHUB_TOKEN` 被去掉。

`worker-bin` 或 `worker-gitconfig` 里一份不是 dsh-superagent 写的文件，会让安装停住。工单目录若在一个 git 仓库里，而那个仓库自己的配置又加了凭证，这张工单会失败。

## 试一试

从一个被允许的 GitHub 账号，开一条议题或发一条评论，内容含 `@<agent login> <request>`。在一个轮询间隔内（默认一分钟），总览应当显示出一张工单。智能体确认收到请求，在它自己的会话和目录里工作，并在原来的对话里交付。若用飞书，在一个已配置的群里提及机器人。

## 升级

1. 等到没有执行器正处在一轮之中。
2. 备份 `<stateDir>/orchestrator.sqlite`。
3. 跑 `git pull && pnpm install && pnpm build`。
4. 跑 `scripts/dsh-superagent restart`，并查看它的状态。

一次重启结束正在跑的派工，并用恢复说明启动接替者。团队文本和被覆盖的提示词部分被保留；没有覆盖的部分跟随新的默认。升级之后审阅标成 **默认内容已更新** 的部分。

被较新版本迁移过的数据库，较旧的版本打不开。因此回滚需要升级前的备份。升级 DeepSeek Harness 本身之后，重启正在跑的实例，这样面板加载匹配的产物。

### 智能体预设

工作者和路由预设在 5 秒后把外壳命令移到后台。会话没有它的预设就加载不了。它们的会话还在用时，把 `dsh-superagent-worker` 和 `dsh-superagent-router` 两行留在 `cordis.patch.yml` 里；否则恢复会以 `agent-preset/not-found` 失败。

## 排障

安装问题出现在总览上，带一条通向修复的链接，用面板的语言；日志和接口显示下面的英文文本。

对照这张表时，先看日志或接口里的英文原文，再在左列找到同一句，右列是可以动手的修法。操作者日常只需要记住几件事。允许名单上还没有人的时候，任何请求都变不成工单。处理被暂停时，来源仍在记录触发，但新的评估、输入送达和回复都停着。状态命令返回的码把「服务没在跑」和「服务在跑、但某一部分报错」分开，只有前者才值得先重启。升级之前先等执行器走完当前这一轮，并备份数据库，因为较新版本迁过的库，较旧的版本打不开。面板上的症状和这张表的左列是同一件事的两种说法，修的时候以表里的动作为准。

| 症状 | 原因与修复 |
| --- | --- |
| `The agent GitHub CLI is not set.` | 在 GitHub 卡片上设置智能体的命令行 |
| `The agent GitHub CLI was not found.` | 设置包装的绝对路径，或 `PATH` 上的一个命令名 |
| `The agent GitHub CLI cannot read its account` | 用 `auth login` 登录这个包装 |
| `@… cannot read …` 或 `@… cannot push to …` | 给智能体账号对该仓库的写权限 |
| `The GitHub Source names no repository.` | 在 GitHub 卡片上设置仓库，`owner/name` |
| 工单以 `The prepare command failed: …` 失败 | 输出点出原因，例如克隆路径错了；修好命令，然后重试工单 |
| `git does not read the github.com credential configuration dsh-superagent writes` | 安装 git 2.32 或更高 |
| 工单以 `The git configuration of the repository in … sets …` 失败 | 从克隆的 `.git/config` 里去掉点名的凭证或 `http.*.extraheader` 设置 |
| `… was not written by dsh-superagent` | 把点名的文件移出 `<stateDir>/worker-bin` 或 `<stateDir>/worker-gitconfig`，或另选一个 `stateDir` |
| `The bot lark-cli allows user identities` | 把飞书来源指向一份仅机器人的配置，如上面的飞书机器人 |
| `The bot lark-cli does not work.` 或 `The bot identity of app … is …` | 检查配置的应用标识和密钥 |
| `The Feishu bot is not a member of the chat.` | 在群的设置里把机器人加进群；下一次尝试会成功 |
| `status` 以 2 退出 | 一个部分报告了错误，重启修不好：`errors` 点出每个部分，如管理接口一节所列，并带上细节；每个部分一旦又工作了，就清掉自己的条目 |
| `status` 以 3 退出 | 安装在等：`setup` 说出缺什么，总览链到修好它的那项设置 |
| `status` 以 1 退出 | 服务没在跑、跑了两份、没有应答，或它的运行时报告错误；输出和最新的日志说出是哪一种 |
| `… timed out after 90 s`、`… was stopped by …`、`… exited with code …` 或 `… could not run: …` | 一个命令行失败了但没印出原因；以操作者的用户身份手工跑点名的程序，看它自己的消息 |
| 面板说 dsh-superagent 没有应答 | 它正在重启，或因为配置没有启动；DeepSeek Harness 的日志点出原因 |
| `dsh-superagent configuration has keys it does not use`（日志） | 去掉点名的键；消息说出每个值现在住在哪里 |
| 日志说插件与正在跑的 dsh 不兼容 | 安装一个落在 `engines.dsh` 范围内的 DeepSeek Harness 版本 |
| 升级 DeepSeek Harness 之后面板显示「Failed to load plugins」 | 重启正在跑的实例 |
| 一次提及之后什么都没发生 | 检查作者是否在来源的允许名单上、来源是否开着、处理是否没有暂停。在 GitHub 上，靠编辑议题正文加上的提及不会被收到；在评论里提及智能体 |
