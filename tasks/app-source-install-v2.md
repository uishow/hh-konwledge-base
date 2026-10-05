---
title: App 源码落地脚本 v2
description: v1 之后执行：加返回键、待办序号、补齐其余5页、整体改版样式
type: note
status: draft
tags:
  - app
  - taro
  - install
---

在**已执行过 v1 脚本**的基础上跑本脚本，覆盖页面与样式、补齐其余 5 个页面。

用法同 v1：把本页唯一那个 `bash` 代码块整段复制粘贴到终端回车；或直接用 `awk` 从本文件抽取后执行。

源码说明见 [App 源码（导出 / 读取层 / 页面）](./app-source.md)，v1 脚本见 [App 源码一键落地脚本](./app-source-install.md)。

> [!NOTE]
> v2 的 `src/app.scss` 是**整体覆盖**（不是追加），会替换 v1 追加的样式和 Taro 默认样式。

```bash
set -e
APP=/Users/beryllewis/Documents/OpenKnowledge/hh-growth-app
cd "$APP"
mkdir -p src/components src/pages/plan src/pages/english src/pages/postgraduate src/pages/internships src/pages/family

cat > src/components/DocPage.tsx <<'EOF'
import { View, Text } from '@tarojs/components'
import Taro from '@tarojs/taro'
import MarkdownView from './MarkdownView'
import { getDoc } from '../utils/kb'

export default function DocPage({ slug, title }: { slug: string; title: string }) {
  const doc = getDoc(slug)

  return (
    <View className='page'>
      <View className='topbar'>
        <Text className='back' onClick={() => Taro.navigateBack()}>
          返回
        </Text>
        <Text className='topbar-title'>{doc?.title || title}</Text>
      </View>

      {!doc && <Text className='empty'>未找到文档：{slug}</Text>}

      {!!doc?.description && <Text className='lead'>{doc.description}</Text>}

      {doc?.sections.map((s) => (
        <View key={s.heading} className='section'>
          {s.heading !== '(概览)' && <Text className='section-title'>{s.heading}</Text>}
          <MarkdownView html={s.html} />
        </View>
      ))}
    </View>
  )
}
EOF
cat > src/pages/index/index.tsx <<'EOF'
import { useEffect, useState } from 'react'
import { View, Text } from '@tarojs/components'
import Taro from '@tarojs/taro'
import { allTodos, generatedAt } from '../../utils/kb'

const KEY_DATES = [
  { label: '英语四级口试', date: '2026-11-21' },
  { label: '英语四级笔试', date: '2026-12-12' }
]

const NAV = [
  { label: '课表', url: '/pages/timetable/timetable' },
  { label: '四年总计划', url: '/pages/plan/plan' },
  { label: '英语', url: '/pages/english/english' },
  { label: '升学', url: '/pages/postgraduate/postgraduate' },
  { label: '实践', url: '/pages/internships/internships' },
  { label: '家庭约定', url: '/pages/family/family' },
  { label: '待办与打卡', url: '/pages/todos/todos' }
]

export default function Index() {
  const [todos, setTodos] = useState(allTodos())

  useEffect(() => {
    setTodos(allTodos())
  }, [])

  const pending = todos.filter((t) => !t.checked)
  const done = todos.length - pending.length

  return (
    <View className='page'>
      <View className='hero'>
        <Text className='hero-title'>HH 成长工作台</Text>
        <Text className='hero-sub'>
          共 {todos.length} 条事项，已完成 {done} 条 · 数据 {generatedAt().slice(0, 10)}
        </Text>
      </View>

      <Text className='block-title'>关键日期</Text>
      <View className='date-grid'>
        {KEY_DATES.map((d) => (
          <View key={d.date} className='date-card'>
            <Text className='date-label'>{d.label}</Text>
            <Text className='date-value'>{d.date}</Text>
          </View>
        ))}
      </View>

      <Text className='block-title'>导航</Text>
      <View className='nav-grid'>
        {NAV.map((n) => (
          <View
            key={n.url}
            className='nav-card'
            onClick={() => Taro.navigateTo({ url: n.url })}
          >
            <Text className='nav-label'>{n.label}</Text>
          </View>
        ))}
      </View>

      <Text className='block-title'>近期待办（{pending.length}）</Text>
      <View className='panel'>
        {pending.slice(0, 8).map((t, i) => (
          <View key={t.id} className='row'>
            <Text className='row-index'>{i + 1}</Text>
            <View className='row-main'>
              <Text className='row-text'>{t.text}</Text>
              <Text className='row-from'>{t.docTitle}</Text>
            </View>
          </View>
        ))}
        {!pending.length && <Text className='empty'>没有未完成事项</Text>}
      </View>
    </View>
  )
}
EOF
cat > src/pages/timetable/timetable.tsx <<'EOF'
import { View, Text } from '@tarojs/components'
import Taro from '@tarojs/taro'
import { getTimetable } from '../../utils/kb'

export default function Timetable() {
  const { courses } = getTimetable()
  const columns = courses.length ? Object.keys(courses[0]) : []

  return (
    <View className='page'>
      <View className='topbar'>
        <Text className='back' onClick={() => Taro.navigateBack()}>
          返回
        </Text>
        <Text className='topbar-title'>课表</Text>
      </View>

      {!courses.length && <Text className='empty'>kb.json 里没有解析到课表表格</Text>}

      {!!courses.length && (
        <View className='panel'>
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
        </View>
      )}
    </View>
  )
}
EOF

cat > src/pages/todos/todos.tsx <<'EOF'
import { useEffect, useState } from 'react'
import { View, Text } from '@tarojs/components'
import Taro from '@tarojs/taro'
import { allTodos } from '../../utils/kb'
import { getCheckin, setCheckin } from '../../store/checkin'

export default function Todos() {
  const [todos, setTodos] = useState(allTodos())

  useEffect(() => {
    setTodos(allTodos())
  }, [])

  const isChecked = (t: { id: string; checked: boolean }) =>
    getCheckin(t.id)?.checked ?? t.checked

  const toggle = (id: string, checked: boolean) => {
    const next = !checked
    setCheckin(id, { checked: next, date: new Date().toISOString().slice(0, 10) })
    setTodos((prev) => prev.map((t) => (t.id === id ? { ...t, checked: next } : t)))
  }

  const doneCount = todos.filter((t) => isChecked(t)).length

  return (
    <View className='page'>
      <View className='topbar'>
        <Text className='back' onClick={() => Taro.navigateBack()}>
          返回
        </Text>
        <Text className='topbar-title'>待办与打卡</Text>
      </View>

      <Text className='block-title'>
        已完成 {doneCount} / {todos.length}
      </Text>

      <View className='panel'>
        {todos.map((t, i) => (
          <View
            key={t.id}
            className={'row clickable' + (isChecked(t) ? ' done' : '')}
            onClick={() => toggle(t.id, isChecked(t))}
          >
            <Text className='row-index'>{i + 1}</Text>
            <Text className={'box' + (isChecked(t) ? ' on' : '')}>
              {isChecked(t) ? '●' : '○'}
            </Text>
            <View className='row-main'>
              <Text className='row-text'>{t.text}</Text>
              <Text className='row-from'>
                {t.docTitle} · {t.section}
              </Text>
            </View>
          </View>
        ))}
        {!todos.length && <Text className='empty'>知识库里没有解析到勾选框</Text>}
      </View>
    </View>
  )
}
EOF
cat > src/pages/plan/plan.tsx <<'EOF'
import DocPage from '../../components/DocPage'

export default function Plan() {
  return <DocPage slug='growth/roadmap' title='四年总计划' />
}
EOF

cat > src/pages/english/english.tsx <<'EOF'
import DocPage from '../../components/DocPage'

export default function English() {
  return <DocPage slug='growth/english' title='英语计划' />
}
EOF

cat > src/pages/postgraduate/postgraduate.tsx <<'EOF'
import DocPage from '../../components/DocPage'

export default function Postgraduate() {
  return <DocPage slug='growth/postgraduate' title='升学探索' />
}
EOF

cat > src/pages/internships/internships.tsx <<'EOF'
import DocPage from '../../components/DocPage'

export default function Internships() {
  return <DocPage slug='growth/internships' title='假期实践' />
}
EOF

cat > src/pages/family/family.tsx <<'EOF'
import DocPage from '../../components/DocPage'

export default function Family() {
  return <DocPage slug='growth/family' title='家庭共同约定' />
}
EOF
cp src/app.config.ts src/app.config.ts.bak2 2>/dev/null || true

cat > src/app.config.ts <<'EOF'
export default defineAppConfig({
  pages: [
    'pages/index/index',
    'pages/timetable/timetable',
    'pages/todos/todos',
    'pages/plan/plan',
    'pages/english/english',
    'pages/postgraduate/postgraduate',
    'pages/internships/internships',
    'pages/family/family'
  ],
  window: {
    backgroundTextStyle: 'light',
    navigationBarBackgroundColor: '#fff',
    navigationBarTitleText: 'HH 成长',
    navigationBarTextStyle: 'black'
  }
})
EOF
cat > src/app.scss <<'EOF'
page {
  background: #f5f7fa;
}

.page {
  max-width: 1080px;
  margin: 0 auto;
  padding: 0 16px 48px;
}

.topbar {
  display: flex;
  align-items: center;
  padding: 14px 0 12px;
  border-bottom: 1px solid #e6e8eb;
  margin-bottom: 16px;
}

.back {
  font-size: 14px;
  color: #1677ff;
  padding: 6px 14px;
  border: 1px solid #cfe0fb;
  background: #f0f6ff;
  border-radius: 999px;
}

.topbar-title {
  margin-left: 12px;
  font-size: 18px;
  font-weight: 600;
  color: #1f2329;
}

.hero {
  padding: 28px 24px;
  margin: 16px 0 24px;
  background: #ffffff;
  border-radius: 14px;
  box-shadow: 0 2px 10px rgba(31, 35, 41, 0.06);
}

.hero-title {
  display: block;
  font-size: 30px;
  font-weight: 700;
  color: #1f2329;
}

.hero-sub {
  display: block;
  margin-top: 8px;
  font-size: 13px;
  color: #8a919f;
}

.block-title {
  display: block;
  font-size: 15px;
  font-weight: 600;
  color: #4e5969;
  margin: 24px 0 12px;
}

.date-grid {
  display: flex;
  flex-wrap: wrap;
}

.date-card {
  flex: 1;
  min-width: 200px;
  padding: 16px 18px;
  margin-right: 12px;
  background: #ffffff;
  border-left: 4px solid #d4380d;
  border-radius: 10px;
  box-shadow: 0 1px 6px rgba(31, 35, 41, 0.06);
}

.date-label {
  display: block;
  font-size: 14px;
  color: #4e5969;
}

.date-value {
  display: block;
  margin-top: 6px;
  font-size: 20px;
  font-weight: 700;
  color: #d4380d;
}

.nav-grid {
  display: flex;
  flex-wrap: wrap;
}

.nav-card {
  width: 160px;
  padding: 18px 16px;
  margin: 0 12px 12px 0;
  background: #ffffff;
  border: 1px solid #eceef1;
  border-radius: 12px;
  box-shadow: 0 1px 6px rgba(31, 35, 41, 0.05);
}

.nav-label {
  font-size: 15px;
  font-weight: 600;
  color: #1f2329;
}
.panel {
  background: #ffffff;
  border: 1px solid #eceef1;
  border-radius: 12px;
  box-shadow: 0 1px 6px rgba(31, 35, 41, 0.05);
  overflow: hidden;
}

.row {
  display: flex;
  align-items: flex-start;
  padding: 12px 16px;
  border-bottom: 1px solid #f2f3f5;
}

.row:last-child {
  border-bottom: none;
}

.row.clickable {
  cursor: pointer;
}

.row-index {
  width: 26px;
  font-size: 13px;
  color: #a9aeb8;
}

.box {
  width: 22px;
  font-size: 14px;
  color: #c9cdd4;
  font-weight: 700;
}

.box.on {
  color: #00b42a;
}

.row-main {
  flex: 1;
  display: flex;
  flex-direction: column;
}

.row-text {
  font-size: 15px;
  color: #1f2329;
  line-height: 1.5;
}

.row-from {
  margin-top: 4px;
  font-size: 12px;
  color: #a9aeb8;
}

.row.done .row-text {
  color: #a9aeb8;
  text-decoration: line-through;
}

.section {
  background: #ffffff;
  border: 1px solid #eceef1;
  border-radius: 12px;
  padding: 18px 20px;
  margin-bottom: 14px;
  box-shadow: 0 1px 6px rgba(31, 35, 41, 0.05);
}

.section-title {
  display: block;
  font-size: 17px;
  font-weight: 600;
  color: #1f2329;
  margin-bottom: 10px;
  padding-bottom: 8px;
  border-bottom: 1px solid #f2f3f5;
}

.lead {
  display: block;
  font-size: 14px;
  color: #4e5969;
  margin-bottom: 14px;
}

.empty {
  display: block;
  padding: 24px;
  text-align: center;
  font-size: 14px;
  color: #a9aeb8;
}

.markdown-body {
  font-size: 14px;
  line-height: 1.8;
  color: #33383f;
}

.table {
  border-radius: 8px;
  overflow: hidden;
}

.tr {
  display: flex;
}

.tr.head {
  background: #f7f8fa;
}

.th,
.td {
  flex: 1;
  padding: 10px 12px;
  font-size: 13px;
}

.th {
  font-weight: 600;
  color: #4e5969;
}

.td {
  color: #33383f;
}

EOF
echo 'OK v2 文件已写入。接下来执行:'
echo '  node scripts/export.mjs'
echo '  npm run dev:h5'
```
