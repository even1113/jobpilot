# JobPilot 求职全流程管理工具 - 技术设计文档（TDD）

> 版本：v1.0  
> 日期：2026-06-27  
> 状态：草案（待评审）  
> 关联文档：[产品需求文档 PRD](./PRD_JobPilot_v1.0.md)

---

## 一、文档概述

### 1.1 文档目的

本文档定义 JobPilot 求职工具的**技术实现方案**，明确"怎么做"，包括技术选型、系统架构、数据模型、接口设计、部署方案等。

产品需求（"做什么"）见 [产品需求文档 PRD](./PRD_JobPilot_v1.0.md)。

### 1.2 读者对象

- 开发工程师
- 架构师
- 运维工程师
- 测试工程师

### 1.3 设计原则

| 原则 | 说明 |
|------|------|
| 职责分离 | PRD 管需求，TDD 管实现，变更互不干扰 |
| 轻量优先 | MVP 阶段选择成熟、轻量的技术栈，避免过度设计 |
| 渐进演进 | 架构支持从单体到微服务的平滑演进 |
| 安全合规 | 简历等敏感数据加密存储，全链路 HTTPS |

---

## 二、技术选型

### 2.1 选型总览

| 层级 | 技术 | 版本 | 选型理由 |
|------|------|------|----------|
| 小程序框架 | Taro | 3.x | 一套代码编译到微信小程序+H5，React 生态 |
| Web 前端 | React + Vite + Ant Design | React 18, Vite 5, AntD 5 | 与 Taro 共享业务逻辑，开发效率高 |
| 后端框架 | FastAPI | 0.100+ | 异步高性能，自动生成 OpenAPI 文档，AI 集成方便 |
| ORM | SQLAlchemy | 2.0+ | 类型安全，异步支持，社区成熟 |
| 数据库 | PostgreSQL | 15+ | JSONB 支持，适合存储灵活结构（简历、AI 结果） |
| 缓存 | Redis | 7+ | 会话管理、热门岗位缓存、Celery Broker |
| 任务队列 | Celery | 5+ | 爬虫定时任务、面试提醒推送 |
| AI 服务 | GLM API | - | 简历诊断、JD 匹配 |
| 文件存储 | MinIO / OSS | - | 简历文件存储 |
| 容器化 | Docker Compose | - | 初期轻量部署，后续可迁移 K8s |
| 反向代理 | Nginx | - | HTTPS 终止、静态资源、负载均衡 |

### 2.2 选型对比与决策

#### 后端框架对比

| 维度 | FastAPI | Django | Flask | Node.js (Express) |
|------|---------|--------|-------|-------------------|
| 异步支持 | 原生 async/await | Django 4+ 支持 | 需扩展 | 原生 |
| AI 集成 | Python 生态最佳 | 同左 | 同左 | 需额外调用 |
| API 文档 | 自动 OpenAPI | 需 drf-spectacular | 需 flask-restx | 需 swagger-jsdoc |
| 性能 | 高 | 中 | 中 | 高 |
| 学习曲线 | 低 | 中 | 低 | 低（前端团队） |

**决策**：选择 FastAPI，核心优势是 Python 生态对 AI 集成最友好，且原生异步性能优秀。

#### 小程序框架对比

| 维度 | Taro 3 | uni-app | 原生微信小程序 |
|------|--------|---------|---------------|
| 跨端能力 | 微信/H5/RN | 全平台 | 仅微信 |
| 技术栈 | React | Vue | WXML/WXSS |
| 生态 | React 生态 | Vue 生态 | 微信原生 |
| 性能 | 接近原生 | 接近原生 | 原生最优 |

**决策**：选择 Taro 3，React 技术栈与 Web 端统一，跨端能力满足需求。

---

## 三、系统架构

### 3.1 整体架构

```
┌──────────────┐  ┌──────────────┐
│  微信小程序    │  │   Web 端     │
│  (Taro 3)    │  │  (React)     │
└──────┬───────┘  └──────┬───────┘
       │                 │
       └────────┬────────┘
                │ HTTPS / REST API
       ┌────────▼────────┐
       │   Nginx 反向代理  │
       └────────┬────────┘
                │
       ┌────────▼────────┐
       │  FastAPI 应用     │
       │  ┌─────────────┐│
       │  │ 认证中间件    ││
       │  ├─────────────┤│
       │  │ 路由层       ││
       │  ├─────────────┤│
       │  │ 服务层       ││
       │  ├─────────────┤│
       │  │ 数据访问层    ││
       │  └─────────────┘│
       └──┬──────┬───────┘
          │      │
   ┌──────▼┐  ┌─▼────────┐
   │PostgreSQL│ │  Redis   │
   └─────────┘ └──────────┘
                │
       ┌────────▼────────┐
       │  Celery Worker   │
       │  ┌─────────────┐│
       │  │ 爬虫任务      ││
       │  │ 提醒推送      ││
       │  └─────────────┘│
       └────────┬────────┘
                │
       ┌────────▼────────┐
       │   GLM API        │
       └─────────────────┘
```

### 3.2 分层架构

```
┌─────────────────────────────────┐
│           API 路由层             │  请求校验、权限检查、响应格式化
├─────────────────────────────────┤
│           服务层                 │  业务逻辑编排、事务管理
├─────────────────────────────────┤
│           数据访问层             │  ORM 操作、数据查询
├─────────────────────────────────┤
│           基础设施层             │  DB/Redis 连接、外部 API 调用
└─────────────────────────────────┘
```

### 3.3 目录结构

```
jobpilot/
├── backend/                    # FastAPI 后端
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py            # FastAPI 入口
│   │   ├── config.py          # 配置管理（环境变量）
│   │   ├── database.py        # DB 连接与会话管理
│   │   ├── models/            # SQLAlchemy ORM 模型
│   │   │   ├── __init__.py
│   │   │   ├── user.py
│   │   │   ├── job.py
│   │   │   ├── application.py
│   │   │   └── resume.py
│   │   ├── schemas/           # Pydantic 请求/响应模型
│   │   │   ├── __init__.py
│   │   │   ├── user.py
│   │   │   ├── job.py
│   │   │   ├── application.py
│   │   │   └── resume.py
│   │   ├── api/               # 路由层
│   │   │   ├── __init__.py
│   │   │   ├── deps.py        # 依赖注入（当前用户、DB 会话）
│   │   │   ├── auth.py
│   │   │   ├── jobs.py
│   │   │   ├── applications.py
│   │   │   ├── resumes.py
│   │   │   └── ai.py
│   │   ├── services/          # 服务层
│   │   │   ├── __init__.py
│   │   │   ├── auth_service.py
│   │   │   ├── job_service.py
│   │   │   ├── application_service.py
│   │   │   ├── resume_service.py
│   │   │   └── ai_service.py
│   │   └── utils/             # 工具函数
│   │       ├── __init__.py
│   │       ├── security.py    # JWT、密码哈希
│   │       └── file_parser.py # 简历文件解析
│   ├── crawlers/              # 爬虫模块
│   │   ├── __init__.py
│   │   ├── base.py            # 爬虫基类
│   │   ├── boss.py            # Boss 直聘
│   │   └── lagou.py           # 拉勾
│   ├── tasks/                 # Celery 任务
│   │   ├── __init__.py
│   │   ├── crawl_tasks.py     # 定时爬取
│   │   └── reminder_tasks.py  # 面试提醒
│   ├── alembic/               # 数据库迁移
│   │   └── versions/
│   ├── alembic.ini
│   ├── requirements.txt
│   ├── Dockerfile
│   └── .env.example
├── frontend/                   # Web 端 (React)
│   ├── src/
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx  # 看板主页
│   │   │   ├── Jobs.tsx       # 岗位搜索
│   │   │   ├── Resume.tsx     # 简历管理
│   │   │   └── Login.tsx      # 登录
│   │   ├── components/
│   │   │   ├── KanbanBoard.tsx
│   │   │   ├── JobCard.tsx
│   │   │   └── DiagnosisResult.tsx
│   │   ├── api/               # API 调用封装
│   │   │   ├── client.ts
│   │   │   ├── jobs.ts
│   │   │   ├── applications.ts
│   │   │   └── ai.ts
│   │   ├── hooks/             # 自定义 Hooks
│   │   ├── store/             # 状态管理 (Zustand)
│   │   ├── App.tsx
│   │   └── main.tsx
│   ├── package.json
│   └── vite.config.ts
├── miniapp/                    # 微信小程序 (Taro)
│   ├── src/
│   │   ├── pages/
│   │   │   ├── index/         # 首页看板
│   │   │   ├── jobs/          # 岗位列表
│   │   │   ├── detail/        # 岗位详情
│   │   │   └── profile/       # 个人中心
│   │   ├── components/
│   │   ├── services/          # API 调用
│   │   └── app.config.ts
│   ├── package.json
│   └── project.config.json
├── docs/                       # 文档
│   ├── PRD_JobPilot_v1.0.md
│   └── TDD_JobPilot_v1.0.md
└── docker-compose.yml
```

---

## 四、数据模型

### 4.1 ER 关系

```
User 1──N Resume
User 1──N Application
Job  1──N Application
Application 1──N StatusLog
```

### 4.2 表结构定义

#### users 用户表

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | 主键 |
| openid | VARCHAR(64) | UNIQUE, NOT NULL | 微信 openid |
| union_id | VARCHAR(64) | UNIQUE | 微信 unionid（跨小程序） |
| nickname | VARCHAR(64) | | 昵称 |
| avatar_url | VARCHAR(512) | | 头像 URL |
| phone | VARCHAR(20) | | 手机号（脱敏） |
| created_at | TIMESTAMP | NOT NULL, DEFAULT NOW | 创建时间 |
| updated_at | TIMESTAMP | NOT NULL, DEFAULT NOW | 更新时间 |

#### jobs 岗位表

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | 主键 |
| source | VARCHAR(20) | NOT NULL | 来源平台（boss/lagou） |
| source_id | VARCHAR(64) | NOT NULL | 平台原始 ID |
| title | VARCHAR(128) | NOT NULL | 岗位名称 |
| company | VARCHAR(128) | NOT NULL | 公司名称 |
| salary_min | INTEGER | | 最低薪资（K/月） |
| salary_max | INTEGER | | 最高薪资（K/月） |
| salary_months | INTEGER | | 薪资月数（如 13、14） |
| location | VARCHAR(64) | | 城市 |
| district | VARCHAR(64) | | 区/商圈 |
| experience | VARCHAR(32) | | 经验要求 |
| education | VARCHAR(32) | | 学历要求 |
| description | TEXT | | 岗位描述 |
| requirements | TEXT | | 任职要求 |
| company_size | VARCHAR(32) | | 公司规模 |
| company_industry | VARCHAR(64) | | 行业 |
| url | VARCHAR(512) | | 原始链接 |
| published_at | TIMESTAMP | | 发布时间 |
| crawled_at | TIMESTAMP | NOT NULL | 抓取时间 |
| UNIQUE(source, source_id) | | | 联合唯一约束 |

#### applications 投递记录表

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | 主键 |
| user_id | UUID | FK→users, NOT NULL | 用户 ID |
| job_id | UUID | FK→jobs | 岗位 ID（可为空，支持手动添加非平台岗位） |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'pending' | 状态枚举 |
| notes | TEXT | | 备注 |
| interview_time | TIMESTAMP | | 面试时间 |
| match_score | INTEGER | | AI 匹配评分 |
| match_detail | JSONB | | AI 匹配详情 |
| applied_at | TIMESTAMP | | 投递时间 |
| created_at | TIMESTAMP | NOT NULL, DEFAULT NOW | 创建时间 |
| updated_at | TIMESTAMP | NOT NULL, DEFAULT NOW | 更新时间 |

**status 枚举值**：`pending`（待投递）、`applied`（已投递）、`interview`（面试中）、`offer`（已Offer）、`ended`（已结束）

#### resumes 简历表

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | 主键 |
| user_id | UUID | FK→users, NOT NULL | 用户 ID |
| name | VARCHAR(64) | NOT NULL | 简历名称/版本 |
| content | JSONB | | 结构化简历内容 |
| raw_text | TEXT | | 纯文本（供 AI 分析） |
| file_url | VARCHAR(512) | | 原始文件 URL（MinIO/OSS） |
| is_default | BOOLEAN | DEFAULT FALSE | 是否默认简历 |
| last_diagnosis | JSONB | | 最近一次诊断结果 |
| created_at | TIMESTAMP | NOT NULL, DEFAULT NOW | 创建时间 |
| updated_at | TIMESTAMP | NOT NULL, DEFAULT NOW | 更新时间 |

#### status_logs 状态变更日志表

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | 主键 |
| application_id | UUID | FK→applications, NOT NULL | 投递记录 ID |
| from_status | VARCHAR(20) | | 原状态（首次创建时为 NULL） |
| to_status | VARCHAR(20) | NOT NULL | 新状态 |
| created_at | TIMESTAMP | NOT NULL, DEFAULT NOW | 变更时间 |

### 4.3 索引设计

| 表 | 索引 | 类型 | 说明 |
|----|------|------|------|
| jobs | (source, source_id) | UNIQUE | 去重 |
| jobs | (title, company) | BTREE | 搜索优化 |
| jobs | (location, salary_min) | BTREE | 筛选优化 |
| applications | (user_id, status) | BTREE | 看板查询 |
| applications | (user_id, updated_at) | BTREE | 列表排序 |
| applications | (interview_time) | BTREE | 面试提醒查询 |
| resumes | (user_id, is_default) | BTREE | 默认简历查询 |
| status_logs | (application_id, created_at) | BTREE | 变更历史查询 |

---

## 五、接口设计

### 5.1 接口规范

- 基础路径：`/api/v1`
- 认证方式：Bearer Token（JWT）
- 响应格式：统一 JSON 封装

**成功响应：**
```json
{
  "data": { ... }
}
```

**分页响应：**
```json
{
  "data": {
    "total": 128,
    "page": 1,
    "page_size": 20,
    "items": [ ... ]
  }
}
```

**错误响应：**
```json
{
  "error": {
    "code": "INVALID_STATUS_TRANSITION",
    "message": "不允许从 ended 状态回退"
  }
}
```

### 5.2 接口清单

| 方法 | 路径 | 说明 | 认证 |
|------|------|------|------|
| POST | `/api/v1/auth/wx-login` | 微信登录 | 否 |
| GET | `/api/v1/auth/qr-code` | 获取登录小程序码 | 否 |
| GET | `/api/v1/jobs` | 搜索岗位 | 是 |
| GET | `/api/v1/jobs/{id}` | 岗位详情 | 是 |
| POST | `/api/v1/applications` | 新增投递记录 | 是 |
| GET | `/api/v1/applications` | 获取投递记录列表 | 是 |
| GET | `/api/v1/applications/board` | 获取看板视图数据 | 是 |
| PATCH | `/api/v1/applications/{id}` | 更新投递记录 | 是 |
| PATCH | `/api/v1/applications/{id}/status` | 更新投递状态 | 是 |
| DELETE | `/api/v1/applications/{id}` | 删除投递记录 | 是 |
| POST | `/api/v1/resumes` | 上传/创建简历 | 是 |
| GET | `/api/v1/resumes` | 获取简历列表 | 是 |
| GET | `/api/v1/resumes/{id}` | 获取简历详情 | 是 |
| DELETE | `/api/v1/resumes/{id}` | 删除简历 | 是 |
| POST | `/api/v1/ai/resume-diagnose` | AI 简历诊断 | 是 |
| POST | `/api/v1/ai/jd-match` | AI JD 匹配评分 | 是 |
| GET | `/api/v1/interviews/upcoming` | 获取即将到来的面试 | 是 |

### 5.3 核心接口详细定义

#### POST /api/v1/auth/wx-login

**请求：**
```json
{
  "code": "string  // 微信 wx.login 获取的 code"
}
```

**响应：**
```json
{
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIs...",
    "token_type": "bearer",
    "expires_in": 604800,
    "user": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "nickname": "张三",
      "avatar_url": "https://thirdwx.qlogo.cn/..."
    }
  }
}
```

**实现要点：**

- 后端用 code 调用微信 `sns/jscode2session` 接口换取 openid + session_key
- 根据 openid 查找或创建用户记录
- 签发 JWT（payload 包含 user_id，有效期 7 天）

---

#### GET /api/v1/jobs

**请求参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| keyword | string | 是 | 搜索关键词 |
| city | string | 否 | 城市 |
| salary_min | int | 否 | 最低薪资(K) |
| salary_max | int | 否 | 最高薪资(K) |
| source | string | 否 | 来源平台（boss/lagou） |
| page | int | 否 | 页码，默认 1 |
| page_size | int | 否 | 每页条数，默认 20，最大 50 |

**响应：**
```json
{
  "data": {
    "total": 128,
    "page": 1,
    "page_size": 20,
    "items": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "title": "高级前端工程师",
        "company": "某科技有限公司",
        "salary_min": 25,
        "salary_max": 40,
        "salary_months": 14,
        "location": "北京",
        "experience": "3-5年",
        "education": "本科",
        "source": "boss",
        "published_at": "2026-06-25T10:00:00Z"
      }
    ]
  }
}
```

**实现要点：**

- 搜索走 PostgreSQL 全文索引，keyword 匹配 title + company + description
- city、salary 范围走 BTREE 索引
- 热门搜索词结果缓存到 Redis（TTL 30min）

---

#### GET /api/v1/applications/board

**响应：**
```json
{
  "data": {
    "pending": [
      {
        "id": "uuid",
        "job": {
          "id": "uuid",
          "title": "高级前端工程师",
          "company": "某科技",
          "salary_min": 25,
          "salary_max": 40
        },
        "created_at": "2026-06-25T10:00:00Z",
        "updated_at": "2026-06-25T10:00:00Z",
        "stay_days": 3
      }
    ],
    "applied": [],
    "interview": [],
    "offer": [],
    "ended": []
  }
}
```

**实现要点：**

- 单次查询 `WHERE user_id = ?`，Python 侧按 status 分组
- `stay_days` = `(now - updated_at).days`
- job 为空时（手动添加的记录）返回 null，前端做兜底展示

---

#### PATCH /api/v1/applications/{id}/status

**请求：**
```json
{
  "status": "interview"
}
```

**响应：**
```json
{
  "data": {
    "id": "uuid",
    "status": "interview",
    "updated_at": "2026-06-25T14:00:00Z"
  }
}
```

**实现要点：**

- 状态流转校验（状态机）：

```python
VALID_TRANSITIONS = {
    "pending":  ["applied"],
    "applied":  ["pending", "interview", "ended"],
    "interview": ["applied", "offer", "ended"],
    "offer":    ["ended"],
    "ended":    [],  # 终态，不可回退
}
```

- 校验不通过返回 `409 Conflict` + `INVALID_STATUS_TRANSITION`
- 状态变更后写入 `status_logs` 表
- 如果新状态为 `interview` 且设置了 `interview_time`，创建 Celery 延时任务发送提醒

---

#### POST /api/v1/ai/resume-diagnose

**请求：**
```json
{
  "resume_id": "uuid",
  "target_direction": "前端开发"
}
```

**响应：**
```json
{
  "data": {
    "overall_score": 72,
    "dimensions": {
      "completeness": {
        "score": 80,
        "comment": "缺少项目成果量化描述",
        "suggestions": ["在XX项目中补充具体数据指标"]
      },
      "keywords": {
        "score": 65,
        "comment": "缺少React、TypeScript等高频关键词",
        "suggestions": ["在技能模块补充React、TypeScript"]
      },
      "quantification": {
        "score": 50,
        "comment": "3处描述可量化但未量化",
        "suggestions": ["将'提升了性能'改为'首屏加载时间降低40%'"]
      },
      "format": {
        "score": 85,
        "comment": "结构清晰，排版规范",
        "suggestions": []
      },
      "differentiation": {
        "score": 70,
        "comment": "缺少独特亮点",
        "suggestions": ["补充开源贡献或技术博客"]
      }
    },
    "summary": "简历整体质量中等偏上，主要改进方向：增加量化成果描述、补充目标岗位关键词"
  }
}
```

**实现要点：**

- 从 resumes 表读取 `raw_text`
- 构造 Prompt 发送至 GLM API，要求输出结构化 JSON
- 解析 GLM 返回结果，校验 JSON 格式
- 诊断结果写入 `resumes.last_diagnosis` 字段
- AI 调用超时设为 30s，失败返回 `503 Service Unavailable`

---

## 六、核心模块技术方案

### 6.1 认证模块

**流程：小程序登录**

```
小程序 wx.login() → code → POST /auth/wx-login
→ 后端调用微信 jscode2session → 获取 openid
→ 查找/创建用户 → 签发 JWT → 返回 token
```

**流程：Web 扫码登录**

```
Web 请求 GET /auth/qr-code → 后端生成 scene_id + 小程序码图片
→ 用户扫码 → 小程序携带 scene_id + token 调用确认接口
→ 后端关联 scene_id 与用户 → Web 端轮询确认 → 获取 token
```

**JWT 设计：**

- Payload：`{ "sub": user_id, "exp": timestamp }`
- 有效期：7 天
- 续期策略：剩余有效期 < 1 天时，响应头返回新 token
- 存储位置：小程序 Storage / Web localStorage

### 6.2 爬虫模块

**架构：**

```
Celery Beat (定时调度)
    │
    ├── crawl_boss_jobs (每4小时)
    │     └── BossZhipinCrawler.crawl(keyword, city)
    │
    └── crawl_lagou_jobs (每4小时)
          └── LagouCrawler.crawl(keyword, city)
```

**爬虫基类接口：**

```python
class BaseCrawler(ABC):
    @abstractmethod
    async def crawl(self, keyword: str, city: str = None) -> list[JobRaw]:
        """抓取岗位列表"""
        pass

    @abstractmethod
    async def parse(self, raw_data: dict) -> JobParsed:
        """解析单条岗位数据"""
        pass
```

**反爬策略：**

| 策略 | 实现方式 |
|------|----------|
| 请求频率控制 | 每次请求间隔 3-5 秒（随机） |
| UA 轮换 | 维护 User-Agent 池，随机选取 |
| 代理 IP | 接入代理 IP 池服务，失败自动切换 |
| Cookie 管理 | 模拟登录获取 Cookie，定期刷新 |
| 失败重试 | 最多重试 3 次，指数退避 |

**数据清洗规则：**

- 薪资：统一为 `salary_min`/`salary_max`（K/月）+ `salary_months`
- 城市：统一为标准城市名（如"北京"而非"北京市"）
- 去重：`UPSERT` on `(source, source_id)`

### 6.3 AI 服务模块

**Prompt 设计：**

简历诊断 Prompt 模板：

```
你是一位资深HR和简历顾问。请对以下简历进行专业评估。

目标岗位方向：{target_direction}
简历内容：
{resume_text}

请从以下5个维度评分（0-100分），并给出具体修改建议：
1. completeness（内容完整度）
2. keywords（关键词匹配）
3. quantification（成果量化）
4. format（格式规范）
5. differentiation（差异化）

请以JSON格式输出：
{
  "overall_score": number,
  "dimensions": {
    "completeness": { "score": number, "comment": "string", "suggestions": ["string"] },
    ...
  },
  "summary": "string"
}
```

**调用策略：**

- 模型：GLM-4
- Temperature：0.3（评分类任务需要稳定性）
- Max Tokens：2000
- 超时：30s
- 重试：失败重试 1 次
- 限流：每用户每天最多 5 次诊断（MVP 阶段）

### 6.4 面试提醒模块

**实现方案：**

```
用户设置面试时间
    │
    ├── 写入 applications.interview_time
    ├── 状态自动流转为 interview
    └── 创建 Celery 定时任务
          │
          ├── T-24h: 微信订阅消息推送
          └── T-2h:  微信订阅消息 + Web 浏览器通知
```

**微信订阅消息：**

- 模板ID：需在微信公众平台申请
- 触发方式：用户操作时引导订阅（`wx.requestSubscribeMessage`）
- 推送内容：`{公司名称} {岗位名称} 面试提醒：{面试时间}`

### 6.5 文件处理模块

**简历文件上传流程：**

```
前端上传 PDF/Word
    │
    ├── 直传 MinIO/OSS（预签名 URL 方式）
    │
    后端异步处理
    ├── 下载文件 → 解析文本（pdfminer / python-docx）
    ├── 存入 resumes.raw_text
    └── 更新 resumes.file_url
```

---

## 七、部署方案

### 7.1 Docker Compose 编排

```yaml
services:
  nginx:
    image: nginx:alpine
    ports: ["443:443", "80:80"]
    volumes: [./nginx/conf.d:/etc/nginx/conf.d, ./frontend/dist:/usr/share/nginx/html]

  api:
    build: ./backend
    env_file: .env
    depends_on: [postgres, redis]

  celery-worker:
    build: ./backend
    command: celery -A app.celery worker -l info
    env_file: .env
    depends_on: [postgres, redis]

  celery-beat:
    build: ./backend
    command: celery -A app.celery beat -l info
    env_file: .env
    depends_on: [redis]

  postgres:
    image: postgres:15-alpine
    volumes: [pgdata:/var/lib/postgresql/data]
    environment:
      POSTGRES_DB: jobpilot
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}

  redis:
    image: redis:7-alpine
    volumes: [redisdata:/data]

  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    volumes: [miniodata:/data]

volumes:
  pgdata:
  redisdata:
  miniodata:
```

### 7.2 环境变量

```env
# 应用
APP_ENV=production
APP_SECRET_KEY=xxx

# 数据库
DB_USER=jobpilot
DB_PASSWORD=xxx
DB_HOST=postgres
DB_PORT=5432
DB_NAME=jobpilot

# Redis
REDIS_URL=redis://redis:6379/0

# 微信
WX_APPID=xxx
WX_SECRET=xxx

# GLM API
GLM_API_KEY=xxx
GLM_API_BASE=https://open.bigmodel.cn/api/paas/v4

# MinIO
MINIO_ENDPOINT=minio:9000
MINIO_ACCESS_KEY=xxx
MINIO_SECRET_KEY=xxx
MINIO_BUCKET=resumes
```

---

## 八、监控与运维

### 8.1 日志规范

| 级别 | 使用场景 |
|------|----------|
| ERROR | 接口异常、爬虫失败、AI 调用失败 |
| WARN | 状态流转被拦截、限流触发 |
| INFO | 用户登录、投递记录创建、面试提醒发送 |
| DEBUG | SQL 查询（仅开发环境） |

### 8.2 监控指标

| 指标 | 采集方式 | 告警阈值 |
|------|----------|----------|
| API 响应时间 P99 | Prometheus + FastAPI middleware | > 2s |
| 错误率 | Prometheus | > 1% |
| 爬虫成功率 | 自定义 metrics | < 80% |
| AI 调用成功率 | 自定义 metrics | < 90% |
| 数据库连接池 | SQLAlchemy events | 使用率 > 80% |
| Celery 队列积压 | Flower | > 100 |

---

## 附录

### A. 技术风险与缓解

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| 招聘平台反爬升级 | 数据采集失败 | 降级为手动添加 + 浏览器插件；爬虫模块抽象基类，方便适配新策略 |
| GLM API 限流/不可用 | AI 功能不可用 | 本地缓存常见诊断建议；降级返回通用模板 |
| PostgreSQL 数据量增长 | 查询变慢 | jobs 表按月分区；冷数据归档 |

### B. 技术债务记录

| 项目 | 说明 | 预计处理时间 |
|------|------|-------------|
| 搜索引擎 | MVP 用 PG 全文索引，后续迁移 ES | Phase 2 |
| 消息队列 | 面试提醒用 Celery 延时任务，后续迁移 RabbitMQ 延时队列 | Phase 2 |
| 前端状态管理 | MVP 用 Zustand，后续按需引入 React Query | Phase 2 |

### C. 变更记录

| 日期 | 版本 | 变更内容 | 作者 |
|------|------|----------|------|
| 2026-06-27 | v1.0 | 初始版本 | JobPilot Team |
