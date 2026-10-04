---
title: 四面同步操作手册
description: Git 端、个人本地 OK 端、小程序云端、小程序个人端的路径、命令、日常闭环与排错。
type: plan
status: draft
tags:
  - sync
  - runbook
  - github
  - weapp
  - open-knowledge
cluster: sync
---
# 四面同步操作手册

本文把 [知识库云地同步方案](./kb-cloud-sync.md) 的四面目标落成可照着敲的命令。选型与三条链路原理见 [App 数据同步方案](./app-sync-plan.md)。导出脚本实现见 [App 源码](./app-source.md)。桥接文件是 [app-sync.md](../app-sync.md)（自动生成，勿手改）。

> [!NOTE]
> 小程序个人端不能直连 GitHub，也不能访问本机 `ok start`。手机只跟微信云开发说话。Git 与手机之间必须经过「本地 OK + hh-growth-app 脚本」这座桥。

## 路径与仓库（先记这三处）

| 名称 | 路径或 URL |
| --- | --- |
| 知识库（本地 OK / Git 工作副本） | `/Users/beryllewis/Documents/OpenKnowledge/鸭仔一家人` |
| 小程序工程 | `/Users/beryllewis/Documents/OpenKnowledge/hh-growth-app` |
| Git 远端 | `https://github.com/uishow/hh-konwledge-base.git` |
| 微信云环境 ID（脚本默认） | `cloud1-d9ghk5qgd4b99b5ce`（可用环境变量 `CLOUD_ENV` 覆盖） |

GitHub 账号：本机桥用 `uishow`。其他家人用**自己的 GitHub 账号**，按下一节邀请为协作者后，再在自己电脑 `ok auth login`。不要互借 token。

## 邀请家人进仓库（Settings → Collaborators）

仓库是个人仓 `uishow/hh-konwledge-base`。步骤依据 [GitHub 官方说明（邀请个人仓库协作者）](../external-sources/github-invite-collaborators.md)（2026-10-04 抓取）。被邀请人得到的是协作者权限（可 clone / pull / push），不能改仓库的 Settings。

### 邀请前，家人先做

1. 打开 [GitHub 注册页](https://github.com/signup) 注册自己的账号（已有账号跳过）。
2. 把 **GitHub 用户名**（头像旁或个人主页 URL 里 `github.com/用户名` 那一段）发给你。也可以发注册用的邮箱。

### 你用 `uishow` 登录后操作

1. 打开仓库首页：[uishow/hh-konwledge-base](https://github.com/uishow/hh-konwledge-base)
2. 点仓库名下方的 **Settings**。若这一行没有 Settings，点 **···**（More）再选 Settings。必须是仓库主人 `uishow` 登录，别的账号看不到完整设置。
3. 左侧栏找到 **Access**（访问），点 **Collaborators**（协作者）。也可直接打开 [协作者设置页](https://github.com/uishow/hh-konwledge-base/settings/access)。
4. 点 **Add people**（添加人员）。
5. 搜索框输入家人的 **GitHub 用户名或邮箱**，在匹配列表里点选正确的人。
6. 点 **Add ＜名字＞ to REPOSITORY**（添加该用户到仓库）。
7. 页面上会出现待接受邀请。邀请未接受前，对方还不能 `git clone` 私有仓；仓库若仍是 Public，别人能看内容，但没有你授权则不能 push。

### 家人接受邀请

1. 查注册邮箱里 GitHub 的邀请信，点链接；或登录 GitHub 后打开 [通知页](https://github.com/notifications)，处理仓库邀请。
2. 也可登录后打开 [仓库首页](https://github.com/uishow/hh-konwledge-base)，页面上方会有 **Accept invitation**（接受邀请）。
3. 接受后，在自己电脑执行（见上文「其他电脑装 OpenKnowledge 后接入 Git」）：

```bash
git clone https://github.com/uishow/hh-konwledge-base.git
cd hh-konwledge-base
ok auth login
ok sync
```

`ok auth login` 必须登**家人自己的** GitHub 账号，不要用 `uishow`。

### 取消邀请或移除协作者

同一页 [协作者设置](https://github.com/uishow/hh-konwledge-base/settings/access)：待接受的邀请可 Cancel；已加入的人可 Remove。移除后对方立刻失去 push 权限。

## 三类数据（命令改错对象会覆盖错文件）

1. **档案**（`growth/`、`tasks/` 等 markdown）：事实源是 Git / 本地 OK。下行命令是在小程序工程里跑 `export.mjs` 生成 `src/data/kb.json`，再让手机读到这份 JSON。
2. **家庭共享运营**（小程序里新建的待办、笔记、习惯、关键日期）：事实源是微信云库集合 `hh_growth` 的 `family` 文档。上行命令是 `npm run sync:kb:pull`，整文件覆盖知识库根目录 [app-sync.md](../app-sync.md)。
3. **个人打卡**：按微信 openid 隔离，事实源是云数据库（方案中的 `checkins`）与该用户手机。不要用 `--write-back` 把个人勾选批量改写别人的 `growth/*.md`，除非明确要改档案里的 `- [ ]`。

## 一次性准备

### 1. 本机（负责人，已有知识库目录）

Git 远端已绑定则跳过 `git remote add`。

```bash
cd "/Users/beryllewis/Documents/OpenKnowledge/鸭仔一家人"
git remote -v
# 应看到 origin → https://github.com/uishow/hh-konwledge-base.git

ok auth login    # 浏览器完成 GitHub Device Flow，首次必做
ok sync          # commit + pull + push
```

若 `git push` 报 `Permission ... denied to uishow`：到 GitHub → Settings → Developer settings → Fine-grained tokens，给当前 token 加上仓库 `hh-konwledge-base`，Contents 设为 Read and write。

若 `git push` 报 non-fast-forward：先 `git pull origin main --no-rebase --allow-unrelated-histories`（仅当远端只有建仓 README、历史无关时），解冲突后再 push。日常用 `ok sync` 即可（内部已 pull-before-push）。

GitHub 仓库若仍是 Public，建议 Settings → Change repository visibility → Private（库内有家庭信息与课表图）。

### 2. 其他电脑装 OpenKnowledge 后接入 Git

```bash
git clone https://github.com/uishow/hh-konwledge-base.git
cd hh-konwledge-base
ok auth login    # 用这台机器主人自己的 GitHub 账号
ok sync
```

不要拷贝另一台机器的 `.ok/local/`。

### 3. 小程序工程（同一台负责人电脑）

```bash
cd "/Users/beryllewis/Documents/OpenKnowledge/hh-growth-app"
npm install
```

上行直读云端（可免「导出到本地」文件）时，在终端会话里设置（不要写进代码、不要提交 git）：

```bash
export WX_APPID=你的小程序AppID
export WX_APPSECRET=你的小程序AppSecret
# 可选：export CLOUD_ENV=cloud1-d9ghk5qgd4b99b5ce
```

### 4. 小程序个人端

- 微信开发者工具打开 `hh-growth-app`，确认已 `Taro.cloud.init` 且环境 ID 正确。
- 真机或体验版登录小程序。写入（待办 / 笔记 / 打卡）应落到云库；断网时只留本地缓存（见 [App 数据同步方案](./app-sync-plan.md)）。
- 云开发控制台确认集合 `hh_growth` 有 `family` 文档（没有则先在小程序里操作一次并等待同步）。

## 日常闭环（负责人机，四面要齐就跑这一套）

顺序不要颠倒：先对齐 Git，再导出给手机，再把手机运营数据拉回知识库，最后再推进 Git。

### 步骤 A — Git 端 ↔ 个人本地 OK 端

```bash
cd "/Users/beryllewis/Documents/OpenKnowledge/鸭仔一家人"
ok sync
```

等价拆开（`ok sync` 不可用时）：

```bash
cd "/Users/beryllewis/Documents/OpenKnowledge/鸭仔一家人"
git add -A
git status
git commit -m "描述本次知识库改动"   # 没有改动会失败，可跳过
git pull --rebase origin main
git push origin main
```

在 OpenKnowledge 里改完档案（计划、课表、约定）后，**先做完本步**，再导出，否则手机会吃到未推送、未拉取的旧稿。

### 步骤 B — 本地 OK → 小程序云端 → 小程序个人端（档案下行）

生成手机要读的 `kb.json`：

```bash
cd "/Users/beryllewis/Documents/OpenKnowledge/hh-growth-app"
node scripts/export.mjs
```

成功时终端会打印文档篇数、待办条数、课表条数。输出文件：`hh-growth-app/src/data/kb.json`。知识库目录由脚本内 `KB_DIR` 指定，默认就是上面的「鸭仔一家人」路径，也可用：

```bash
KB_DIR="/Users/beryllewis/Documents/OpenKnowledge/鸭仔一家人" node scripts/export.mjs
```

让手机用上这份 JSON，按当前实现选一种：

#### B1 随包编译（现在就能做）

```bash
cd "/Users/beryllewis/Documents/OpenKnowledge/hh-growth-app"
npm run sync:kb
```

这等于 `node scripts/export.mjs && npm run build:weapp`。然后用微信开发者工具 **上传代码**，手机预览/体验版才能看到新档案。改一次档案就要发一次小程序，适合尚未做云存储热更新时。

#### B2 云存储热更新（方案已定，执行清单未勾完）

见 [App 数据同步方案](./app-sync-plan.md) 链路 2：把 `src/data/kb.json` 上传到微信云存储；在云数据库写入/更新 `kb_meta.generatedAt`。小程序启动：读 `kb_meta` → 与本地缓存比对 → 有更新才 `wx.cloud.downloadFile`。落地前不要假设「只 export 不发版，手机就会变」。

控制台操作（无 CLI 时）：

1. 微信开发者工具 → 云开发 → 存储，上传 `src/data/kb.json`（覆盖同名文件）。
2. 云开发 → 数据库，集合 `kb_meta` 一条记录，字段 `generatedAt` 与 JSON 里的 `generatedAt` 一致。
3. 家人**完全退出小程序再打开**（不要只切后台），才会走启动拉取。

### 步骤 C — 小程序个人端 ↔ 小程序云端（家人日常，无电脑命令）

在手机小程序里：

- 新增/勾选待办、写笔记、打卡习惯：保存后即写微信云（集合 `hh_growth` 的 `family`，以及方案中的个人 `checkins`）。
- 换一台手机、同一个微信：应能从云库拉回自己的打卡（链路 1 落地后）。未落地时小程序端 `setCheckin` 可能仍是空操作，退出即丢，见 [App 数据同步方案](./app-sync-plan.md) 阻塞点。
- 不要在小程序里指望直接改 GitHub 上的 `growth/*.md`。

可选：小程序里点 **导出到本地**，把 JSON 拷到电脑，文件名保存为：

`/Users/beryllewis/Documents/OpenKnowledge/hh-growth-app/app-state.json`

供下一步文件模式使用。

### 步骤 D — 小程序云端 → 本地 OK → Git 端（运营上行）

**推荐：终端已设置 `WX_APPID` / `WX_APPSECRET`，且云端已有 `family` 文档**

```bash
cd "/Users/beryllewis/Documents/OpenKnowledge/hh-growth-app"
npm run sync:kb:pull
```

该脚本是 `node scripts/export.mjs --sync-back app-state.json --apply`。若当前目录没有 `app-state.json`，会自动请求微信接口读 `hh_growth/family`，并 **整文件覆盖** 知识库的 [app-sync.md](../app-sync.md)。

只预览、不写盘（不要加 `--apply`，否则会覆盖 [app-sync.md](../app-sync.md)）：

```bash
cd "/Users/beryllewis/Documents/OpenKnowledge/hh-growth-app"
node scripts/export.mjs --sync-back
```

**备用：网络到不了 `api.weixin.qq.com`，或没有 AppSecret**

1. 手机小程序「导出到本地」，得到 JSON。
2. 保存为 `hh-growth-app/app-state.json`。
3. 再跑：

```bash
cd "/Users/beryllewis/Documents/OpenKnowledge/hh-growth-app"
npm run sync:kb:pull
```

有 `app-state.json` 时走文件，不访问微信接口。

然后把桥接文件推进 Git，其他装了 OK 的电脑才能 `ok sync` 拉到：

```bash
cd "/Users/beryllewis/Documents/OpenKnowledge/鸭仔一家人"
ok sync
```

其他人只 `ok sync` / `git pull`，**不要手改** [app-sync.md](../app-sync.md)。

### 步骤 E — 可选：把档案里的 `- [ ]` 按导出状态勾掉

这会改 `growth/`、`tasks/` 正文，与覆盖 `app-sync.md` 不是一回事。先预览：

```bash
cd "/Users/beryllewis/Documents/OpenKnowledge/hh-growth-app"
node scripts/export.mjs --write-back state.json
```

确认无误再：

```bash
node scripts/export.mjs --write-back state.json --apply
cd "/Users/beryllewis/Documents/OpenKnowledge/鸭仔一家人"
ok sync
```

`state.json` 键形如 `growth/english#0`，与 export 待办 id 一致。见 [App 源码](./app-source.md) 与脚本注释。

## 按角色谁做什么

| 谁 | 日常操作 |
| --- | --- |
| 负责人（本机有 OK + hh-growth-app） | 跑闭环 A→B→D→再 `ok sync`；改档案用 OpenKnowledge |
| 其他装了 OK 的家人 | 打开项目前 `ok sync`，改完再 `ok sync`；不跑 export，不改 `app-sync.md` |
| 只使用小程序的家人 | 打开小程序读写；等负责人跑完 B/D 后，再打开一次小程序看新档案 / 等 OK 里出现新的 `app-sync.md` |

建议频率：打开项目时做 A；改完一组档案做 A+B；家人在小程序写了一批笔记后做 D+`ok sync`。家庭低频，每日一次闭环通常够。

## 命令速查

| 目的 | 在哪个目录 | 命令 |
| --- | --- | --- |
| Git ↔ 本地 OK | 鸭仔一家人 | `ok sync` |
| 首次登录 GitHub | 鸭仔一家人 | `ok auth login` |
| 知识库 → `kb.json` | hh-growth-app | `node scripts/export.mjs` |
| 导出并编译小程序包 | hh-growth-app | `npm run sync:kb` |
| 云端/导出 JSON → `app-sync.md` | hh-growth-app | `npm run sync:kb:pull` |
| 预览将生成的 `app-sync.md` | hh-growth-app | `node scripts/export.mjs --sync-back` |
| 按 JSON 改档案勾选（预览） | hh-growth-app | `node scripts/export.mjs --write-back state.json` |
| 本地开发 H5 | hh-growth-app | `npm run dev:h5` |
| 本地编译微信小程序 | hh-growth-app | `npm run dev:weapp` |

## 失败时对照

| 现象 | 处理 |
| --- | --- |
| `Permission denied to uishow` | token 没有该仓库 Contents 写权限，见上文准备第 1 节 |
| `non-fast-forward` | 远端有本地没有的提交，先 pull 再 push，不要对 `main` 强推除非你明确要覆盖远端 |
| `github.com` 443 超时 | 换网络/代理后再 `ok sync`；API 通、git 不通时网页操作仓库也无法替代日常 sync |
| `sync:kb:pull` 找不到 AppID | 设环境变量，或改用 `app-state.json` 文件模式 |
| 连不上 `api.weixin.qq.com` | 用文件模式；脚本会提示 |
| 云端没有 family 文档 | 先在小程序里写一条待办/笔记并同步 |
| Git 新了手机旧 | 漏了步骤 B（export + 发版或上传云存储），家人需重新打开小程序 |
| 手机写了 OK/Git 没有 | 漏了步骤 D，或写完后未 `ok sync` |
| 手改 `app-sync.md` 又消失 | 下次 `sync:kb:pull` 会整文件覆盖，属预期 |

## 相关

- [知识库云地同步方案](./kb-cloud-sync.md)
- [App 数据同步方案](./app-sync-plan.md)
- [App 构建方案](./app-build-plan.md)
- [App 源码](./app-source.md)
- [app-sync.md](../app-sync.md)
