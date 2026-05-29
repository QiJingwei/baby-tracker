# 🌸 宝宝日记 — Claude Code 项目说明

## 项目概览

家庭宝宝记录 PWA，供家人共同记录宝宝日常数据（喂养、睡眠、便便、辅食等）。数据存储在 Supabase，多端实时同步，可安装到手机桌面使用。

**⚠️ 整个应用是单文件：`index.html`，无 node_modules，无构建流程，直接编辑即可。**

### 技术栈

| 层级 | 技术 |
|------|------|
| 前端框架 | Vue 3（CDN 版，无构建步骤） |
| 数据库 | Supabase（PostgreSQL + 实时订阅） |
| 部署 | Vercel（连接 GitHub 自动部署） |
| PWA | manifest.json + Service Worker |
| AI 识别 | Claude API（claude-sonnet-4-20250514） |

---

## 仓库 & 部署

### 文件结构

```
index.html      # 主应用，所有逻辑都在这里
manifest.json   # PWA 配置
sw.js           # Service Worker，支持离线
```

### Supabase

- **Project URL**：`https://xagaahvgvkwshmcigild.supabase.co`
- **anon key（前端使用）**：
  ```
  eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InhhZ2FhaHZndmt3c2htY2lnaWxkIiwicm9sZSI6ImFub24iLCJpYXQiOjE3Nzk5NTQ3MzQsImV4cCI6MjA5NTUzMDczNH0.DeagiWOyPYc_xXhaDTpPGmq5km98cbSgVaoTNH3OvVw
  ```
- **⚠️ service_role key 权限极高，不要放进前端代码**

### Vercel 部署

- 连接 GitHub 仓库，Framework 选 **Other**（无构建步骤）
- Push 代码后 Vercel 自动 1 分钟内完成部署

---

## 数据库表结构

表名：`records`（单表，`type` 字段区分记录类型）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint generated always as identity primary key | 主键 |
| type | text not null | feed/sleep/food/poop/play/mood/weight/note/bath |
| detail | text | 展示文本，格式见下方约定 |
| recorder | text | 爸爸/妈妈/爷爷/奶奶 |
| time | timestamptz | 记录时间（排序用） |
| date | date | 冗余字段，用于按天过滤 |
| start_time | timestamptz | 仅 sleep 类型：开始时间 |
| end_time | timestamptz | 仅 sleep 类型：结束时间 |
| created_at | timestamptz default now() | 创建时间 |

### detail 字段格式约定

```
feed：   配方奶 · 205ml · 10分钟
sleep：  小睡（31分钟）   [时段从 start_time/end_time 读取]
food：   蛋黄、南瓜 · 50g · 很喜欢 😍
poop：   黄色 · 糊状 · 正常
bath：   38°C · 洗完抚触
weight： 体重 6.5kg · 身长 65cm
note：   自由文本
```

**⚠️ 辅食食材用「、」分隔（不是逗号），历史页聚合功能依赖这个分隔符。**

---

## 应用功能模块

### Tab 结构

```
📖 日记  |  📚 历史  |  👶 宝宝（暂未实现）  |  ⚙️ 设置
```

另有 **import 页**（从设置进入，不在 Tab 栏）。

### 日记页

- 顶部 Header：今日睡眠 / 喂养 / 便便统计
- 睡眠横幅：睡眠进行中时全局显示计时（关掉页面也保持，用 localStorage 存时间戳）
- 9 个快速记录按钮：睡眠 / 喂养 / 辅食 / 便便 / 游玩 / 心情 / 体重 / 备注 / 洗澡
- 时间线：按日期浏览，升序排列，支持左右翻日期

### 历史页

- 类型筛选（全部 / 辅食 / 便便 / 睡眠 / 喂养 / 体重 / 游玩 / 心情）
- 时间范围：近 7 天 / 近 30 天 / 全部
- 辅食模式：食材聚合（次数 + 接受程度 + 最近日期）
- 便便模式：自动关联前 24h 辅食，异常便便高亮（绿 / 红 / 稀水样 / 血丝）

### 导入页（拍照批量导入）

- 批量上传照片，每张独立确认日期（**重要：AI 不判断年份，必须手动填**）
- 支持月嫂服务日志 & 宝宝喂养记录表两种格式，AI 自动判断
- 识别结果预览，可逐条勾选 / 取消
- 确认后批量写入 Supabase

---

## 关键代码位置（index.html 内）

| 位置 | 说明 |
|------|------|
| 文件顶部 `SUPABASE_URL` / `SUPABASE_KEY` 常量 | Supabase 初始化 |
| `// ── 工具函数` 注释块 | `nowTime()` 函数定义处，**必须在所有 `ref({...nowTime()...})` 之前** |
| `// ── 表单 ──` 注释块 | 9 个表单声明，含 `noteForm` |
| `sleepStartTs` | 睡眠计时持久化，存 `localStorage("sleep_start_ts")` |
| `insertRecord(payload)` | 写入记录，乐观更新 + 失败回滚 |
| `subscribeRealtime()` | 实时订阅，监听 INSERT/DELETE |
| `historyFiltered` → `historyGrouped` computed | 历史筛选逻辑 |
| `foodAggList` computed | 辅食聚合，解析 detail 的「、」分隔 |
| `linkedFoods(poopRecord)` | 便便关联，查前 24h 所有 food 记录 |
| `parseImage(item)` | AI 识别导入，调用 Anthropic API |

---

## 已知 Bug

### Bug 1：白屏（已定位）

**原因**：`noteForm` 声明遗漏；`nowTime()` 调用时机早于函数定义。

**修复**：
1. 确认 `nowTime` 函数在 `// ── 工具函数` 注释后定义，且在所有 `ref()` 初始化之前
2. 在 `// ── 表单 ──` 注释块里确认 `noteForm` 存在：
   ```js
   const noteForm = ref({ content: "" })
   ```

### Bug 2：导入页响应式不更新

**原因**：`item.status` / `item.records` 直接赋值，Vue 3 Proxy 可能不追踪。

**修复**：用 `splice` 替换整个对象触发响应式：
```js
const idx = importQueue.value.findIndex(i => i.id === item.id)
importQueue.value.splice(idx, 1, { ...item, status: "done", records: parsed })
```

---

## 待完成功能

- [ ] 宝宝 Tab（宝宝基本信息：姓名、生日、性别）
- [ ] 历史页洗澡筛选（`historyFilters` 加 `bath` 类型）
- [ ] 统计图表（暂无需求，备用）
- [ ] 导入页响应式 Bug 修复（Bug 2）

---

## 本地开发

- **无需 Node.js**，VS Code + Live Server 插件直接打开 `index.html` 预览
- Supabase 连线上库，本地改完推 GitHub，Vercel 自动部署
- 如需脱机开发，可临时把 `fetchRecords` 的结果 mock 成本地数组

---

## 启动新任务时的建议提示词

```
这是一个宝宝记录 PWA，单文件 Vue 3（index.html），接入 Supabase。
请先检查 setup() 函数里：
1. nowTime 函数是否在所有 ref({...nowTime()...}) 之前定义
2. noteForm 是否在 // ── 表单 ── 块里声明
3. importQueue 的响应式更新是否用 splice 替换

然后帮我实现：[你的需求]
```
