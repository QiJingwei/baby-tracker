# 宝宝日记 - 部署指南

## 当前状态
这是 **原型版本**，数据存在浏览器 localStorage（本地，不同设备不互通）。
接入 Supabase 后可实现家人共享。

---

## 第一步：上传到 GitHub

1. 在 GitHub 新建仓库，例如 `baby-tracker`（建议设为 Public）
2. 将以下三个文件上传到仓库根目录：
   - `index.html`
   - `manifest.json`
   - `sw.js`

---

## 第二步：Vercel 部署

1. 打开 https://vercel.com，用 GitHub 账号登录
2. 点击 "New Project" → Import 刚才的仓库
3. 框架选 **Other**（无需构建，纯静态）
4. 点击 Deploy，等待约 1 分钟
5. 部署完成后获得域名，例如 `baby-tracker-xxx.vercel.app`

---

## 第三步：接入 Supabase（多设备共享）

### 3.1 创建 Supabase 项目
1. 打开 https://supabase.com，注册免费账号
2. 新建项目（选亚洲节点，如 Singapore）
3. 记录 `Project URL` 和 `anon public key`

### 3.2 创建数据库表
在 Supabase SQL Editor 执行：

```sql
-- 宝宝信息表
create table baby (
  id uuid default gen_random_uuid() primary key,
  name text not null,
  birthday date,
  created_at timestamptz default now()
);

-- 记录主表
create table records (
  id uuid default gen_random_uuid() primary key,
  baby_id uuid references baby(id),
  type text not null,  -- sleep/feed/food/poop/play/mood/weight/note
  detail text,
  recorded_at timestamptz default now(),
  date date default current_date,
  created_by text  -- 记录者（妈妈/爸爸/奶奶等）
);

-- 开放匿名读写（家庭内共享，无需登录）
alter table records enable row level security;
create policy "allow all" on records for all using (true);
alter table baby enable row level security;
create policy "allow all" on baby for all using (true);
```

### 3.3 修改 index.html 接入 Supabase
在 `<head>` 里加入：
```html
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
```

在 `<script>` 顶部加入：
```javascript
const supabase = window.supabase.createClient(
  'https://你的项目ID.supabase.co',
  '你的anon-key'
)
```

然后修改 `addRecord()` 函数，同时写入 Supabase：
```javascript
async function addRecord(type, detail) {
  const r = {
    type,
    detail,
    recorded_at: new Date().toISOString(),
    date: new Date().toISOString().slice(0, 10)
  }
  // 写入 Supabase
  await supabase.from('records').insert(r)
  // 保留本地缓存...
}
```

---

## 安装为手机 App（PWA）

### iOS（Safari）
1. 用 Safari 打开你的 Vercel 网址
2. 点击底部分享按钮 →「添加到主屏幕」
3. 点击「添加」，桌面上会出现图标

### Android（Chrome）
1. 用 Chrome 打开网址
2. 地址栏右侧会出现「安装」提示，或进菜单 →「添加到主屏幕」

---

## 文件说明

| 文件 | 说明 |
|------|------|
| index.html | 主应用，所有功能都在这里 |
| manifest.json | PWA 配置，定义图标、主题色等 |
| sw.js | Service Worker，支持离线使用 |
