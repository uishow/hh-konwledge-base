---
title: App 源码（导出 / 读取层 / 页面）
description: hh-growth-app 的可复制实现源码
type: note
status: draft
tags:
  - app
  - taro
  - source
---

本页保存 [App 构建方案（PC 网页 + 微信小程序）](./app-build-plan.md) 的实现源码。

> [!WARNING]
> 当前会话**无法向知识库目录之外写文件**（Bash `spawn ENOENT`、native 写工具 `Internal error`），所以代码先落在这里，再手动复制到 `/Users/beryllewis/Documents/OpenKnowledge/hh-growth-app/` 的对应路径。

## 步骤 1：`scripts/export.mjs`

把知识库 markdown 解析成 `src/data/kb.json`。依赖 `gray-matter` 与 `marked`。

```js
// 读 OpenKnowledge 知识库 → 生成 src/data/kb.json
// 用法: node scripts/export.mjs
import fs from 'node:fs/promises'
import path from 'node:path'
import matter from 'gray-matter'
import { marked } from 'marked'

// 知识库目录（可用环境变量覆盖）
const KB_DIR =
  process.env.KB_DIR || '/Users/beryllewis/Documents/OpenKnowledge/鸭仔一家人'
const SOURCE_DIRS = ['growth', 'tasks']
const SOURCE_FILES = ['app-sync.md'] // 根目录的 app↔KB 桥接镜像，含小程序自建待办
const OUT_FILE = path.resolve(process.cwd(), 'src/data/kb.json')

// 知识库文档 → 应用内路由映射
const ROUTE_MAP = {
  'growth/roadmap': '/pages/plan/plan',
  'growth/timetable-2026-autumn': '/pages/timetable/timetable',
  'growth/english': '/pages/english/english',
  'growth/postgraduate': '/pages/postgraduate/postgraduate',
  'growth/internships': '/pages/internships/internships',
  'growth/family': '/pages/family/family',
}

function slugOf(dir, file) {
  return `${dir}/${path.basename(file, '.md')}`
}

// 按 "## " 切分小节；"## " 之前的内容归到 "(概览)"
function splitSections(body) {
  const sections = []
  let heading = '(概览)'
  let buf = []
  const push = () => {
    const markdown = buf.join('\n').trim()
    if (markdown) sections.push({ heading, markdown })
    buf = []
  }
  for (const line of body.split('\n')) {
    const m = /^##\s+(.+?)\s*$/.exec(line)
    if (m) {
      push()
      heading = m[1]
    } else {
      buf.push(line)
    }
  }
  push()
  return sections
}

// 提取 - [ ] / - [x] 勾选框
function extractCheckboxes(markdown) {
  const out = []
  const re = /^\s*[-*]\s+\[([ xX])\]\s+(.+?)\s*$/gm
  let m
  while ((m = re.exec(markdown))) {
    out.push({ text: m[2], checked: m[1].toLowerCase() === 'x' })
  }
  return out
}

// 知识库内部 ./x.md 链接 → 应用内路由；无法映射的降级为纯文本
function rewriteLinks(markdown, currentDir) {
  return markdown.replace(
    /\[([^\]]+)\]\(\.\/([^)]+?)\.md(?:#[^)]*)?\)/g,
    (_all, text, target) => {
      const route = ROUTE_MAP[`${currentDir}/${target}`]
      return route ? `[${text}](${route})` : text
    }
  )
}
// 极简表格解析，供课表页结构化使用
function parseTable(markdown) {
  const rows = []
  let header = null
  for (const line of markdown.split('\n')) {
    const trimmed = line.trim()
    if (!trimmed.startsWith('|')) {
      header = null
      continue
    }
    const cells = trimmed
      .replace(/^\||\|$/g, '')
      .split('|')
      .map((c) => c.trim())
    if (/^[-: |]+$/.test(trimmed.replace(/\|/g, ''))) continue // 分隔行
    if (!header) {
      header = cells
    } else {
      const row = {}
      header.forEach((h, i) => {
        row[h] = cells[i] ?? ''
      })
      rows.push(row)
    }
  }
  return { header, rows }
}

async function readDir(dir) {
  try {
    const files = await fs.readdir(path.join(KB_DIR, dir))
    return files.filter((f) => f.endsWith('.md')).sort()
  } catch (err) {
    console.warn(`跳过 ${dir}: ${err.message}`)
    return []
  }
}

async function main() {
  const docs = []

  async function ingest(dir: string, file: string) {
    const raw = await fs.readFile(path.join(KB_DIR, dir, file), 'utf8')
    const { data, content } = matter(raw)
    const slug = dir ? slugOf(dir, file) : path.basename(file, '.md')
    const sections = splitSections(content).map((s) => {
      const markdown = rewriteLinks(s.markdown, dir)
      return {
        heading: s.heading,
        markdown,
        html: marked.parse(markdown),
        checkboxes: extractCheckboxes(markdown),
      }
    })
    docs.push({
      slug,
      title: data.title || slug,
      description: data.description || '',
      type: data.type || '',
      status: data.status || '',
      tags: data.tags || [],
      sections,
    })
  }

  for (const dir of SOURCE_DIRS) {
    for (const file of await readDir(dir)) {
      await ingest(dir, file)
    }
  }
  // 根目录桥接文件（小程序自建待办）不在子目录内，单独读
  for (const file of SOURCE_FILES) {
    try {
      await fs.access(path.join(KB_DIR, file))
      await ingest('', file)
    } catch {
      console.warn(`跳过根文件 ${file}: 不存在`)
    }
  }

  // 聚合全部勾选框为待办
  const todos = []
  for (const doc of docs) {
    for (const section of doc.sections) {
      section.checkboxes.forEach((box, i) => {
        todos.push({
          id: `${doc.slug}#${i}`,
          doc: doc.slug,
          docTitle: doc.title,
          section: section.heading,
          text: box.text,
          checked: box.checked,
        })
      })
    }
  }

  // 课表：取课表文档里第一个表格
  let timetable = { courses: [], bell: {} }
  const timetableDoc = docs.find((d) => d.slug.includes('timetable'))
  if (timetableDoc) {
    for (const section of timetableDoc.sections) {
      const { rows } = parseTable(section.markdown)
      if (rows.length) {
        timetable = { courses: rows, bell: {} }
        break
      }
    }
  }

  const payload = {
    generatedAt: new Date().toISOString(),
    source: KB_DIR,
    docs,
    todos,
    timetable,
  }

  await fs.mkdir(path.dirname(OUT_FILE), { recursive: true })
  await fs.writeFile(OUT_FILE, JSON.stringify(payload, null, 2), 'utf8')
  console.log(
    `✔ src/data/kb.json — ${docs.length} 篇文档，${todos.length} 条待办，${timetable.courses.length} 条课表`
  )
}

main().catch((err) => {
  console.error(err)
  process.exit(1)
})
```

## 步骤 2：`src/utils/kb.ts` + `src/components/MarkdownView.tsx`

数据读取与渲染的公共层。

### `src/utils/kb.ts`

```ts
import kb from '../data/kb.json'

export interface Checkbox {
  text: string
  checked: boolean
}

export interface Section {
  heading: string
  markdown: string
  html: string
  checkboxes: Checkbox[]
}

export interface Doc {
  slug: string
  title: string
  description: string
  type: string
  status: string
  tags: string[]
  sections: Section[]
}

export interface Todo {
  id: string
  doc: string
  docTitle: string
  section: string
  text: string
  checked: boolean
}

interface KB {
  generatedAt: string
  source: string
  docs: Doc[]
  todos: Todo[]
  timetable: { courses: Record<string, string>[]; bell: Record<string, string> }
}

const data = kb as unknown as KB

export function allDocs(): Doc[] {
  return data.docs
}

export function getDoc(slug: string): Doc | undefined {
  return data.docs.find((d) => d.slug === slug || d.slug.endsWith(`/${slug}`))
}

export function allTodos(): Todo[] {
  return data.todos
}

export function todosOfDoc(slug: string): Todo[] {
  return data.todos.filter((t) => t.doc === slug)
}

export function getTimetable() {
  return data.timetable
}

export function generatedAt(): string {
  return data.generatedAt
}
```

### `src/components/MarkdownView.tsx`

H5 用 `dangerouslySetInnerHTML`，小程序用 `<RichText>`。

```tsx
import { View, RichText } from '@tarojs/components'
import { ENV_TYPE, getEnv } from '@tarojs/taro'

export default function MarkdownView({ html }: { html: string }) {
  if (getEnv() === ENV_TYPE.WEAPP) {
    return <RichText nodes={html} />
  }
  return (
    <View className='markdown-body' dangerouslySetInnerHTML={{ __html: html }} />
  )
}
```

### `src/store/checkin.ts`

H5 走 `localStorage`；小程序端留待接云开发。

```ts
import { getEnv, ENV_TYPE } from '@tarojs/taro'

const KEY = 'hh-growth:checkin'

type Map = Record<string, { checked: boolean; note?: string; date?: string }>

function read(): Map {
  if (getEnv() !== ENV_TYPE.WEB) return {}
  try {
    return JSON.parse(localStorage.getItem(KEY) || '{}')
  } catch {
    return {}
  }
}

function write(map: Map) {
  if (getEnv() !== ENV_TYPE.WEB) return
  localStorage.setItem(KEY, JSON.stringify(map))
}

export function getCheckin(id: string) {
  return read()[id]
}

export function setCheckin(
  id: string,
  value: { checked: boolean; note?: string; date?: string }
) {
  const map = read()
  map[id] = value
  write(map)
}
```

## 步骤 3：三个页面

先做总览、课表、待办打卡三页，跑通后再补其余 5 页。

### `src/pages/index/index.tsx`（总览）

```tsx
import { useEffect, useState } from 'react'
import { View, Text } from '@tarojs/components'
import Taro from '@tarojs/taro'
import { allTodos, generatedAt } from '../../utils/kb'

const KEY_DATES = [
  { label: '英语四级口试', date: '2026-11-21' },
  { label: '英语四级笔试', date: '2026-12-12' },
]

export default function Index() {
  const [todos, setTodos] = useState(allTodos())

  useEffect(() => {
    setTodos(allTodos())
  }, [])

  const pending = todos.filter((t) => !t.checked)

  return (
    <View className='page'>
      <Text className='h1'>HH 成长工作台</Text>
      <Text className='meta'>数据生成于 {generatedAt().slice(0, 10)}</Text>

      <Text className='h2'>关键日期</Text>
      {KEY_DATES.map((d) => (
        <View key={d.date} className='card'>
          <Text className='card-title'>{d.label}</Text>
          <Text className='card-date'>{d.date}</Text>
        </View>
      ))}

      <Text className='h2'>近期待办（{pending.length}）</Text>
      {pending.slice(0, 8).map((t) => (
        <View key={t.id} className='todo'>
          <Text className='todo-text'>{t.text}</Text>
          <Text className='todo-from'>{t.docTitle}</Text>
        </View>
      ))}

      <View className='nav'>
        <Text className='link' onClick={() => Taro.navigateTo({ url: '/pages/timetable/timetable' })}>
          课表 →
        </Text>
        <Text className='link' onClick={() => Taro.navigateTo({ url: '/pages/todos/todos' })}>
          待办与打卡 →
        </Text>
      </View>
    </View>
  )
}
```

### `src/pages/timetable/timetable.tsx`（课表）

```tsx
import { View, Text } from '@tarojs/components'
import { getTimetable } from '../../utils/kb'

export default function Timetable() {
  const { courses } = getTimetable()
  const columns = courses.length ? Object.keys(courses[0]) : []

  return (
    <View className='page'>
      <Text className='h1'>课表</Text>
      {!courses.length && <Text className='meta'>kb.json 里没有解析到课表表格</Text>}
      {!!courses.length && (
        <View className='table'>
          <View className='tr head'>
            {columns.map((c) => (
              <Text key={c} className='th'>
                {c}
              </Text>
            ))}
          </View>
          {courses.map((row, i) => (
            <View key={i} className='tr'>
              {columns.map((c) => (
                <Text key={c} className='td'>
                  {row[c]}
                </Text>
              ))}
            </View>
          ))}
        </View>
      )}
    </View>
  )
}
```

### `src/pages/todos/todos.tsx`（待办与打卡）

```tsx
import { useEffect, useState } from 'react'
import { View, Text } from '@tarojs/components'
import { allTodos } from '../../utils/kb'
import { getCheckin, setCheckin } from '../../store/checkin'

export default function Todos() {
  const [todos, setTodos] = useState(allTodos())

  useEffect(() => {
    setTodos(allTodos())
  }, [])

  const toggle = (id: string, checked: boolean) => {
    const next = !checked
    setCheckin(id, { checked: next, date: new Date().toISOString().slice(0, 10) })
    setTodos((prev) => prev.map((t) => (t.id === id ? { ...t, checked: next } : t)))
  }

  const isChecked = (t: { id: string; checked: boolean }) =>
    getCheckin(t.id)?.checked ?? t.checked

  return (
    <View className='page'>
      <Text className='h1'>待办与打卡</Text>
      {todos.map((t) => (
        <View
          key={t.id}
          className={'todo' + (isChecked(t) ? ' done' : '')}
          onClick={() => toggle(t.id, isChecked(t))}
        >
          <Text className='box'>{isChecked(t) ? '☑' : '☐'}</Text>
          <View className='todo-main'>
            <Text className='todo-text'>{t.text}</Text>
            <Text className='todo-from'>
              {t.docTitle} · {t.section}
            </Text>
          </View>
        </View>
      ))}
      {!todos.length && <Text className='meta'>知识库里没有解析到勾选框</Text>}
    </View>
  )
}
```

### `src/app.config.ts`（注册页面）

在 `pages` 数组里补上新页面，其余保持 init 生成的原样：

```ts
export default defineAppConfig({
  pages: [
    'pages/index/index',
    'pages/timetable/timetable',
    'pages/todos/todos',
  ],
  window: {
    backgroundTextStyle: 'light',
    navigationBarBackgroundColor: '#fff',
    navigationBarTitleText: 'HH 成长',
    navigationBarTextStyle: 'black',
  },
})
```

### `src/app.scss`（追加样式）

PC 端用 `max-width` 收成工作台宽度，避免 Taro 默认的移动端铺满。

```scss
.page {
  max-width: 960px;
  margin: 0 auto;
  padding: 24px 20px 48px;
}

.h1 {
  display: block;
  font-size: 28px;
  font-weight: 700;
  margin-bottom: 8px;
}

.h2 {
  display: block;
  font-size: 20px;
  font-weight: 600;
  margin: 24px 0 12px;
}

.meta {
  display: block;
  font-size: 13px;
  color: #888;
}

.card {
  display: flex;
  justify-content: space-between;
  padding: 12px 16px;
  margin-bottom: 8px;
  border: 1px solid #eee;
  border-radius: 8px;
}

.card-title {
  font-size: 15px;
}

.card-date {
  font-size: 15px;
  color: #d4380d;
}

.todo {
  display: flex;
  gap: 10px;
  padding: 10px 12px;
  border-bottom: 1px solid #f0f0f0;
}

.todo.done .todo-text {
  color: #aaa;
  text-decoration: line-through;
}

.box {
  font-size: 18px;
}

.todo-main {
  display: flex;
  flex-direction: column;
}

.todo-text {
  font-size: 15px;
}

.todo-from {
  font-size: 12px;
  color: #999;
}

.nav {
  display: flex;
  gap: 20px;
  margin-top: 32px;
}

.link {
  color: #1677ff;
  font-size: 15px;
}

.table {
  border: 1px solid #eee;
  border-radius: 8px;
  overflow: hidden;
}

.tr {
  display: flex;
}

.tr.head {
  background: #fafafa;
}

.th,
.td {
  flex: 1;
  padding: 8px 10px;
  font-size: 13px;
}

.th {
  font-weight: 600;
}

.markdown-body {
  font-size: 14px;
  line-height: 1.7;
}
```

## 落地与验证

在 `hh-growth-app/` 里依次执行：

```bash
node scripts/export.mjs   # 生成 src/data/kb.json
npm run dev:h5            # 起 H5 开发服务
```

打开终端打印的地址，依次确认：

1. **总览页**显示关键日期与近期待办条数（不再是 hello world）；
2. 点「课表 →」能看到从 [大一上学期课程表](../growth/timetable-2026-autumn.md) 解析出的表格；
3. 点「待办与打卡 →」能勾选，刷新后状态保留（`localStorage`）。

> [!NOTE]
> 若 `import kb from '../data/kb.json'` 报 TS 错误，在 `tsconfig.json` 里确认 `"resolveJsonModule": true`（Taro 默认模板通常已开启）。

## 后续

- 补齐其余 5 个页面（[四年总计划](../growth/roadmap.md)、英语、升学、实践、家庭约定），复用 `MarkdownView` 渲染文档正文。
- 小程序端打卡接云开发，替换 `src/store/checkin.ts` 里的 `localStorage` 实现。
- v2 再按方案接 OpenKnowledge 服务做实时编辑同步。
