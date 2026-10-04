---
title: Work Log
description: Append-only audit trail of changes to this knowledge base.
---
Append-only audit trail. Add one dated entry per turn that creates, edits, or restructures content. The knowledge-base skill describes what to log and the entry shape.

## 2026-10-04：确认四面同步目标

- 用户要求 Git 端、个人本地 OK 端、小程序云端、小程序个人端都能实现数据同步。写入 [知识库云地同步方案](./tasks/kb-cloud-sync.md)：三类数据分工、个人端只连微信云、闭环四步。同步更新 [App 数据同步方案](./tasks/app-sync-plan.md) 半自动桥的含义。
- Files touched: [知识库云地同步方案](./tasks/kb-cloud-sync.md), [App 数据同步方案](./tasks/app-sync-plan.md)
- Open follow-ups: 把导出上传与 `sync:kb:pull` 做成负责人机定时任务，并完成 app-sync-plan 执行清单中尚未勾选的云开发项。

## 2026-10-04：修正导出脚本以纳入 app-sync.md 待办

- 修订[App 源码（导出 / 读取层 / 页面）](./tasks/app-source.md) 与[App 源码一键落地脚本](./tasks/app-source-install.md) 中的 `scripts/export.mjs`：新增 `SOURCE_FILES = ['app-sync.md']`，主流程抽出 `ingest()` 并单独读取根目录桥接文件，使小程序自建待办进入 `kb.json` 的 todos 聚合。
- 背景：此前 `SOURCE_DIRS = ['growth', 'tasks']` 仅扫子目录，漏掉根目录 `app-sync.md`（自动生成，勿手改）；小程序新建的待办因而既不在小程序清单、也不在全景待办里。
- 本会话仅改知识库内源码副本，未触及 `hh-growth-app/`；需把脚本复制到 `hh-growth-app/scripts/export.mjs` 后 `node scripts/export.mjs` 验证待办条数增加。

## 2026-10-03：定稿 App 数据同步方案

- 新增[App 数据同步方案](./tasks/app-sync-plan.md)：针对用户已接入的微信端，记录三条链路——云开发 `checkins` 集合做打卡持久化、`kb.json` 上云存储 + `kb_meta` 版本比对实现免发版热更新、勾选回写知识库采用半自动。
- 明确阻塞点：`store/checkin.ts` 仅实现 H5 分支，小程序端 `setCheckin` 为空操作，勾选退出即丢失（见[App 源码](./tasks/app-source.md)）。
- 记录环境约束：`ok start` 起的服务运行在本机 localhost，小程序在云端无法访问，故实时双向同步改为半自动回写。
- 本会话仅写知识库，未生成或修改 `hh-growth-app` 内代码；方案内的云开发版 `checkin.ts` 需手动落地。

## 2026-10-01：产出 App 源码（导出 / 读取层 / 三页面）

- 按用户「1→2→3」指令，在[App 源码](../tasks/app-source.md)落地三段实现：`scripts/export.mjs`（知识库 → `src/data/kb.json`）、`src/utils/kb.ts` + `MarkdownView` + `store/checkin.ts`、总览/课表/待办打卡三页与 `src/app.scss` 样式。
- 本会话仍**无法向知识库目录之外写文件**（Bash `spawn ENOENT`、native 写工具 `Internal error`），源码只在知识库中留存，需手动复制到 `hh-growth-app/` 对应路径后运行。
- 用户侧已确认 `npm run dev:h5` 编译成功、浏览器显示 Taro 默认 hello world，知识库真实内容尚未接入。

## 2026-10-01：补一键落地脚本

- 新增[App 源码一键落地脚本](../tasks/app-source-install.md)：整段复制到终端执行即可生成 `scripts/export.mjs`、`src/utils/kb.ts`、`MarkdownView`、`store/checkin.ts`、总览/课表/待办三页、`app.config.ts`（原文件备份为 `.bak`）及 `app.scss` 样式。
- 用户已授权写入 `hh-growth-app`，但本会话 native 写工具对知识库内外路径均报 `Internal error`（Bash 仍 `spawn ENOENT`），故改为「脚本 + 用户执行」的方式交付。

## 2026-09-30：建立HH四年成长档案

- 经用户 y 授权，建立[四年成长总计划](./growth/roadmap.md)、[家庭共同约定](./growth/family.md)、[英语计划](./growth/english.md)、[升学探索](./growth/postgraduate.md)及[逐个假期实践计划](./growth/internships.md)。
- 保存[家庭原始信息](./external-sources/family-brief.md)，记录 HUTB 更正；补充[建档复核](./tasks/todo.md)和[防错记录](./tasks/lessons.md)。
- 完成新增内容链接审计与结构回读；具体分数目标、官方政策及岗位机会待后续核实。

## 2026-09-30：课程表入档与英语周计划调整

- 经用户 go 授权，新增[大一上学期课程表](./growth/timetable-2026-autumn.md)，按用户更正将政治经济学记录为周三第9—12节。
- 更新[英语计划](./growth/english.md)为每周210分钟、先试行两周；补充[总计划](./growth/roadmap.md)的大一上学期安排。
- 记录[误读防范规则](./tasks/lessons.md)及[验证结果](./tasks/todo.md)。新增与修改内容链接审计通过；具体钟点、周日及其他课程安排待补充。

## 2026-09-30：准备 Cursor 接续

- 按用户迁移请求建立[Cursor 对话交接](./tasks/cursor-handoff.md)，记录已确认背景、课表更正、文档入口、验证范围及待办。
- 此操作仅建立交接文档，未执行聊天导入或模型切换。

## 2026-09-30：四级报名与四六级目标更新

- 经用户 go 授权，在[家庭原始信息](./external-sources/family-brief.md)追加四级已报名（口试11月21日、笔试12月12日）及目标分（四级过线、六级525分）原话。
- 更新[英语计划](./growth/english.md)：摸底提前、新增到12月12日的分阶段备考与口语安排；同步[总计划](./growth/roadmap.md)、[家庭约定](./growth/family.md)、[防错记录](./tasks/lessons.md)、[交接文档](./tasks/cursor-handoff.md)和[任务清单](./tasks/todo.md)。
- 考点、准考证时间、口试形式及“过线”分数待官方核实。

## 2026-09-30：称呼统一为 HH

- 按用户要求，将鸭妈妈原称呼统一改为「HH」：更新[四年成长总计划](./growth/roadmap.md)、[家庭共同约定](./growth/family.md)、[英语计划](./growth/english.md)、[升学探索](./growth/postgraduate.md)、[假期实践](./growth/internships.md)、[家庭原始信息](./external-sources/family-brief.md)、[Cursor 交接](./tasks/cursor-handoff.md)，以及 growth 文件夹标题。

## 2026-09-30：补全上课作息时间

- 经用户确认 HH 在**北校区**、10月1日起执行**冬季时间**，将[学校作息时间表](./external-sources/hutb-bell-schedule.md)图片存档为来源，并在[课表](./growth/timetable-2026-autumn.md)新增「作息时间（节次对应钟点）」小节。
- 据此：周三第9—12节 政治经济学 = 19:00–22:15；周五第7—8节 体育 = 15:45–17:20。周日及其他课程安排仍待补充。

## 2026-09-30：教务系统课表截图入库

- 经用户选择「原样保存」，将教务系统「我的课表」截图存档为来源 [./external-sources/hutb-course-timetable-2026-autumn.md](./external-sources/hutb-course-timetable-2026-autumn.md)（图片含真实姓名与学号，按用户要求保留）。
- 更新[课表](../growth/timetable-2026-autumn.md)「来源与确认范围」，由「未保存原始图片副本」改为指向已保存来源；正文仍不转录学号。

## 2026-09-30：课表补充教师与地点

- 据[教务系统课表截图来源](../external-sources/hutb-course-timetable-2026-autumn.md)，在[课表](../growth/timetable-2026-autumn.md)「每周课程」表格新增教师、地点两列（如政治经济学 沈时伯 / 2203博学楼；体育 北校区田径场）。

## 2026-09-30：定稿 App 构建方案

- 新增[App 构建方案（PC 网页 + 微信小程序）](../tasks/app-build-plan.md)：记录 Taro 单代码库编译 H5 与微信小程序的选型、目录结构、静态导出方案、页面清单、执行步骤与风险。
- 代码落点定于知识库目录之外；本会话 Bash 与本地写文件工具均不可用，建库与构建需在具备命令行与文件工具的环境执行。仅记录方案，未生成任何应用代码。

## 2026-10-03：统一公仔名字

- 按用户明确要求，将[家庭共同约定](./growth/family.md)、[家庭信息](./external-sources/family-brief.md)及[Cursor交接](./tasks/cursor-handoff.md)中的公仔名字统一改为黄小鸭，并补充[防错记录](./tasks/lessons.md)。家庭名称及项目路径保持不变。

## 2026-10-04：确立知识库云地同步方案（OpenKnowledge 原生 git 同步）

- 新增[知识库云地同步方案](./tasks/kb-cloud-sync.md)：基于 OpenKnowledge 原生 `ok sync`（GitHub 远端）实现云地同步；经用户确认采用「原生 git 同步 / 混合形态（部分本地 OK、部分仅用云端小程序）/ 最终一致」三决策，规避自建服务器与备案成本。
- 澄清拓扑：云端 = GitHub 远端（KB 内容单一事实源），微信云开发 `hh_growth` 仅承载小程序运营数据并经 `app-sync.md` 桥接回库（与既有 app-sync-plan.md 链路 2/3 衔接）。
- 确认环境：已 `ok auth login`（uishow）；`.ok/local/` 已由 OpenKnowledge 自带 `.ok/.gitignore` 隔离；根 `.gitignore` 增补 `.claude/`、`.codex/`、`.cursor/`，避免本地 AI 编辑器配置进入共享仓库。
- 待用户侧完成：创建 GitHub 远端仓库、`git remote add origin`、首次 `ok sync` 推送；此三步为外部动作，需用户授权 / 命名后执行。
