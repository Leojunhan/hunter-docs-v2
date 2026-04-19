# Hunter 系统 Agent 管理模块 · 开发启动

---

## 一、可用 Agent 支持

### 1. CoPaw 内置 Skills（11 个）

| Skill | 状态 | 用途 |
|-------|------|------|
| `browser_visible` | 可启用 | 可见浏览器自动化 |
| `cron` | 可启用 | 定时任务管理 |
| `dingtalk_channel` | 可启用 | 钉钉频道接入 |
| `docx` | 可启用 | Word 文档处理 |
| `file_reader` | 可启用 | 文件读取 |
| `guidance` | 可启用 | CoPaw 配置帮助 |
| `himalaya` | 可启用 | 邮件管理（IMAP/SMTP） |
| `news` | 可启用 | 新闻获取 |
| `pdf` | 可启用 | PDF 处理 |
| `pptx` | 可启用 | PPT 处理 |
| `xlsx` | 可启用 | Excel 处理 |

**启用方式：** `copaw skills enable <skill_name>`

---

### 2. 自定义 Skills（7 个）

| Skill | 用途 |
|-------|------|
| `business-architecture` | 业务架构文档生成 |
| `doc_validator` | 文档结构验证 |
| `ljh-diagram` | 图表绘制（Mermaid/PlantUML） |
| `ljh-prd` | PRD 文档生成 |
| `market-research` | 市场调研分析 |
| `mrd-brd` | MRD+BRD 战略文档 |
| `trend-tracking` | 行业趋势追踪 |

---

### 3. 外部 Agent 工具（工作区已配置）

| Agent | 类型 | 用途 |
|-------|------|------|
| **Claude Desktop** | AI 助手 | 代码生成、对话 |
| **Continue** | VSCode 插件 | IDE 内代码辅助 |
| **OpenHands** | 代码 Agent | 自主编程 |
| **Goose** | 代码 Agent | 自主编程 |
| **Windsurf/Cursor** | AI IDE | 代码生成（需单独安装） |

---

### 4. 元智能体能力（已开发完成）

| 模块 | 能力 | 状态 |
|------|------|------|
| **A2A Client** | A2A 协议 Skill 发现 | ✅ 完成 |
| **MCP Client** | MCP 协议 Skill 发现 | ✅ 完成 |
| **Skill Registry** | 统一 Skill 管理 | ✅ 完成 |
| **Workflow Engine** | 多 Agent 编排 | ✅ 完成 |

**可复用代码：** `/app/working/workspaces/RAKkDm/meta_agent/`

---

## 二、推荐方案

### 方案对比

| 方案 | 优势 | 劣势 | 推荐度 |
|------|------|------|--------|
| **小宁（当前 Agent）** | 上下文完整、产品思维一致 | 代码能力通用级 | ⭐⭐⭐ |
| **OpenHands/Goose** | 专精代码生成 | 无业务上下文、需重新配置 | ⭐⭐ |
| **Cursor/Windsurf** | IDE 集成好、代码质量高 | 需单独安装、无业务上下文 | ⭐⭐ |
| **混合模式** | 小宁设计 + 专精 Agent 编码 | 协调成本高 | ⭐⭐⭐ |

---

### 我的推荐：**混合模式**

```
小宁（产品 + 架构） + OpenHands/Goose（代码实现） + Diwei（评审决策）
```

**分工：**
| 角色 | 职责 | 工具 |
|------|------|------|
| **小宁** | 产品设计、架构设计、PRD 输出 | 当前 Agent |
| **OpenHands** | 代码实现、单元测试 | 代码专精 Agent |
| **Diwei** | 方向把控、关键决策 | 人工评审 |

---

## 三、开场输入描述（复制给开发 Agent）

```markdown
# Hunter 系统 Agent 管理模块 · 开发任务

## 项目背景

Hunter 系统是一个 OFW 信贷获客系统，需要开发 Agent 管理模块来管理多个 AI Agent 的协同工作。

## 技术栈

- 后端：Python FastAPI
- 数据库：PostgreSQL / SQLite
- Agent 协议：A2A + MCP（已有实现）
- 工作流引擎：YAML 配置（已有实现）

## 已有资产

以下代码可直接复用（位于 `/app/working/workspaces/RAKkDm/meta_agent/`）：

1. **Skill Discovery**
   - `skill_discovery/a2a_client.py` - A2A 协议客户端
   - `skill_discovery/mcp_client.py` - MCP 协议客户端
   - `skill_discovery/skill_wrapper.py` - A2A Skill 封装
   - `skill_discovery/mcp_wrapper.py` - MCP Skill 封装
   - `skill_discovery/registry.py` - Skill 注册中心

2. **Workflow Engine**
   - `orchestration/workflow.py` - Workflow 引擎
   - 支持线性编排、变量传递、条件分支、重试机制

## 开发任务

### Phase 1: Agent 注册与发现（优先级：⭐⭐⭐）

**目标：实现 Agent 的注册、发现、状态管理**

**功能需求：**
1. Agent 注册 API
   - POST /agents/register
   - 参数：agent_id, name, type (a2a/mcp/local), config, metadata
   - 返回：agent_id, status

2. Agent 发现 API
   - GET /agents/list
   - GET /agents/{agent_id}
   - 支持按类型、状态筛选

3. Agent 状态管理
   - 心跳检测（每 30 秒）
   - 状态更新（online/offline/busy）
   - 自动下线（超时 90 秒）

4. 数据库设计
   - agents 表：id, name, type, config, status, last_heartbeat, created_at

**代码结构：**
```
hunter_agent/
├── api/
│   ├── agents.py        # Agent 管理 API
│   └── schemas.py       # Pydantic 模型
├── core/
│   ├── registry.py      # Agent 注册中心（复用 meta_agent）
│   └── heartbeat.py     # 心跳检测
├── models/
│   └── agent.py         # SQLAlchemy 模型
├── db/
│   └── database.py      # 数据库连接
└── main.py              # FastAPI 入口
```

**验收标准：**
- [ ] 可以注册 Agent
- [ ] 可以查询 Agent 列表
- [ ] 心跳检测正常工作
- [ ] 单元测试覆盖率 >80%

---

### Phase 2: Agent 编排与执行（优先级：⭐⭐⭐）

**目标：实现多 Agent 协同工作流**

**功能需求：**
1. Workflow 定义 API
   - POST /workflows
   - GET /workflows/{workflow_id}
   - YAML 配置存储

2. Workflow 执行 API
   - POST /workflows/{workflow_id}/execute
   - 异步执行，返回 execution_id
   - GET /executions/{execution_id}/status 查询状态

3. 执行日志
   - 记录每个步骤的执行结果
   - 支持日志查询

**验收标准：**
- [ ] 可以创建 Workflow
- [ ] 可以执行 Workflow
- [ ] 可以查询执行状态
- [ ] 单元测试覆盖率 >80%

---

### Phase 3: Hunter 业务集成（优先级：⭐⭐）

**目标：与 Hunter 现有业务集成**

**功能需求：**
1. 获客场景 Workflow
   - FB 社群监控 → 意图识别 → 申请引导
   - KOL 合作管理 → 效果追踪 → 佣金结算

2. 风控场景 Workflow
   - 申请采集 → 风控预审 → 人工复核 → 放款

3. 贷后场景 Workflow
   - 还款提醒 → 逾期预警 → 复借引导

**验收标准：**
- [ ] 3 个核心业务流程跑通
- [ ] 与 Hunter 现有系统对接
- [ ] 端到端测试通过

---

## 开发规范

### 代码风格
- Python 3.10+
- 使用 type hints
- 遵循 PEP 8
- 异步优先（async/await）

### 测试要求
- 单元测试：pytest
- 覆盖率：>80%
- CI/CD：GitHub Actions

### 文档要求
- API 文档：OpenAPI/Swagger
- 代码注释：Google Style
- README：包含安装、使用示例

---

## 交付物

1. **代码仓库**
   - 完整源代码
   - 单元测试
   - 集成测试

2. **文档**
   - API 文档
   - 部署文档
   - 使用示例

3. **演示**
   - 可运行的 Demo
   - 核心功能演示视频

---

## 时间估算

| 阶段 | 预计时间 | 优先级 |
|------|---------|--------|
| Phase 1 | 3-5 天 | ⭐⭐⭐ |
| Phase 2 | 3-5 天 | ⭐⭐⭐ |
| Phase 3 | 5-7 天 | ⭐⭐ |
| **总计** | **11-17 天** | - |

---

## 联系方式

- 产品经理：Diwei
- 技术顾问：小宁（RAKkDm）
- 问题反馈：直接在工作区提问

---

**开始开发前，请先确认：**
1. 理解需求了吗？
2. 技术栈熟悉吗？
3. 需要更多上下文吗？

**确认后开始 Phase 1 开发。**
```

---

## 四、下一步行动

### 如果你选择我继续开发

**我会：**
1. 阅读 Hunter 系统现有代码（如果有）
2. 设计详细架构
3. 输出 PRD 文档
4. 开始 Phase 1 开发

**你需要：**
- 提供 Hunter 系统代码库访问
- 确认技术栈选择
- 评审我的设计文档

---

### 如果你选择其他 Agent

**我会帮你：**
1. 整理完整上下文文档
2. 准备交接材料
3. 参与设计评审
4. 保持产品思维一致性

**你需要：**
- 选择目标 Agent（OpenHands/Goose/Cursor 等）
- 配置开发环境
- 评审代码质量

---

## 五、我的建议

**建议采用混合模式：**

```
Phase 1（架构设计） → 小宁
    ↓
Phase 2（代码实现） → OpenHands/Goose
    ↓
Phase 3（集成测试） → 小宁 + Diwei 评审
```

**理由：**
- 我擅长架构设计，保证方向正确
- 代码 Agent 擅长实现，保证代码质量
- 你把握关键决策，保证业务对齐

---

**文档版本：v1.0**  
**创建时间：2026-04-16**  
**创建者：小宁（RAKkDm）**
