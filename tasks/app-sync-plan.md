---
title: App 数据同步方案（打卡持久化 / 内容热更新 / 知识库回写）
description: 小程序接入微信端后的三条同步链路：云开发打卡持久化、kb.json 免发版热更新、勾选回写知识库
type: plan
status: draft
tags:
  - app
  - sync
  - taro
  - cloud
---

本页记录 HH 成长工作台小程序接入微信端后的**数据同步方案**。方案分三条链路，按优先级排列。

选型沿用 [App 构建方案（PC 网页 + 微信小程序）](./app-build-plan.md) 已确定的 Taro 单代码库与微信云开发（serverless，免自有服务器）。

## 背景

用户于 2026-10-03 会话中告知，已通过 WorkBuddy 完成小程序的微信端接入。此前代码由 [App 源码（导出 / 读取层 / 页面）](./app-source.md) 与 [App 源码落地脚本 v2](./app-source-install-v2.md) 落地，数据层为静态 `kb.json` + 本地存储。

## 现状与阻塞点

### 阻塞点：小程序端打卡未落库

`src/store/checkin.ts` 只实现了 H5 分支（见 [App 源码（导出 / 读取层 / 页面）](./app-source.md) 步骤 2）：

- `getEnv() !== ENV_TYPE.WEB` 时 `read()` 返回 `{}`、`write()` 直接 return；
- 该 store 注释为「H5 走 `localStorage`；小程序端留待接云开发」。

后果：小程序端 `setCheckin` 是空操作，勾选**退出即丢失**；`localStorage` 在小程序环境亦不可用。这是当前最需先修的一处。

### 内容侧：改档案必须重新发版

`src/data/kb.json` 由 `scripts/export.mjs` 生成后随包编译（见 [App 源码（导出 / 读取层 / 页面）](./app-source.md) 步骤 1），档案任何改动都要重新构建并上传小程序。

## 链路 1：打卡数据持久化（跨设备保留）

云开发集合 `checkins`，字段 `{ todoId, checked, date, note }`，权限设**「仅创建者可读写」**——按 openid 天然隔离用户，无需自建鉴权。

`src/store/checkin.ts` 改为异步 + 本地缓存兜底：

```ts
import Taro from '@tarojs/taro'

const KEY = 'hh-growth:checkin'
type Map = Record<string, { checked: boolean; note?: string; date?: string }>

const db = () => Taro.cloud.database()

// 本地缓存：断网或云开发未初始化时降级使用
function readLocal(): Map {
  try {
    return JSON.parse(Taro.getStorageSync(KEY) || '{}')
  } catch {
    return {}
  }
}

function writeLocal(map: Map) {
  Taro.setStorageSync(KEY, JSON.stringify(map))
}

// 启动时拉取；权限为「仅创建者可读写」，只会返回自己的记录
export async function loadCheckins(): Promise<Map> {
  try {
    const { data } = await db().collection('checkins').limit(100).get()
    const map: Map = {}
    data.forEach((r: any) => {
      map[r.todoId] = { checked: r.checked, date: r.date, note: r.note }
    })
    writeLocal(map)
    return map
  } catch {
    return readLocal()
  }
}

// 先写本地立即生效，再异步上云
export async function setCheckin(
  id: string,
  v: { checked: boolean; date?: string; note?: string }
) {
  const map = readLocal()
  map[id] = v
  writeLocal(map)

  const { data } = await db().collection('checkins').where({ todoId: id }).get()
  if (data.length) {
    await db().collection('checkins').doc(data[0]._id).update({ data: v })
  } else {
    await db().collection('checkins').add({ data: { todoId: id, ...v } })
  }
}
```

配套改动：`getCheckin` 由同步改异步后，页面不能直接在渲染中调用。待办页改为在 `useEffect` 里 `loadCheckins()` 存入 state，渲染读 state（当前 `todos.tsx` 是同步读的，见 [App 源码（导出 / 读取层 / 页面）](./app-source.md) 步骤 3）。

## 链路 2：知识库内容热更新（改档案不用发版）

把 `kb.json` 从「编译进包」改为「云端拉取」，使档案更新不再需要重新上传小程序。

- 跑完 `node scripts/export.mjs` 后，将 `src/data/kb.json` 上传到**云存储**；
- 数据库另存一条 `kb_meta { generatedAt }`，做轻量版本比对，避免每次全量下载；
- 小程序启动流程：读 `kb_meta.generatedAt` → 与本地缓存比对 → 有更新才 `wx.cloud.downloadFile` 拉取 `kb.json` → 解析后写入 Storage；首次启动或断网时使用本地缓存。

落地后，档案更新只需「导出 + 上传」两步。

## 链路 3：勾选回写知识库（半自动）

[App 构建方案](./app-build-plan.md) 的 v2 设想是接 OpenKnowledge 服务（`ok start` 的接口或 MCP）做实时编辑与同步。在小程序场景下存在**环境约束**：`ok start` 起的服务运行在本机 localhost，小程序运行在云端，无法访问。

| 做法 | 代价 |
| --- | --- |
| 把 OK 服务部署到公网可达的机器 | 需服务器与备案，成本最高 |
| 云函数中转，再由云函数连回本机服务 | 本机仍需公网暴露，约束不变 |
| **半自动**：打卡只写云数据库，知识库 markdown 由人工或脚本回写 `- [ ]` → `- [x]` | 最省，适合家庭低频使用 |

结论：按当前规模（家庭自用、档案低频更新）采用**半自动**。打卡状态以云数据库为准，知识库 markdown 作为内容源，两侧不追求毫秒级一致。

## 关键约束

- 小程序需已开通云开发并填好环境 ID，且 `app.tsx` 中调用 `Taro.cloud.init({ env })`；
- [App 构建方案](./app-build-plan.md) 已注明：知识库里的勾选框多为**待核实事项而非每日打卡**，同步时不应强行套用每日打卡逻辑；
- 待办 id 沿用 `export.mjs` 生成的 `${doc.slug}#${index}`，回写 markdown 时用该 id 定位原文。

## 执行清单

- [ ] 在微信开发者工具开通云开发，记录环境 ID；
- [ ] 新建 `checkins` 集合并设为「仅创建者可读写」；
- [ ] 替换 `src/store/checkin.ts` 为上面的云开发版本；
- [ ] 待办页改为 `useEffect` 载入 state 渲染；
- [ ] 上传 `kb.json` 到云存储并建 `kb_meta`，改造启动拉取逻辑；
- [ ] 验证：小程序勾选 → 退出重进保留 → 换设备仍保留。

## 相关

- [App 构建方案（PC 网页 + 微信小程序）](./app-build-plan.md)
- [App 源码（导出 / 读取层 / 页面）](./app-source.md)
- [App 源码一键落地脚本](./app-source-install.md)
- [App 源码落地脚本 v2](./app-source-install-v2.md)
