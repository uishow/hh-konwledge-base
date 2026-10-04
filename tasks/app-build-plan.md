---
title: App 构建方案（PC 网页 + 微信小程序）
description: 把 HH 成长知识库做成 Taro 双端应用的定稿方案：技术选型、目录结构、数据导出、页面清单、执行步骤与风险
type: plan
status: draft
tags:
  - app
  - build
  - taro
---
本页记录把本项目做成**电脑端网页工作台**与**微信小程序**的定稿方案，供后续在具备文件/终端工具的环境里执行。

## 已确认决策

| 项 | 决定 |
| --- | --- |
| 技术 | **Taro（React + TypeScript）** 单代码库 → H5（PC 网页）+ weapp（微信小程序） |
| 范围 | PC 网页与小程序**并行**开发 |
| 后端 | 微信**云开发**（云数据库、授权登录、文件存储），serverless，免自有服务器、免域名备案 |
| 数据 | **先静态**：导出 markdown 为 JSON；v2 再接 OpenKnowledge 服务做实时编辑与同步 |
| 功能 | 展示 + 待办提醒/打卡（先）；编辑与实时同步（v2） |
| 代码落点 | `/Users/beryllewis/Documents/OpenKnowledge/hh-growth-app/`（**在知识库目录之外**，避免被当成 KB 文档） |

## 环境约束（2026-09-30 会话实测）

- 该会话 **Bash 不可用**（`spawn ENOENT`），native 写文件工具报内部错误：只能写入知识库，**无法**创建外部仓库、`npm install`、构建或跑服务。
- 该会话**没有「一键发布 WorkBuddy 小程序」工具**：产出标准微信云开发小程序源码后，由用户在 WorkBuddy 小程序发布或微信开发者工具完成上传。
- WorkBuddy 的小程序发布能力已于 2026-09-24 上线（5.6.1 及以上），公开报道称其自带云数据库、授权登录与文件存储，无需自建服务器（来源尚未入库：`(TODO: needs source)`）。

## 目录结构

```text
hh-growth-app/
├─ package.json            # Taro 依赖与脚本
├─ config/                 # H5 + weapp 编译配置
├─ tsconfig.json
├─ project.config.json     # 微信小程序项目配置
├─ scripts/export.mjs      # 读 OK 知识库 → src/data/kb.json
├─ src/
│  ├─ data/kb.json         # 导出产物（不入 git，脚本生成）
│  ├─ app.tsx / app.config.ts
│  ├─ pages/  dashboard | plan | timetable | english | postgraduate | internships | family | todos
│  ├─ components/  MarkdownView | Checklist | ReminderCard
│  ├─ store/               # 打卡状态：localStorage（网页）/ 云开发（小程序）
│  └─ utils/kb.ts          # 读取与聚合 kb.json
└─ README.md               # 安装 / 导出 / 构建 / 发布步骤
```

## 数据导出（静态阶段）

`scripts/export.mjs` 解析 `growth/*.md`、`tasks/*.md`：

- `gray-matter` 取 frontmatter；按 `##` 切分节；`marked` 渲染 markdown；正则提取 `- [ ]` / `- [x]` 为勾选框。
- `docs[]`：`slug` / `title` / `description` / `type` / `status` / `sections[{heading, markdown, checkboxes}]`。
- `todos[]`：聚合全部勾选框 `{id, doc, section, text, checked}`。
- `timetable{courses[], bell{}}`：课表额外用表格解析出结构化课程（星期、节次、课程、教师、地点、周次），作息钟点见下方来源。
- 源 markdown 内的 `./x.md` 链接在导出时转为应用内路由，避免外链失效。

## 页面与功能

- 8 个页面：总览（三目标 + 近期待办 + 关键日期）、[四年总计划](../growth/roadmap.md)、[课表](../growth/timetable-2026-autumn.md)、[英语](../growth/english.md)、[升学](../growth/postgraduate.md)、[实践](../growth/internships.md)、[家庭约定](../growth/family.md)、待办与打卡。
- **PC 网页**：`localStorage` 打卡 + 浏览器通知提醒 + 关键日期卡片（四级口试 11月21日、笔试 12月12日）。
- **微信小程序**：`<rich-text>` 渲染 markdown，打卡写入云开发数据库（跨设备保留），可做订阅消息提醒。
- 打卡呈现为「勾选 + 备注 + 日期」。知识库里的勾选框多为**待核实事项**而非每日打卡，不强行套用每日打卡逻辑。

## 执行步骤（需在终端环境执行）

```bash
mkdir -p /Users/beryllewis/Documents/OpenKnowledge/hh-growth-app
cd /Users/beryllewis/Documents/OpenKnowledge/hh-growth-app
npx @tarojs/cli init . --template default
npm install gray-matter marked
node scripts/export.mjs        # 生成 src/data/kb.json
npm run dev:h5                 # 本地预览 PC 网页
npm run build:weapp            # 产出 dist/ 供小程序发布
```

小程序侧：用 WorkBuddy 小程序发布导入 `dist/`，或在微信开发者工具导入 `dist/` 并开通云开发后上传。

## v2：实时编辑与同步

接 OpenKnowledge 服务（`ok start` 的接口或 MCP）：内容编辑经此写回知识库，并配合云开发做跨端实时同步。

## 风险与待验证

- **无法本地构建验证**：若目标环境同样不可用命令行，首版需来回修构建错误。
- **Taro H5 视口**：默认偏移动端，需响应式样式才具备「工作台」形态；若不满意，可改为「Vite 网页 + 原生云开发小程序」两套代码。
- **小程序 markdown 交互**：`<rich-text>` 仅展示，内部链接不可点击，锚点跳转需额外处理。
- 本项目已有可点击查看的来源：
  - [学校作息时间表](../external-sources/hutb-bell-schedule.md)（课表钟点依据）
  - [教务系统课表截图](../external-sources/hutb-course-timetable-2026-autumn.md)（课程、教师、地点依据）
  - [家庭建档原始信息](../external-sources/family-brief.md)（目标与更正原文）
