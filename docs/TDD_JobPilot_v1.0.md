# JobPilot AI 求职投递效率工具 - 技术设计文档（TDD）

> 版本：v1.1  
> 日期：2026-06-29  
> 状态：草案（定位修正版）  
> 关联文档：[产品需求文档 PRD](./PRD_JobPilot_v1.0.md)

---

## 一、文档概述

### 1.1 文档目的

本文档定义 JobPilot v1.1 的技术实现方案。新版技术设计围绕“AI JD 分类、投递策略生成、平台辅助动作、HR 沟通辅助”展开，不再以简历诊断作为核心架构中心。

### 1.2 设计原则

| 原则 | 说明 |
|------|------|
| 省心自动化 | 用户完成关键设置后，AI 默认自动筛选、打招呼和整理回复 |
| 少量打断 | 仅在验证码/风控、薪资超范围、高风险岗位、不确定回复时进入用户确认 |
| 平台适配隔离 | 招聘平台差异封装在 adapter 层，避免业务逻辑绑定 Boss 单个平台 |
| 安全合规 | 不绕过验证码和风控，不生成虚假经历，敏感信息加密存储 |
| 任务可恢复 | 投递和消息同步都是长任务，必须支持暂停、失败重试和断点续跑 |

---

## 二、技术选型

| 层级 | 技术 | 说明 |
|------|------|------|
| Web 前端 | React + Vite + Ant Design | 辅助查看岗位分析、投递效果和策略复盘 |
| 移动端 | Taro/微信小程序 | 主体验入口，开启托管、查看进展、处理少量异常和 HR 回复 |
| 浏览器插件/自动化助手 | Chrome Extension + Playwright/DOM Adapter | 在招聘平台上下文中读取 JD、辅助打招呼、同步消息 |
| 后端框架 | FastAPI | API 服务、AI 编排、任务管理 |
| 数据库 | PostgreSQL | 结构化业务数据、JSONB 存储 AI 结果 |
| 缓存与队列 | Redis + Celery | 投递任务、消息同步、AI 批处理 |
| AI 服务 | GLM/OpenAI 兼容接口 | JD 分类、岗位摘要、回复建议 |
| 文件/密钥 | OSS/MinIO + KMS/环境密钥 | 简历、附件、平台会话敏感信息 |

---

## 三、系统架构

```text
微信小程序 / Web
       │
       │ HTTPS
       ▼
FastAPI API Gateway
       │
       ├── User/Profile Service
       ├── Job Intelligence Service
       ├── Strategy Service
       ├── Delivery Task Service
       ├── Conversation Service
       └── Platform Adapter Service
       │
       ├── PostgreSQL
       ├── Redis
       └── Celery Worker
              │
              ├── AI Batch Worker
              ├── Delivery Worker
              └── Message Sync Worker
                     │
                     ▼
          Browser Extension / Platform Adapter
                     │
                     ▼
              Boss 直聘等招聘平台
```

### 3.1 模块边界

| 模块 | 职责 |
|------|------|
| User/Profile Service | 用户、求职偏好、个人回答资料、黑名单 |
| Job Intelligence Service | JD 清洗、AI 分类、岗位摘要、匹配度评估 |
| Strategy Service | 关键设置、托管规则、排序规则和避雷条件 |
| Delivery Task Service | 自动执行状态机、频率控制、异常暂停、用户接管、执行日志 |
| Conversation Service | HR 消息聚合、问题识别、话术推荐 |
| Platform Adapter Service | 平台登录状态、页面识别、动作执行、异常上报 |

---

## 四、数据模型

### 4.1 ER 关系

```text
User 1──N UserPreference
User 1──N AnswerProfile
User 1──N Job
Job  1──1 JobAnalysis
User 1──N DeliveryStrategy
DeliveryStrategy 1──N DeliveryTask
DeliveryTask 1──N DeliveryTaskItem
Job 1──N Conversation
Conversation 1──N Message
```

### 4.2 核心表

#### users

| 字段 | 类型 | 说明 |
|------|------|------|
| id | UUID | 主键 |
| openid | VARCHAR(64) | 微信 openid |
| nickname | VARCHAR(64) | 昵称 |
| created_at | TIMESTAMP | 创建时间 |
| updated_at | TIMESTAMP | 更新时间 |

#### user_preferences

| 字段 | 类型 | 说明 |
|------|------|------|
| id | UUID | 主键 |
| user_id | UUID | 用户 ID |
| target_roles | JSONB | 目标方向，如 AI 产品、B 端产品 |
| target_cities | JSONB | 目标城市 |
| acceptable_cities | JSONB | 可接受城市 |
| blocked_cities | JSONB | 不考虑城市 |
| salary_expectation | JSONB | 期望薪资和最低薪资 |
| company_preferences | JSONB | 公司规模、行业、融资阶段偏好 |
| blacklist_keywords | JSONB | 黑名单关键词 |
| delivery_mode | VARCHAR(32) | 练手优先、均衡投递、目标冲刺 |

#### answer_profiles

| 字段 | 类型 | 说明 |
|------|------|------|
| id | UUID | 主键 |
| user_id | UUID | 用户 ID |
| expected_salary | VARCHAR(64) | 期望薪资 |
| availability | VARCHAR(128) | 到岗时间 |
| interview_time_slots | JSONB | 可面试时间段 |
| core_experience | JSONB | 核心项目经历 |
| custom_answers | JSONB | 高频问题自定义答案 |

#### jobs

| 字段 | 类型 | 说明 |
|------|------|------|
| id | UUID | 主键 |
| user_id | UUID | 所属用户 |
| source | VARCHAR(32) | 平台，如 boss |
| source_job_id | VARCHAR(128) | 平台岗位 ID |
| title | VARCHAR(128) | 岗位标题 |
| company | VARCHAR(128) | 公司名称 |
| location | VARCHAR(64) | 城市 |
| salary_text | VARCHAR(64) | 原始薪资文本 |
| jd_text | TEXT | JD 原文 |
| url | VARCHAR(512) | 原始链接 |
| status | VARCHAR(32) | 待筛选、待投递、已打招呼、已回复等 |
| created_at | TIMESTAMP | 创建时间 |
| updated_at | TIMESTAMP | 更新时间 |

#### job_analyses

| 字段 | 类型 | 说明 |
|------|------|------|
| id | UUID | 主键 |
| job_id | UUID | 岗位 ID |
| direction | VARCHAR(64) | 岗位方向分类 |
| match_level | VARCHAR(16) | high/medium/low |
| priority_score | INTEGER | 策略排序分 |
| requirement_summary | JSONB | 核心要求摘要 |
| keywords | JSONB | 能力关键词 |
| risk_tags | JSONB | 风险标签 |
| reasoning | TEXT | AI 判断依据 |
| manually_corrected | BOOLEAN | 是否被人工修正 |

#### delivery_strategies

| 字段 | 类型 | 说明 |
|------|------|------|
| id | UUID | 主键 |
| user_id | UUID | 用户 ID |
| name | VARCHAR(64) | 策略名称 |
| mode | VARCHAR(32) | 策略模板 |
| rules | JSONB | 排序、筛选、上限、间隔规则 |
| is_active | BOOLEAN | 是否启用 |

#### delivery_tasks

| 字段 | 类型 | 说明 |
|------|------|------|
| id | UUID | 主键 |
| user_id | UUID | 用户 ID |
| strategy_id | UUID | 策略 ID |
| status | VARCHAR(32) | idle/running/paused/manual_required/completed/failed/canceled |
| daily_limit | INTEGER | 每日上限 |
| hourly_limit | INTEGER | 每小时上限 |
| random_delay_range | JSONB | 随机间隔 |
| started_at | TIMESTAMP | 开始时间 |
| finished_at | TIMESTAMP | 结束时间 |

#### delivery_task_items

| 字段 | 类型 | 说明 |
|------|------|------|
| id | UUID | 主键 |
| task_id | UUID | 任务 ID |
| job_id | UUID | 岗位 ID |
| status | VARCHAR(32) | pending/running/succeeded/skipped/failed/manual_required |
| greeting_text | TEXT | 打招呼文案 |
| sort_reason | TEXT | 进入队列和排序原因 |
| execution_log | JSONB | 执行记录 |
| last_error | TEXT | 最近错误 |

#### conversations

| 字段 | 类型 | 说明 |
|------|------|------|
| id | UUID | 主键 |
| user_id | UUID | 用户 ID |
| job_id | UUID | 岗位 ID |
| platform_conversation_id | VARCHAR(128) | 平台会话 ID |
| hr_name | VARCHAR(64) | HR 名称 |
| status | VARCHAR(32) | unread/need_reply/replied/interviewing/closed |
| last_message_at | TIMESTAMP | 最近消息时间 |

#### messages

| 字段 | 类型 | 说明 |
|------|------|------|
| id | UUID | 主键 |
| conversation_id | UUID | 会话 ID |
| sender_type | VARCHAR(16) | user/hr/system |
| content | TEXT | 消息内容 |
| question_type | VARCHAR(64) | 薪资、面试时间、到岗时间等 |
| reply_suggestion | TEXT | AI 回复建议 |
| needs_manual_confirm | BOOLEAN | 是否需要人工确认 |
| sent_at | TIMESTAMP | 发送时间 |

---

## 五、接口设计

### 5.1 接口清单

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/v1/auth/wx-login` | 微信登录 |
| GET | `/api/v1/preferences` | 获取求职偏好 |
| PUT | `/api/v1/preferences` | 更新求职偏好 |
| GET | `/api/v1/answer-profile` | 获取个人回答资料 |
| PUT | `/api/v1/answer-profile` | 更新个人回答资料 |
| POST | `/api/v1/jobs/import` | 导入岗位/JD |
| GET | `/api/v1/jobs` | 岗位列表 |
| POST | `/api/v1/jobs/analyze` | 批量触发 AI 分类 |
| PATCH | `/api/v1/jobs/{id}/analysis` | 人工修正岗位分类 |
| POST | `/api/v1/strategies` | 创建投递策略 |
| POST | `/api/v1/hosting/start` | 开启 AI 求职托管 |
| POST | `/api/v1/hosting/pause` | 暂停 AI 求职托管 |
| POST | `/api/v1/hosting/resume` | 继续 AI 求职托管 |
| GET | `/api/v1/hosting/summary` | 获取今日托管进展和需要用户处理的事项 |
| POST | `/api/v1/delivery-tasks` | 创建自动执行批次 |
| POST | `/api/v1/delivery-tasks/{id}/cancel` | 取消任务 |
| GET | `/api/v1/conversations` | 会话列表 |
| POST | `/api/v1/conversations/{id}/suggest-reply` | 生成回复建议 |

### 5.2 POST /api/v1/jobs/analyze

**请求**：

```json
{
  "job_ids": ["uuid-1", "uuid-2"],
  "preference_id": "uuid"
}
```

**响应**：

```json
{
  "data": {
    "task_id": "uuid",
    "status": "queued"
  }
}
```

**实现要点**：

- 后端将分析任务写入 Celery 队列。
- Worker 批量读取 JD 和用户偏好。
- AI 返回结构化 JSON，写入 `job_analyses`。
- 失败任务记录错误并允许重试。

### 5.3 POST /api/v1/hosting/start

**请求**：

```json
{
  "preference_id": "uuid",
  "mode": "practice_first"
}
```

**响应**：

```json
{
  "data": {
    "hosting_status": "running",
    "today_limit": 12,
    "message": "AI 托管已开启，将按规则自动筛选和打招呼"
  }
}
```

**实现要点**：

- 读取用户偏好、黑名单、薪资下限和投递节奏。
- 创建或恢复当天自动执行批次。
- 后台 Worker 自动生成执行顺序，无需用户逐条确认。
- 命中异常规则的岗位进入 `manual_required`，其余岗位继续自动执行。

### 5.4 POST /api/v1/conversations/{id}/suggest-reply

**请求**：

```json
{
  "message_id": "uuid"
}
```

**响应**：

```json
{
  "data": {
    "question_type": "expected_salary",
    "suggestion": "我的期望薪资是 25-30K，具体也会结合岗位职责、团队情况和整体薪酬结构沟通。",
    "needs_manual_confirm": false
  }
}
```

---

## 六、AI 服务设计

### 6.1 JD 分类 Prompt 输出 Schema

```json
{
  "direction": "AI产品经理",
  "match_level": "high",
  "priority_score": 86,
  "requirement_summary": [
    "负责 AI 应用产品从需求到上线的完整流程",
    "要求有大模型或智能客服相关经验",
    "重视数据分析和跨团队协作"
  ],
  "keywords": ["AI应用", "大模型", "需求分析", "数据分析"],
  "risk_tags": ["职责边界偏宽"],
  "reasoning": "岗位方向与用户目标一致，薪资和城市符合偏好，但 JD 中存在部分运营支持职责。"
}
```

### 6.2 回复建议策略

- 从 `answer_profiles` 获取确定性资料。
- 判断 HR 消息是否属于高频问题。
- 对薪资、到岗时间、面试时间等确定性问题生成短回复；用户授权后可自动发送。
- 对经历真实性、技术细节、意愿不明确的问题标记 `needs_manual_confirm=true`，进入“需要我看”。
- 回复内容不得捏造用户经历。

---

## 七、投递任务状态机

```text
idle -> running -> completed
        │
        ├── paused -> running
        ├── manual_required -> running
        ├── failed
        └── canceled
```

### 7.1 自动暂停条件

- 平台登录失效。
- 出现验证码或风控提示。
- 连续失败次数达到阈值。
- 当前小时或当天执行量达到上限。
- 检测到岗位命中黑名单。
- 薪资、城市、公司类型或岗位风险超出用户授权范围。
- HR 问题涉及不确定承诺或用户未提供资料。

### 7.2 频率控制

- 每日上限和每小时上限由策略配置决定。
- 单次动作之间加入随机间隔。
- 同公司、同 HR、同岗位去重。
- 重复模板文案比例过高时降低执行频率。

---

## 八、平台适配设计

### 8.1 Adapter 接口

```python
class RecruitingPlatformAdapter(Protocol):
    async def detect_login_status(self) -> LoginStatus:
        ...

    async def collect_jobs(self, query: JobQuery) -> list[RawJob]:
        ...

    async def open_job(self, source_job_id: str) -> RawJobDetail:
        ...

    async def send_greeting(self, source_job_id: str, text: str) -> ActionResult:
        ...

    async def sync_messages(self) -> list[RawMessage]:
        ...
```

### 8.2 合规处理

- Adapter 不处理绕过验证码、破解接口、规避风控等逻辑。
- 一旦遇到平台安全提示，返回 `manual_required`。
- 所有平台动作写入执行日志，方便用户审计。

---

## 九、安全与隐私

| 数据 | 处理方式 |
|------|----------|
| 简历和个人资料 | 加密存储，访问鉴权 |
| 平台会话信息 | 尽量保存在本地插件侧，服务端只保存必要状态 |
| 聊天记录 | 用户授权后同步，支持删除 |
| AI 输入 | 最小化传输，仅传任务所需字段 |
| 执行日志 | 保留用于审计，支持用户清理 |

---

## 十、监控指标

| 指标 | 说明 |
|------|------|
| JD 分类成功率 | AI 分类任务成功比例 |
| 需要用户处理率 | 衡量 AI 托管是否足够省心，越低越好 |
| 投递任务成功率 | 打招呼成功数 / 执行数 |
| 自动暂停次数 | 反映平台异常和风控情况 |
| HR 回复率 | 已回复岗位 / 已打招呼岗位 |
| 约面率 | 约面岗位 / 已回复岗位 |
| 回复建议采纳率 | 用户采用 AI 建议的比例 |

---

## 十一、风险与技术债

| 风险 | 应对 |
|------|------|
| Boss 页面结构变化 | Adapter 层隔离，增加页面选择器健康检查 |
| 平台限制自动化 | 降低频率、异常暂停、插件辅助，不绕过风控 |
| AI 结果不稳定 | Schema 校验、重试、人工修正反馈 |
| 长任务中断 | Celery 任务幂等设计，item 级状态恢复 |
| 数据过度采集 | 最小化存储，用户可删除聊天和岗位数据 |

---

## 附录

### A. 变更记录

| 日期 | 版本 | 变更内容 | 作者 |
|------|------|----------|------|
| 2026-06-27 | v1.0 | 初始版本 | JobPilot Team |
| 2026-06-29 | v1.1 | 技术架构改为围绕 JD 分类、投递策略、平台动作和沟通辅助 | Codex |
