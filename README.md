# 🚀 Fingerprint Nexus Pro

浏览器指纹管理系统 - 基于 Next.js 16 + TypeScript 的现代化 Web 应用

## ✨ 技术栈

This scaffold provides a robust foundation built with:

### 🎯 核心框架
- **⚡ Next.js 16** - App Router 架构
- **📘 TypeScript 5** - 全栈类型安全
- **🎨 Tailwind CSS 4** - 原子化 CSS 框架

### 🧩 UI 组件
- **🧩 shadcn/ui** - 高质量可访问组件（48 个）
- **🎯 Lucide React** - 图标库
- **🌈 Framer Motion** - 动画效果
- **🌙 next-themes** - 深浅色主题切换

### 🔄 状态管理
- **🐻 Zustand** - 客户端状态管理
- **🔄 TanStack Query** - 服务端状态管理
- **🔌 Socket.IO** - WebSocket 实时通信

### 🗄️ 数据库与 ORM
- **🗄️ Prisma** - 类型安全 ORM
- **💾 SQLite** - 开发数据库（生产可迁移 PostgreSQL）
- **📦 Redis** - 缓存 + Pub/Sub

### 🔧 外部服务集成
- **🌐 Steel API** - 浏览器自动化
- **🔍 DeviceJS** - 指纹采集
- **🤖 AI Models** - qwen/zhipu/deepseek/kimi

## 🎯 核心功能

1. **Dashboard 仪表盘** - 统计数据 + 实时日志终端
2. **指纹管理** - CRUD + 自然化 + 相似度对比
3. **模型绑定** - AI 模型 Token 管理（AES 加密存储）
4. **定时任务** - Cron 表达式调度器
5. **历史记录** - 指纹快照对比 + 回滚
6. **反检测测试** - 浏览器环境检测

## 🚀 快速启动

```bash
# 安装依赖
npm install

# 开发模式
npm run dev

# 构建生产版本
npm run build

# 启动生产服务
npm start
```

访问 http://localhost:3000

## 📁 项目结构

```
.
├── src/
│   ├── app/                 # Next.js 主应用
│   │   ├── api/             # API Routes
│   │   │   ├── fingerprint/ # 指纹管理 API
│   │   │   ├── steel/       # Steel 浏览器 API
│   │   │   ├── models/      # AI 模型绑定 API
│   │   │   └── scheduler/   # 定时任务 API
│   │   ├── page.tsx         # 主页面（单页多模块）
│   │   └── layout.tsx       # 根布局
│   ├── components/
│   │   └── ui/              # shadcn/ui 组件
│   ├── hooks/
│   │   ├── use-realtime.ts  # WebSocket Hook
│   │   └── use-toast.ts     # Toast Hook
│   └── lib/
│       ├── steel-client.ts  # Steel API 客户端
│       ├── collector.ts     # 指纹采集器
│       └── db.ts            # Prisma 实例
├── prisma/
│   └── schema.prisma        # 数据模型定义
├── mini-services/
│   └── realtime-service/    # Socket.IO 实时服务
└── package.json
```

## 🔧 开发说明

### 环境变量配置

复制 `.env.example` 到 `.env` 并填写必要配置：

```env
# Steel API
STEEL_API_URL=https://api.steel.dev/v1
STEEL_API_TOKEN=your_token_here

# DeviceJS
DEVICEJS_URL=https://devicejs.example.com

# 数据库
DATABASE_URL="file:./db/dev.db"

# Redis
REDIS_URL=redis://localhost:6379
```

### 数据库操作

```bash
# 生成 Prisma 客户端
npx prisma generate

# 数据库迁移
npx prisma migrate dev

# 查看数据库
npx prisma studio
```

## 📊 API 路由

| 端点 | 方法 | 功能 |
|------|------|------|
| `/api/fingerprint` | GET/POST | 指纹采集和管理 |
| `/api/steel` | GET/POST | Steel 浏览器会话 |
| `/api/models` | POST | AI 模型绑定 |
| `/api/scheduler` | POST | 定时任务 |
| `/api/history` | GET | 历史快照查询 |
| `/api/anti-detection` | GET | 反检测测试 |

## 🛠️ 部署建议

### Docker 部署

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npx prisma generate
RUN npm run build
EXPOSE 3000
CMD ["npm", "start"]
```

### Vercel 部署

```bash
vercel deploy --prod
```

## 📝 开发日志

详见 [worklog.md](./worklog.md)

---

Built with ❤️ using Next.js 16 + TypeScript + Prisma
