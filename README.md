# dsh-plugins-standalone

两个 **DeepSeek Harness** 插件，各自独立。装到**任意** `dsh web` 后端上即可用。

两者都按**官方插件格式**打包：`locale/<lang>.json`（提供界面上的标题与描述）、
`icon.svg`（由 package.json 顶层 `icon` 引用），以及 `dsh.bundle.patch` 指向的
`cordis.patch.yml`（profile 靠它插入插件行）；另各带一份 MIT `LICENSE`。
装 / 启停 / 卸载一律走官方 `pluginManager`，**不需要**先装任何“主插件”或“管理桥”。

| 插件 | 提供什么 |
| --- | --- |
| [`dsh-file-transfer`](#1-dsh-file-transfer) | 原始字节写盘 `transfer.write`、建目录 `transfer.mkdir`、列举任意绝对目录 `transfer.list`、打包 `transfer.archive`、探测 `transfer.describe` |
| [`dsh-ssh-link`](#2-dsh-ssh-link) | `ssh_exec` 工具、`ssh` subagent provider、`/api/ssh.*` 路由（列目标 / 测连通 / 跑命令 / 自检） |

两个包都是**零运行时依赖**的（只用 Node 内置模块），这是可移植性的前提：
插件被 `npm pack` 成 tgz 装进 profile 的 `node_modules/`，而 `@deepseek-ai/*`
**不在那个解析路径上**，所以用不了 `defineTool` 之类的宿主侧工具函数。

## 安装

先决条件：目标主机有 `node >= 18` 和 **`pnpm`**（`dsh plugin add` 是把参数转发给 pnpm 的）。

```sh
dsh plugin --profile web add \
  https://github.com/184647604/dsh-plugins-standalone/releases/download/v0.1.0/dsh-file-transfer-0.3.0.tgz

dsh plugin --profile web add \
  https://github.com/184647604/dsh-plugins-standalone/releases/download/v0.1.0/dsh-ssh-link-0.2.0.tgz
```

装完**重启一次 `dsh web`**（首次要靠 profile 的 bundle 层把它带起来）。之后的启停/卸载**免重启**。

> 上面两条 URL 都指向 `v0.1.0` —— 新仓库的第一个 release，**两个包都要挂齐**。
> 一个 release 是整套的：少挂一个 tgz，那条直链就是 404，而 App 的「复制安装命令」
> 正是把这条链接交给用户。历史上旧仓库在 `v0.1.9` 上真踩过这个坑。

## ⚠️ 一律用完整 release URL，不要用裸包名

**这是本仓库唯一一条硬性安装纪律**，因为裸包名会**静默装错**：

| 包名 | npm 上是什么 |
| --- | --- |
| `dsh-file-transfer` | 无人占用 —— 裸名会明确 404 |
| `dsh-ssh-link` | 无人占用 —— 裸名会明确 404 |
| `dsh-ssh-bridge` | ⚠️ **是别人的**（lance-kanglu 的 OpenWRT 路由器 SSH 桥 v0.4.2，关键词同样是 `dsh / deepseek-harness / plugin / ssh / bridge`） |
| `dsh-plugin-center` | ⚠️ **是别人的**客户端插件（作者 `gh503`，2026-08-22 发布） |

`dsh plugin add dsh-ssh-bridge` 会**装上别人的代码而且不报错**。`dsh-ssh-link` 就是
为了绕开这个名字才改的：万一真走了裸名路径，也只会得到一条明确的 404。
**不要再改回 `dsh-ssh-bridge`。**

装本仓库的包**一律带上完整的 release URL**。

## 为什么用 tarball URL 而不是 `git+https://...`

本仓库是**两个包并列**的，仓库根目录没有 `package.json`。
`dsh plugin add git+https://github.com/184647604/dsh-plugins-standalone.git` 指向的是仓库根，
不是一个可安装的包，**会失败**。用 release 挂出来的 `.tgz` 是每个包独立、版本固定、
内容不可变的产物。

## 1. `dsh-file-transfer`

给 dsh web 后端加一个原始字节写盘端点，让客户端（AiChatbox Android App 的文件传输页）
能把手机上的文件直接写进后端文件系统。

**为什么不用官方的 `/api/session/uploadFileBinary`**：官方那个流式上传端点把字节交给
attachment 服务，而该服务靠「逐级 fsync 到文件系统根」证明持久性。在 Termux 上，
`DSH_HOME` 的祖先链会走到 `/data/data`，而对它 `open(..., O_RDONLY)` 必然 **EACCES** ——
于是**任何**上传都以这个错误告终，且没有任何配置项可以限制。本插件完全绕开 attachment
存储，直接把字节写到调用方指定的路径。

| 端点 | 作用 |
| --- | --- |
| `POST /api/transfer.write?path=<绝对目录>&name=<文件名>` | 请求体是**原始字节**，先写 `.<name>.<uuid>.part` 再 `fsync`+`rename` |
| `POST /api/transfer.mkdir` | 建目录（旧协议信封，参数在 payload 里） |
| `POST /api/transfer.list` | 列举**任意绝对目录**，条目形状与官方 `workspaceFiles/list` 一致 |
| `POST /api/transfer.archive` | 把后端上的绝对目录打成压缩包 |
| `POST /api/transfer.describe` | 探测「这个后端装没装」，并回报 `places` 入口候选 |

安全约束：目标目录必须**已存在**、文件名必须是纯文件名（含 `/`、`\`、`\0`、`.`、`..`
一律 400 拒绝，不做静默 basename 改写）、POST + Origin 门禁、可选 `config.token`
（要求 `x-dsh-file-transfer-token` 头）、`config.roots` 白名单、`config.maxBytes`（默认 8 GiB）。

`list` 存在的理由：官方 `workspaceFiles` 的边界是**不对称**的 —— `list` 调 `confine()`
拒绝工作区外的路径，而 `readBytes`/`stat` 走 `locateFile()` **没有**边界检查
（上游注释即写明 *"files outside it are allowed"*）。结果是工作区外的文件**读得到、
却列不出来**：你知道绝对路径就能下载，却永远发现不了。本端点补的正是「列举」那一半。

## 2. `dsh-ssh-link`

用 SSH 把远端机器接成本机可用的能力 —— **不需要配对**，只要公钥认证能用。

| 能力 | 形态 | 用途 |
|---|---|---|
| `ssh_exec` 工具 | 模型工具 | 直接在那台机器上跑命令，**不起 agent、不烧 token** |
| `ssh` subagent provider | 进程级注册 | 配合一行 preset 配置得到 `subagent_ssh`：每次调用 = 远端起一个 `dsh --profile headless` |
| `/api/ssh.*` 路由 | HTTP | 给 App / 脚本用：列目标、测连通、跑命令、自检 |

**这个插件只有客户端能力：它连出去，不接进来。** 不监听任何端口、不装也不启动
`sshd`、不读写任何 `authorized_keys`、不认识"配对"这个概念。要双向，就在两边各装一份，
并各自把 `sshd` 准备好 —— 那是机器自己的运维，不属于本插件。

⚠️ 零依赖是本仓库的硬约束，所以插件**从不**替你装 openssh（不碰 `pkg`/`apt`/`brew`）。
「插件装上了」**不等于**「能用」，因此有 `ssh.doctor` 做环境自检。

**装完还必须再配一行**才能让模型委派：provider 注册在 host 组合里，而每个 agent 可见的
委派工具在 **preset** 层。「host 里有 provider」不等于「agent 有工具」——
详见包内 `README.md`。

## 从源码更新

改完插件后，在项目根目录跑：

```sh
tools/publish-plugins.sh v0.1.1
```

它会重新镜像两个插件、打包、提交、推送，并把新版本的 tgz 传成 release 资产。

## License

MIT
