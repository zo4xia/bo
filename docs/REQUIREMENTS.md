# Fingerprint Nexus Pro - 项目需求文档

**版本**: v1.0  
**更新日期**: 2026-02-22  
**技术栈**: Next.js 16 + TypeScript + Prisma

---

## 一、核心目标

### 用户原话
> "用户只需首次登录即可，其余参数全部浏览器获取"

### 核心价值
1. **一次登录**：扫码登录 Steel 浏览器，自动获取所有参数
2. **自动指纹**：浏览器自动采集和管理指纹
3. **多模型绑定**：每个 AI 模型（千问、智谱等）独立绑定指纹
4. **定时更新**：自动定时更新指纹，保持新鲜度
5. **反检测**：从对手视角设计，绕过指纹检测

---

## 二、功能需求

### 2.1 Dashboard 仪表盘

**功能**:
- 统计数据展示（总指纹数、活跃指纹、已自然化、今日新增）
- 实时日志终端（WebSocket 推送）
- 快捷操作按钮（采集指纹、测试 Steel、清除日志）

**数据来源**:
- Prisma 数据库查询
- WebSocket 实时日志推送

---

### 2.2 指纹管理

**功能**:
- 指纹列表展示（表格 + 分页）
- 指纹详情查看（JSON 数据）
- 指纹自然化（添加噪声：Canvas/Audio/WebGL）
- 相似度计算（与原始指纹对比）
- 指纹删除/批量操作

**API 端点**: `/api/fingerprint`

---

### 2.3 模型绑定

**功能**:
- AI 模型选择（qwen/zhipu/deepseek/kimi）
- API Token 输入（AES 加密存储）
- 绑定关系管理（指纹 ↔ 模型）
- 使用次数统计

**安全要求**: Token 必须 AES-256 加密存储

**API 端点**: `/api/models`

---

### 2.4 定时任务

**功能**:
- Cron 表达式配置
- 自动更新触发器
- 更新间隔设置（每小时/每天/每周）
- 下次执行时间显示

**技术实现**: `node-cron` + Prisma SchedulerJob 表

**API 端点**: `/api/scheduler`

---

### 2.5 历史记录

**功能**:
- 指纹快照列表（时间线展示）
- 快照详情对比（Diff 视图）
- 回滚到历史版本
- 相似度趋势图

**数据模型**: `FingerprintHistory`

**API 端点**: `/api/history`

---

### 2.6 反检测测试

**功能**:
- Canvas 指纹检测
- WebGL 指纹检测
- Audio 指纹检测
- UA 一致性检查
- 时区/语言一致性检查

**API 端点**: `/api/anti-detection`

---

## 三、技术架构

### 3.1 系统架构图

```
┌─────────────────────────────────────────────────────────────┐
│                      前端层 (Next.js 16)                     │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ React 组件 (shadcn/ui)                               │   │
│  │ Zustand 状态管理                                      │   │
│  │ TanStack Query 数据获取                               │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                          ↓ API 调用
┌─────────────────────────────────────────────────────────────┐
│                   API 层 (Next.js Routes)                    │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ /api/fingerprint  → 指纹 CRUD                         │   │
│  │ /api/steel        → Steel 会话管理                    │   │
│  │ /api/models       → AI 模型绑定                       │   │
│  │ /api/scheduler    → 定时任务                          │   │
│  │ /api/history      → 历史快照                          │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                          ↓ Prisma ORM
┌─────────────────────────────────────────────────────────────┐
│                    数据层 (SQLite + Redis)                   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ SQLite (主数据库)                                     │   │
│  │ Redis (缓存 + Pub/Sub)                                │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                          ↓ 外部服务
┌─────────────────────────────────────────────────────────────┐
│                    外部服务层                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ Steel API    │  │ DeviceJS     │  │ AI Models    │     │
│  │ 浏览器自动化  │  │ 指纹采集     │  │ qwen/zhipu   │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

---

### 3.2 数据模型（Prisma Schema）

**核心模型**:

```prisma
// 指纹数据
model Fingerprint {
  id              String    @id @default(cuid())
  hash            String    @unique
  data            String    // JSON 完整数据
  userAgent       String?
  platform        String?
  similarityScore Float?
  isNaturalized   Boolean   @default(false)
  status          String    @default("active")
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt
  
  // 关联关系
  history       FingerprintHistory[]
  sessions      SteelSession[]
  modelBindings ModelBinding[]
}

// AI 模型绑定
model ModelBinding {
  id            String   @id @default(cuid())
  fingerprintId String
  fingerprint   Fingerprint @relation(fields: [fingerprintId], references: [id])
  modelName     String   // qwen, zhipu, deepseek, kimi
  apiToken      String   // AES 加密存储
  usageCount    Int      @default(0)
  status        String   @default("active")
  createdAt     DateTime @default(now())
}

// Steel 会话
model SteelSession {
  id              String   @id @default(cuid())
  sessionId       String   @unique  // Steel API 返回的 ID
  fingerprintId   String?
  fingerprint     Fingerprint? @relation(fields: [fingerprintId], references: [id])
  qrCodeUrl       String?
  sessionData     String?  // JSON
  expiresAt       DateTime?
  createdAt       DateTime @default(now())
}

// 历史快照
model FingerprintHistory {
  id            String   @id @default(cuid())
  fingerprintId String
  fingerprint   Fingerprint @relation(fields: [fingerprintId], references: [id])
  snapshot      String   // JSON 完整快照
  similarity    Float?   // 与原始指纹的相似度
  createdAt     DateTime @default(now())
}

// 定时任务
model SchedulerJob {
  id          String   @id @default(cuid())
  name        String
  cron        String   // Cron 表达式
  nextRun     DateTime
  isActive    Boolean  @default(true)
  lastRunAt   DateTime?
  createdAt   DateTime @default(now())
}

// 系统日志
model SystemLog {
  id        String   @id @default(cuid())
  level     String   // INFO/WARNING/ERROR/SUCCESS
  category  String   // FINGERPRINT/STEEL/SCHEDULER/API
  message   String
  details   String?  // JSON
  createdAt DateTime @default(now())
}
```

---

## 四、API 设计规范

### 4.1 统一响应格式

```typescript
interface ApiResponse<T = any> {
  code: number       // 200=成功，400=请求错误，500=服务器错误
  message: string    // 描述信息
  data: T           // 实际数据
  timestamp: number  // 时间戳
}
```

### 4.2 错误处理

```typescript
// 客户端错误
if (!response.ok) {
  const error = await response.json()
  throw new Error(error.message || '请求失败')
}

// Toast 提示
toast({
  title: '错误',
  description: error.message,
  variant: 'destructive'
})
```

### 4.3 请求拦截器

```typescript
// src/lib/api-client.ts
axios.interceptors.request.use(config => {
  // 添加 Token
  const token = localStorage.getItem('token')
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  // 显示 Loading
  showLoading()
  return config
})

axios.interceptors.response.use(
  res => {
    hideLoading()
    return res.data
  },
  err => {
    hideLoading()
    // 401 跳转登录
    if (err.response?.status === 401) {
      router.push('/login')
    }
    // 500 系统繁忙
    if (err.response?.status >= 500) {
      toast('系统繁忙，请稍后重试')
    }
    return Promise.reject(err)
  }
)
```

---

## 五、安全要求

### 5.1 Token 加密存储

```typescript
import CryptoJS from 'crypto-js'

// 加密
const encrypted = CryptoJS.AES.encrypt(
  apiToken,
  process.env.ENCRYPTION_KEY!
).toString()

// 解密
const bytes = CryptoJS.AES.decrypt(
  encrypted,
  process.env.ENCRYPTION_KEY!
)
const decrypted = bytes.toString(CryptoJS.enc.Utf8)
```

### 5.2 环境变量

```env
# .env
DATABASE_URL="file:./db/dev.db"

# Steel API
STEEL_API_URL=https://api.steel.dev/v1
STEEL_API_TOKEN=your_steel_token

# DeviceJS
DEVICEJS_URL=https://devicejs.example.com

# 加密密钥（32 位随机字符串）
ENCRYPTION_KEY=your_32_char_secret_key_here

# Redis
REDIS_URL=redis://localhost:6379
```

---

## 六、性能优化

### 6.1 缓存策略

```typescript
// Redis 缓存热点数据
async function getFingerprint(id: string) {
  const cacheKey = `fingerprint:${id}`
  const cached = await redis.get(cacheKey)
  
  if (cached) {
    return JSON.parse(cached)
  }
  
  const fp = await prisma.fingerprint.findUnique({ where: { id } })
  
  // 缓存 5 分钟
  await redis.setex(cacheKey, 300, JSON.stringify(fp))
  return fp
}
```

### 6.2 查询优化

```typescript
// ❌ N+1 查询
const fps = await prisma.fingerprint.findMany()
for (const fp of fps) {
  const history = await prisma.history.findMany({
    where: { fingerprintId: fp.id }
  })
}

// ✅ 预加载
const fps = await prisma.fingerprint.findMany({
  include: { 
    history: true,
    modelBindings: true
  }
})
```

### 6.3 分页加载

```typescript
// 游标分页
const fingerprints = await prisma.fingerprint.findMany({
  take: 20,
  skip: 1,
  cursor: { id: lastId },
  orderBy: { createdAt: 'desc' }
})
```

---

## 七、部署方案

### 7.1 Docker 部署

```dockerfile
FROM node:20-alpine AS base
WORKDIR /app

# 依赖安装
FROM base AS deps
COPY package*.json ./
RUN npm ci

# 构建
FROM base AS builder
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npx prisma generate
RUN npm run build

# 生产镜像
FROM base AS runner
ENV NODE_ENV=production
COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/prisma ./prisma

EXPOSE 3000
CMD ["node", "server.js"]
```

### 7.2 Vercel 部署

```bash
# 安装 Vercel CLI
npm i -g vercel

# 部署
vercel --prod
```

---

## 八、开发规范

### 8.1 代码风格

```typescript
// ✅ 使用 TypeScript 严格模式
interface Fingerprint {
  id: string
  hash: string
  data: Record<string, unknown>
}

// ✅ 使用 async/await
async function getFingerprint(id: string) {
  const fp = await prisma.fingerprint.findUnique({ where: { id } })
  return fp
}

// ❌ 避免 any
function getData(): any {}  // 不允许
function getData(): unknown {}  // 推荐
```

### 8.2 提交规范

```bash
# 功能新增
git commit -m "feat: 添加指纹自然化功能"

# Bug 修复
git commit -m "fix: 修复相似度计算错误"

# 文档更新
git commit -m "docs: 更新 API 文档"

# 代码重构
git commit -m "refactor: 重构指纹采集逻辑"
```

---

## 九、测试要求

### 9.1 单元测试（Vitest）

```typescript
// src/lib/__tests__/collector.test.ts
import { describe, it, expect } from 'vitest'
import { FingerprintCollector } from '../collector'

describe('FingerprintCollector', () => {
  it('should collect fingerprint correctly', async () => {
    const collector = new FingerprintCollector()
    const fp = await collector.collect()
    
    expect(fp.hash).toBeDefined()
    expect(fp.userAgent).toBeDefined()
  })
})
```

### 9.2 集成测试（Playwright）

```typescript
// e2e/dashboard.spec.ts
import { test, expect } from '@playwright/test'

test('dashboard loads correctly', async ({ page }) => {
  await page.goto('/')
  
  await expect(page.getByText('总指纹数')).toBeVisible()
  await expect(page.getByText('活跃指纹')).toBeVisible()
})
```

---

## 十、后续迭代计划

### Phase 1 (Week 1-2)
- [ ] 修复 P0 安全问题（Token 加密、Rate Limiting）
- [ ] 拆分 1804 行 page.tsx
- [ ] 优化 N+1 查询

### Phase 2 (Week 3-4)
- [ ] Redis 缓存层
- [ ] WebSocket 重连机制
- [ ] 数据库索引优化

### Phase 3 (Month 2)
- [ ] 单元测试（覆盖率>60%）
- [ ] API 文档（Swagger）
- [ ] CI/CD 流水线

### Phase 4 (Month 3)
- [ ] PostgreSQL 迁移
- [ ] Docker 容器化
- [ ] 国际化支持（i18n）

---

**文档结束**
