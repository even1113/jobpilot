# JobPilot

JobPilot 是一个面向求职投递场景的 AI 效率工具项目，目标是帮助求职者降低 JD 阅读、岗位筛选、投递排序、HR 沟通和状态跟进的重复成本。

当前仓库处于产品定义和原型阶段，包含产品需求文档、技术设计文档和一个可直接打开的静态交互原型。

## 项目定位

JobPilot 的核心不是简历美化或简历评分，而是围绕真实投递流程提供辅助能力：

- 批量理解招聘 JD，提取岗位方向、关键要求、风险点和匹配度。
- 根据用户偏好、城市、薪资、公司类型和岗位优先级生成投递策略。
- 支持先练手、再投目标公司的分阶段投递节奏。
- 聚合 HR 回复，减少重复沟通和手动状态维护。
- 在用户授权和规则约束下辅助完成招聘平台上的重复操作。

## 仓库结构

```text
.
|-- docs/
|   |-- PRD_JobPilot_v1.0.md    # 产品需求文档
|   `-- TDD_JobPilot_v1.0.md    # 技术设计文档
|-- prototype/
|   `-- index.html              # C 端自动化体验静态原型
|-- Records.md                  # 项目过程记录
`-- README.md
```

## 快速预览原型

当前原型是纯静态 HTML 文件，不需要安装依赖。

直接用浏览器打开：

```text
prototype/index.html
```

或在项目根目录启动一个简单静态服务：

```bash
python -m http.server 8000
```

然后访问：

```text
http://localhost:8000/prototype/
```

## 文档

- 产品需求文档：[docs/PRD_JobPilot_v1.0.md](docs/PRD_JobPilot_v1.0.md)
- 技术设计文档：[docs/TDD_JobPilot_v1.0.md](docs/TDD_JobPilot_v1.0.md)
- 项目过程记录：[Records.md](Records.md)

## 规划中的技术方向

根据当前 TDD，后续实现可能包含：

- Web 前端：React + Vite + Ant Design
- 移动端：Taro / 微信小程序
- 浏览器插件：Chrome Extension + DOM Adapter / Playwright
- 后端服务：FastAPI
- 数据库：PostgreSQL
- 队列与缓存：Redis + Celery
- AI 服务：OpenAI / GLM 兼容接口

具体实现以技术设计文档和后续迭代为准。

## 当前状态

- 已完成：产品定位、PRD、TDD、C 端体验原型。
- 进行中：基于原型和文档收敛 MVP 范围。
- 待完成：工程化前后端实现、浏览器插件能力、任务队列、AI 编排和数据持久化。

## 注意事项

本项目涉及招聘平台辅助操作，后续实现应遵守平台规则和安全合规原则：

- 不绕过验证码、风控或平台访问限制。
- 不生成虚假经历或误导性求职信息。
- 不鼓励无差别骚扰式投递。
- 敏感信息需要加密存储，并遵循最小权限原则。
