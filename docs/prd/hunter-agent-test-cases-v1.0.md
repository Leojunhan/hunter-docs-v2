# Hunter 系统 Agent 管理模块 · 测试用例

> **版本：** v1.0  
> **作者：** 小宁（RAKkDm）  
> **日期：** 2026-04-16  
> **状态：** 待执行  
> **测试类型：** 单元测试 + 集成测试 + 性能测试

---

## 一、测试总览

### 1.1 测试范围

| 模块 | 测试类型 | 用例数 | 优先级 |
|:-----|:---------|:------:|:------:|
| **Agent 新建与管理** | 单元测试 + 集成测试 | 25 | P0 |
| **心跳检测** | 单元测试 + 集成测试 | 10 | P0 |
| **工作流编排** | 单元测试 + 集成测试 | 20 | P0 |
| **工作流执行** | 单元测试 + 集成测试 | 25 | P0 |
| **管理后台** | 集成测试 | 15 | P1 |
| **数据看板** | 集成测试 | 8 | P2 |
| **性能测试** | 压力测试 | 6 | P1 |
| **合计** | - | **109** | - |

### 1.2 测试环境

| 环境 | URL | 用途 | 负责人 |
|:-----|:----|:-----|:-------|
| 开发环境 | http://localhost:8000 | 开发自测 | 开发 |
| 测试环境 | http://test-hunter.internal | 集成测试 | 测试 |
| 预发布环境 | http://staging-hunter.internal | 验收测试 | 产品 + 测试 |
| 生产环境 | http://hunter.ofw.internal | 线上验证 | 运维 |

### 1.3 测试工具

| 工具 | 用途 | 版本 |
|:-----|:-----|:-----|
| pytest | 单元测试框架 | 7.0+ |
| pytest-asyncio | 异步测试支持 | 0.21+ |
| httpx | API 测试客户端 | 0.24+ |
| locust | 性能压测 | 2.0+ |
| pytest-cov | 覆盖率统计 | 4.0+ |

---

## 二、测试用例详情

### 2.1 Agent 新建与管理（25 个用例）

#### 2.1.1 新建 Agent

| 用例 ID | 用例名称 | 前置条件 | 测试步骤 | 预期结果 | 优先级 |
|:--------|:---------|:---------|:---------|:---------|:------:|
| **TC-AG-001** | 新建 A2A 类型 Agent | 无 | 1. 调用 POST /agents<br>2. 传入 type="a2a"<br>3. 传入有效 config | 返回 201，agent_id 不为空 | P0 |
| **TC-AG-002** | 新建 MCP 类型 Agent | 无 | 1. 调用 POST /agents<br>2. 传入 type="mcp"<br>3. 传入有效 config | 返回 201，agent_id 不为空 | P0 |
| **TC-AG-003** | 新建 Local 类型 Agent | 无 | 1. 调用 POST /agents<br>2. 传入 type="local"<br>3. 传入有效 config | 返回 201，agent_id 不为空 | P0 |
| **TC-AG-004** | 新建 Agent - 名称重复 | 已存在同名 Agent | 1. 调用 POST /agents<br>2. 传入已存在的 name | 返回 409，错误信息提示名称重复 | P0 |
| **TC-AG-005** | 新建 Agent - 名称为空 | 无 | 1. 调用 POST /agents<br>2. name 字段为空 | 返回 400，验证错误 | P0 |
| **TC-AG-006** | 新建 Agent - 类型无效 | 无 | 1. 调用 POST /agents<br>2. 传入 type="invalid" | 返回 400，验证错误 | P0 |
| **TC-AG-007** | 新建 Agent - config 缺失 | 无 | 1. 调用 POST /agents<br>2. config 字段缺失 | 返回 400，验证错误 | P0 |
| **TC-AG-008** | 新建 Agent - config 格式错误 | 无 | 1. 调用 POST /agents<br>2. config 不是合法 JSON | 返回 400，验证错误 | P0 |
| **TC-AG-009** | 新建 Agent - metadata 可选 | 无 | 1. 调用 POST /agents<br>2. 不传 metadata 字段 | 返回 201，metadata 为 null | P1 |
| **TC-AG-010** | 新建 Agent - 名称超长 | 无 | 1. 调用 POST /agents<br>2. name 长度>50 字符 | 返回 400，验证错误 | P1 |

**测试代码示例：**
```python
# tests/test_agents.py
@pytest.mark.asyncio
async def test_create_a2a_agent(client):
    """TC-AG-001: 新建 A2A 类型 Agent"""
    payload = {
        "name": "weather-agent",
        "type": "a2a",
        "config": {"url": "http://agent:8000"},
        "metadata": {"description": "天气查询"}
    }
    response = await client.post("/agents", json=payload)
    assert response.status_code == 201
    data = response.json()
    assert "agent_id" in data
    assert data["status"] == "offline"

@pytest.mark.asyncio
async def test_create_duplicate_agent(client):
    """TC-AG-004: 新建 Agent - 名称重复"""
    # 先创建一个
    payload1 = {"name": "test-agent", "type": "a2a", "config": {}}
    await client.post("/agents", json=payload1)
    
    # 再创建同名
    payload2 = {"name": "test-agent", "type": "a2a", "config": {}}
    response = await client.post("/agents", json=payload2)
    assert response.status_code == 409
    assert "名称重复" in response.json()["detail"]
```

---

#### 2.1.2 查询 Agent

| 用例 ID | 用例名称 | 前置条件 | 测试步骤 | 预期结果 | 优先级 |
|:--------|:---------|:---------|:---------|:---------|:------:|
| **TC-AG-011** | 查询 Agent 列表 - 空数据 | 无 Agent | 1. 调用 GET /agents | 返回 200，total=0，agents=[] | P0 |
| **TC-AG-012** | 查询 Agent 列表 - 有数据 | 存在 3 个 Agent | 1. 调用 GET /agents | 返回 200，total=3 | P0 |
| **TC-AG-013** | 查询 Agent 列表 - 分页 | 存在 20 个 Agent | 1. 调用 GET /agents?page=1&page_size=10 | 返回 200，返回 10 条 | P0 |
| **TC-AG-014** | 查询 Agent 列表 - 按类型筛选 | 存在 A2A/MCP 类型 | 1. 调用 GET /agents?type=a2a | 返回 200，只包含 a2a 类型 | P0 |
| **TC-AG-015** | 查询 Agent 列表 - 按状态筛选 | 存在 online/offline | 1. 调用 GET /agents?status=online | 返回 200，只包含 online | P0 |
| **TC-AG-016** | 查询单个 Agent - 存在 | 存在 agent_001 | 1. 调用 GET /agents/agent_001 | 返回 200，返回完整信息 | P0 |
| **TC-AG-017** | 查询单个 Agent - 不存在 | 无 | 1. 调用 GET /agents/not_exist | 返回 404，错误信息 | P0 |
| **TC-AG-018** | 查询 Agent - 组合筛选 | 存在多种类型和状态 | 1. 调用 GET /agents?type=a2a&status=online | 返回 200，符合筛选条件 | P1 |

---

#### 2.1.3 更新/删除 Agent

| 用例 ID | 用例名称 | 前置条件 | 测试步骤 | 预期结果 | 优先级 |
|:--------|:---------|:---------|:---------|:---------|:------:|
| **TC-AG-019** | 更新 Agent 配置 | 存在 agent_001 | 1. 调用 PUT /agents/agent_001/config<br>2. 传入新 config | 返回 200，配置已更新 | P0 |
| **TC-AG-020** | 更新 Agent - 不存在 | 无 | 1. 调用 PUT /agents/not_exist/config | 返回 404 | P0 |
| **TC-AG-021** | 启用 Agent | 存在 disabled 的 Agent | 1. 调用 PATCH /agents/agent_001/enable | 返回 200，status=enabled | P1 |
| **TC-AG-022** | 禁用 Agent | 存在 enabled 的 Agent | 1. 调用 PATCH /agents/agent_001/disable | 返回 200，status=disabled | P1 |
| **TC-AG-023** | 删除 Agent - 成功 | 存在 agent_001 | 1. 调用 DELETE /agents/agent_001 | 返回 200，success=true | P0 |
| **TC-AG-024** | 删除 Agent - 不存在 | 无 | 1. 调用 DELETE /agents/not_exist | 返回 404 | P0 |
| **TC-AG-025** | 删除 Agent - 有关联工作流 | 存在关联的 workflow | 1. 调用 DELETE /agents/agent_001 | 返回 400，提示有关联 | P1 |

---

### 2.2 心跳检测（10 个用例）

#### 2.2.1 心跳上报

| 用例 ID | 用例名称 | 前置条件 | 测试步骤 | 预期结果 | 优先级 |
|:--------|:---------|:---------|:---------|:---------|:------:|
| **TC-HB-001** | 心跳上报 - 成功 | 存在 agent_001 | 1. 调用 POST /agents/agent_001/heartbeat | 返回 200，last_heartbeat 更新 | P0 |
| **TC-HB-002** | 心跳上报 - Agent 不存在 | 无 | 1. 调用 POST /agents/not_exist/heartbeat | 返回 404 | P0 |
| **TC-HB-003** | 心跳上报 - 状态更新 | Agent 为 offline | 1. 调用 POST /agents/agent_001/heartbeat | 返回 200，status=online | P0 |
| **TC-HB-004** | 心跳上报 - 并发处理 | 存在 agent_001 | 1. 并发发送 10 个心跳请求 | 返回 200，无数据竞争 | P1 |

---

#### 2.2.2 心跳检测

| 用例 ID | 用例名称 | 前置条件 | 测试步骤 | 预期结果 | 优先级 |
|:--------|:---------|:---------|:---------|:---------|:------:|
| **TC-HB-005** | 心跳超时 - 标记离线 | Agent 心跳正常 | 1. 等待 90 秒无心跳<br>2. 查询 Agent 状态 | status=offline | P0 |
| **TC-HB-006** | 心跳超时 - 触发告警 | Agent 心跳正常 | 1. 等待 90 秒无心跳<br>2. 检查告警通知 | 收到告警通知 | P1 |
| **TC-HB-007** | 心跳恢复 - 状态更新 | Agent 为 offline | 1. 发送心跳<br>2. 查询状态 | status=online | P0 |
| **TC-HB-008** | 心跳间隔配置 | 无 | 1. 修改 heartbeat_interval=10<br>2. 观察心跳频率 | 每 10 秒一次心跳 | P2 |
| **TC-HB-009** | 心跳超时配置 | 无 | 1. 修改 heartbeat_timeout=60<br>2. 等待 60 秒 | 60 秒后标记离线 | P2 |
| **TC-HB-010** | 心跳检测 - 大批量 | 存在 1000 个 Agent | 1. 启动心跳检测<br>2. 观察性能 | CPU<70%，无阻塞 | P1 |

**测试代码示例：**
```python
# tests/test_heartbeat.py
@pytest.mark.asyncio
async def test_heartbeat_updates_status(client):
    """TC-HB-003: 心跳上报 - 状态更新"""
    # 创建 Agent（初始 offline）
    agent = await create_agent(client)
    assert agent["status"] == "offline"
    
    # 发送心跳
    response = await client.post(f"/agents/{agent['agent_id']}/heartbeat")
    assert response.status_code == 200
    
    # 查询状态
    response = await client.get(f"/agents/{agent['agent_id']}")
    assert response.json()["status"] == "online"

@pytest.mark.asyncio
async def test_heartbeat_timeout(client):
    """TC-HB-005: 心跳超时 - 标记离线"""
    agent = await create_agent(client)
    
    # 发送心跳
    await client.post(f"/agents/{agent['agent_id']}/heartbeat")
    
    # 等待超时
    await asyncio.sleep(95)  # 90 秒超时 + 5 秒缓冲
    
    # 查询状态
    response = await client.get(f"/agents/{agent['agent_id']}")
    assert response.json()["status"] == "offline"
```

---

### 2.3 工作流编排（20 个用例）

#### 2.3.1 创建工作流

| 用例 ID | 用例名称 | 前置条件 | 测试步骤 | 预期结果 | 优先级 |
|:--------|:---------|:---------|:---------|:---------|:------:|
| **TC-WF-001** | 创建工作流 - 成功 | 无 | 1. 调用 POST /workflows<br>2. 传入有效 YAML | 返回 201，workflow_id 不为空 | P0 |
| **TC-WF-002** | 创建工作流 - YAML 格式错误 | 无 | 1. 调用 POST /workflows<br>2. 传入无效 YAML | 返回 400，验证错误 | P0 |
| **TC-WF-003** | 创建工作流 - 名称为空 | 无 | 1. 调用 POST /workflows<br>2. name 为空 | 返回 400，验证错误 | P0 |
| **TC-WF-004** | 创建工作流 - 名称重复 | 已存在同名 workflow | 1. 调用 POST /workflows<br>2. 传入已存在的 name | 返回 409，错误信息 | P0 |
| **TC-WF-005** | 创建工作流 - 无 steps | 无 | 1. 调用 POST /workflows<br>2. YAML 无 steps | 返回 400，验证错误 | P0 |
| **TC-WF-006** | 创建工作流 - Agent 不存在 | 无 | 1. 调用 POST /workflows<br>2. 引用不存在的 Agent | 返回 400，警告信息 | P1 |
| **TC-WF-007** | 创建工作流 - 变量引用错误 | 无 | 1. 调用 POST /workflows<br>2. input=${not_exist.output} | 返回 400，验证错误 | P1 |
| **TC-WF-008** | 创建工作流 - 循环依赖 | 无 | 1. 调用 POST /workflows<br>2. step1 依赖 step2，step2 依赖 step1 | 返回 400，检测循环依赖 | P1 |
| **TC-WF-009** | 创建工作流 - description 可选 | 无 | 1. 调用 POST /workflows<br>2. 不传 description | 返回 201，description=null | P2 |
| **TC-WF-010** | 创建工作流 - 复杂 YAML | 无 | 1. 调用 POST /workflows<br>2. 传入 10 个 steps 的 YAML | 返回 201，解析成功 | P1 |

---

#### 2.3.2 查询/更新工作流

| 用例 ID | 用例名称 | 前置条件 | 测试步骤 | 预期结果 | 优先级 |
|:--------|:---------|:---------|:---------|:---------|:------:|
| **TC-WF-011** | 查询工作流列表 | 存在 3 个 workflow | 1. 调用 GET /workflows | 返回 200，total=3 | P0 |
| **TC-WF-012** | 查询单个工作流 | 存在 wf_001 | 1. 调用 GET /workflows/wf_001 | 返回 200，返回完整信息 | P0 |
| **TC-WF-013** | 查询工作流 - 不存在 | 无 | 1. 调用 GET /workflows/not_exist | 返回 404 | P0 |
| **TC-WF-014** | 查询工作流 - 按状态筛选 | 存在 draft/published | 1. 调用 GET /workflows?status=published | 返回 200，只包含 published | P1 |
| **TC-WF-015** | 更新工作流配置 | 存在 wf_001 | 1. 调用 PUT /workflows/wf_001/config | 返回 200，配置已更新 | P0 |
| **TC-WF-016** | 发布工作流 | 存在 draft 的 workflow | 1. 调用 PATCH /workflows/wf_001/publish | 返回 200，status=published | P0 |
| **TC-WF-017** | 归档工作流 | 存在 published 的 workflow | 1. 调用 PATCH /workflows/wf_001/archive | 返回 200，status=archived | P1 |
| **TC-WF-018** | 删除工作流 - 成功 | 存在 wf_001 | 1. 调用 DELETE /workflows/wf_001 | 返回 200，success=true | P0 |
| **TC-WF-019** | 删除工作流 - 有执行中任务 | 存在 running 的 execution | 1. 调用 DELETE /workflows/wf_001 | 返回 400，提示有执行中任务 | P1 |
| **TC-WF-020** | 验证 YAML - 语法检查 | 无 | 1. 调用 POST /workflows/validate<br>2. 传入 YAML | 返回 200，验证结果 | P2 |

**测试代码示例：**
```python
# tests/test_workflows.py
@pytest.mark.asyncio
async def test_create_workflow(client):
    """TC-WF-001: 创建工作流 - 成功"""
    payload = {
        "name": "获客流程",
        "description": "FB 监控 → 意图识别 → 申请引导",
        "yaml_config": """
name: 获客流程
version: 1.0
steps:
  - id: step1
    agent: fb-monitor
    action: monitor
    output: posts
  - id: step2
    agent: intent-agent
    action: classify
    input: ${step1.output.posts}
    output: intents
"""
    }
    response = await client.post("/workflows", json=payload)
    assert response.status_code == 201
    data = response.json()
    assert "workflow_id" in data
    assert data["status"] == "draft"

@pytest.mark.asyncio
async def test_create_workflow_invalid_yaml(client):
    """TC-WF-002: 创建工作流 - YAML 格式错误"""
    payload = {
        "name": "test-workflow",
        "yaml_config": "invalid: yaml: :"  # 无效 YAML
    }
    response = await client.post("/workflows", json=payload)
    assert response.status_code == 400
    assert "YAML 格式错误" in response.json()["detail"]
```

---

### 2.4 工作流执行（25 个用例）

#### 2.4.1 执行工作流

| 用例 ID | 用例名称 | 前置条件 | 测试步骤 | 预期结果 | 优先级 |
|:--------|:---------|:---------|:---------|:---------|:------:|
| **TC-EX-001** | 执行工作流 - 成功 | 存在 published workflow | 1. 调用 POST /workflows/wf_001/execute | 返回 202，execution_id 不为空 | P0 |
| **TC-EX-002** | 执行工作流 - workflow 不存在 | 无 | 1. 调用 POST /workflows/not_exist/execute | 返回 404 | P0 |
| **TC-EX-003** | 执行工作流 - draft 状态 | 存在 draft workflow | 1. 调用 POST /workflows/wf_001/execute | 返回 400，提示未发布 | P0 |
| **TC-EX-004** | 执行工作流 - 带输入数据 | 存在 workflow | 1. 调用 POST /workflows/wf_001/execute<br>2. 传入 input_data | 返回 202，input_data 保存 | P0 |
| **TC-EX-005** | 执行工作流 - 无输入数据 | 存在 workflow | 1. 调用 POST /workflows/wf_001/execute<br>2. 不传 input_data | 返回 202，input_data=null | P1 |
| **TC-EX-006** | 执行工作流 - 并发执行 | 存在 workflow | 1. 并发发送 10 个执行请求 | 返回 202，生成 10 个 execution_id | P1 |
| **TC-EX-007** | 执行工作流 - Agent 离线 | workflow 引用 offline Agent | 1. 调用 POST /workflows/wf_001/execute | 返回 400，提示 Agent 离线 | P1 |
| **TC-EX-008** | 执行工作流 - 超时处理 | 存在 workflow | 1. 执行 workflow<br>2. 等待超时 | 返回 failed，error_message 提示超时 | P1 |

---

#### 2.4.2 查询执行状态

| 用例 ID | 用例名称 | 前置条件 | 测试步骤 | 预期结果 | 优先级 |
|:--------|:---------|:---------|:---------|:---------|:------:|
| **TC-EX-009** | 查询执行状态 - pending | 刚提交执行 | 1. 调用 GET /executions/exec_001/status | 返回 200，status=pending | P0 |
| **TC-EX-010** | 查询执行状态 - running | 执行中 | 1. 调用 GET /executions/exec_001/status | 返回 200，status=running, current_step=step2 | P0 |
| **TC-EX-011** | 查询执行状态 - completed | 执行完成 | 1. 调用 GET /executions/exec_001/status | 返回 200，status=completed, output_data 存在 | P0 |
| **TC-EX-012** | 查询执行状态 - failed | 执行失败 | 1. 调用 GET /executions/exec_001/status | 返回 200，status=failed, error_message 存在 | P0 |
| **TC-EX-013** | 查询执行状态 - 不存在 | 无 | 1. 调用 GET /executions/not_exist/status | 返回 404 | P0 |
| **TC-EX-014** | 查询执行状态 - 进度信息 | 执行中 | 1. 调用 GET /executions/exec_001/status | 返回 progress 字段（百分比） | P1 |
| **TC-EX-015** | 查询执行状态 - 耗时信息 | 执行完成 | 1. 调用 GET /executions/exec_001/status | 返回 duration 字段（秒） | P1 |

---

#### 2.4.3 执行日志

| 用例 ID | 用例名称 | 前置条件 | 测试步骤 | 预期结果 | 优先级 |
|:--------|:---------|:---------|:---------|:---------|:------:|
| **TC-EX-016** | 查询执行日志 - 成功 | 执行完成 | 1. 调用 GET /executions/exec_001/logs | 返回 200，包含所有步骤日志 | P0 |
| **TC-EX-017** | 查询执行日志 - 不存在 | 无 | 1. 调用 GET /executions/not_exist/logs | 返回 404 | P0 |
| **TC-EX-018** | 查询执行日志 - 步骤详情 | 执行完成 | 1. 调用 GET /executions/exec_001/logs?step=step1 | 返回 200，只包含 step1 日志 | P1 |
| **TC-EX-019** | 执行日志 - 输入输出记录 | 执行完成 | 1. 调用 GET /executions/exec_001/logs | 返回 step_input 和 step_output | P1 |
| **TC-EX-020** | 执行日志 - 错误信息 | 执行失败 | 1. 调用 GET /executions/exec_001/logs | 返回 error_message 字段 | P0 |

---

#### 2.4.4 执行控制

| 用例 ID | 用例名称 | 前置条件 | 测试步骤 | 预期结果 | 优先级 |
|:--------|:---------|:---------|:---------|:---------|:------:|
| **TC-EX-021** | 重试执行 - 成功 | 存在 failed execution | 1. 调用 POST /executions/exec_001/retry | 返回 202，新 execution_id | P1 |
| **TC-EX-022** | 重试执行 - 非失败状态 | 存在 completed execution | 1. 调用 POST /executions/exec_001/retry | 返回 400，提示不能重试 | P1 |
| **TC-EX-023** | 取消执行 - 成功 | 存在 running execution | 1. 调用 POST /executions/exec_001/cancel | 返回 200，status=cancelled | P1 |
| **TC-EX-024** | 取消执行 - 已完成 | 存在 completed execution | 1. 调用 POST /executions/exec_001/cancel | 返回 400，提示已完成 | P1 |
| **TC-EX-025** | 变量传递 - 步骤间数据 | 存在多步骤 workflow | 1. 执行 workflow<br>2. 验证 step2 收到 step1 输出 | output 数据正确传递 | P0 |

**测试代码示例：**
```python
# tests/test_executions.py
@pytest.mark.asyncio
async def test_execute_workflow(client):
    """TC-EX-001: 执行工作流 - 成功"""
    # 先创建并发布工作流
    workflow = await create_and_publish_workflow(client)
    
    # 执行
    response = await client.post(f"/workflows/{workflow['workflow_id']}/execute", json={})
    assert response.status_code == 202
    data = response.json()
    assert "execution_id" in data
    assert data["status"] == "pending"

@pytest.mark.asyncio
async def test_query_execution_status(client):
    """TC-EX-011: 查询执行状态 - completed"""
    execution = await execute_workflow(client)
    
    # 等待完成
    for _ in range(30):
        response = await client.get(f"/executions/{execution['execution_id']}/status")
        if response.json()["status"] == "completed":
            break
        await asyncio.sleep(1)
    
    assert response.json()["status"] == "completed"
    assert "output_data" in response.json()

@pytest.mark.asyncio
async def test_variable_passing(client):
    """TC-EX-025: 变量传递 - 步骤间数据"""
    # 创建工作流，step2 依赖 step1 的输出
    workflow_payload = {
        "name": "变量传递测试",
        "yaml_config": """
name: 变量传递测试
steps:
  - id: step1
    agent: test-agent-1
    action: process
    output: result1
  - id: step2
    agent: test-agent-2
    action: process
    input: ${step1.output.result1}
    output: result2
"""
    }
    workflow = await create_workflow(client, workflow_payload)
    
    # 执行并验证
    execution = await execute_workflow(client, workflow["workflow_id"])
    
    # 等待完成
    await wait_execution_complete(client, execution["execution_id"])
    
    # 查询日志，验证变量传递
    response = await client.get(f"/executions/{execution['execution_id']}/logs")
    logs = response.json()["logs"]
    assert len(logs) == 2
    assert logs[1]["step_input"] == logs[0]["step_output"]
```

---

### 2.5 管理后台（15 个用例）

| 用例 ID | 用例名称 | 前置条件 | 测试步骤 | 预期结果 | 优先级 |
|:--------|:---------|:---------|:---------|:---------|:------:|
| **TC-AD-001** | 启用 Agent | 存在 disabled Agent | 1. 调用 PATCH /agents/agent_001/enable | 返回 200，status=enabled | P1 |
| **TC-AD-002** | 禁用 Agent | 存在 enabled Agent | 1. 调用 PATCH /agents/agent_001/disable | 返回 200，status=disabled | P1 |
| **TC-AD-003** | 发布工作流 | 存在 draft workflow | 1. 调用 PATCH /workflows/wf_001/publish | 返回 200，status=published | P1 |
| **TC-AD-004** | 归档工作流 | 存在 published workflow | 1. 调用 PATCH /workflows/wf_001/archive | 返回 200，status=archived | P1 |
| **TC-AD-005** | 重试执行 | 存在 failed execution | 1. 调用 POST /executions/exec_001/retry | 返回 202，新 execution_id | P1 |
| **TC-AD-006** | 取消执行 | 存在 running execution | 1. 调用 POST /executions/exec_001/cancel | 返回 200，status=cancelled | P1 |
| **TC-AD-007** | 查询日志 - 全部 | 存在 execution | 1. 调用 GET /executions/exec_001/logs | 返回 200，完整日志 | P1 |
| **TC-AD-008** | 查询日志 - 筛选 | 存在 execution | 1. 调用 GET /executions/exec_001/logs?level=error | 返回 200，只包含 error 级别 | P2 |
| **TC-AD-009** | 导出日志 | 存在 execution | 1. 调用 GET /executions/exec_001/logs?export=csv | 返回 200，CSV 格式 | P2 |
| **TC-AD-010** | 系统配置 - 心跳间隔 | 管理员权限 | 1. 调用 PUT /config/heartbeat_interval | 返回 200，配置已更新 | P2 |
| **TC-AD-011** | 系统配置 - 超时时间 | 管理员权限 | 1. 调用 PUT /config/heartbeat_timeout | 返回 200，配置已更新 | P2 |
| **TC-AD-012** | 系统配置 - 权限校验 | 普通用户权限 | 1. 调用 PUT /config/heartbeat_interval | 返回 403，权限不足 | P1 |
| **TC-AD-013** | 批量操作 - 启用多个 Agent | 存在多个 disabled Agent | 1. 调用 POST /agents/batch/enable | 返回 200，全部启用 | P2 |
| **TC-AD-014** | 批量操作 - 禁用多个 Agent | 存在多个 enabled Agent | 1. 调用 POST /agents/batch/disable | 返回 200，全部禁用 | P2 |
| **TC-AD-015** | 批量操作 - 部分失败 | 存在部分不存在的 Agent | 1. 调用 POST /agents/batch/enable | 返回 207，部分成功 | P2 |

---

### 2.6 数据看板（8 个用例）

| 用例 ID | 用例名称 | 前置条件 | 测试步骤 | 预期结果 | 优先级 |
|:--------|:---------|:---------|:---------|:---------|:------:|
| **TC-DB-001** | 查询统计数据 - Agent | 存在 10 个 Agent | 1. 调用 GET /dashboard/stats | 返回 agents.total=10 | P2 |
| **TC-DB-002** | 查询统计数据 - Workflow | 存在 5 个 workflow | 1. 调用 GET /dashboard/stats | 返回 workflows.total=5 | P2 |
| **TC-DB-003** | 查询统计数据 - Execution | 存在 100 个 execution | 1. 调用 GET /dashboard/stats | 返回 executions.today=100 | P2 |
| **TC-DB-004** | 查询趋势数据 - 7 天 | 存在历史数据 | 1. 调用 GET /dashboard/trends?days=7 | 返回 7 天的趋势数据 | P2 |
| **TC-DB-005** | 查询趋势数据 - 30 天 | 存在历史数据 | 1. 调用 GET /dashboard/trends?days=30 | 返回 30 天的趋势数据 | P2 |
| **TC-DB-006** | 查询成功率统计 | 存在成功/失败 execution | 1. 调用 GET /dashboard/stats | 返回 success_rate 字段 | P2 |
| **TC-DB-007** | 查询平均耗时统计 | 存在 completed execution | 1. 调用 GET /dashboard/stats | 返回 avg_duration 字段 | P2 |
| **TC-DB-008** | 查询数据 - 权限校验 | 普通用户权限 | 1. 调用 GET /dashboard/stats | 返回 200，viewer 角色可访问 | P2 |

---

### 2.7 权限控制（集成在以上用例中）

| 用例 ID | 用例名称 | 前置条件 | 测试步骤 | 预期结果 | 优先级 |
|:--------|:---------|:---------|:---------|:---------|:------:|
| **TC-RB-001** | 管理员 - 全部权限 | admin 角色 | 1. 使用 admin token 调用所有 API | 全部返回 200 | P1 |
| **TC-RB-002** | 开发人员 - Agent 管理 | developer 角色 | 1. 使用 developer token 调用 Agent API | 返回 200 | P1 |
| **TC-RB-003** | 开发人员 - 系统配置 | developer 角色 | 1. 使用 developer token 调用配置 API | 返回 403 | P1 |
| **TC-RB-004** | 运营人员 - 执行权限 | operator 角色 | 1. 使用 operator token 调用执行 API | 返回 200 | P1 |
| **TC-RB-005** | 运营人员 - 新建 Agent | operator 角色 | 1. 使用 operator token 调用新建 Agent API | 返回 403 | P1 |
| **TC-RB-006** | 查看者 - 只读权限 | viewer 角色 | 1. 使用 viewer token 调用查询 API | 返回 200 | P1 |
| **TC-RB-007** | 查看者 - 写入操作 | viewer 角色 | 1. 使用 viewer token 调用新建 API | 返回 403 | P1 |
| **TC-RB-008** | 未授权 - 无 token | 无 | 1. 调用 API 不带 token | 返回 401 | P0 |

---

## 三、性能测试（6 个用例）

### 3.1 压测场景

| 用例 ID | 用例名称 | 场景描述 | 并发数 | 持续时间 | 性能指标 | 优先级 |
|:--------|:---------|:---------|:------:|:---------|:---------|:------:|
| **TC-PR-001** | Agent 注册压测 | 1000 并发注册 Agent | 1000 | 60s | P95<100ms, 错误率<1% | P1 |
| **TC-PR-002** | 心跳压测 | 5000/s 心跳请求 | 5000 | 300s | P95<50ms, 错误率<0.1% | P1 |
| **TC-PR-003** | 工作流执行压测 | 500 并发执行工作流 | 500 | 300s | P95<500ms, 错误率<1% | P1 |
| **TC-PR-004** | 查询压测 | 2000 并发查询请求 | 2000 | 60s | P95<100ms, 错误率<0.5% | P1 |
| **TC-PR-005** | 混合场景压测 | 注册 + 心跳 + 执行混合 | 1000 | 600s | P95<200ms, 错误率<1% | P1 |
| **TC-PR-006** | 稳定性压测 | 持续运行 24 小时 | 500 | 24h | 无内存泄漏，无崩溃 | P1 |

**Locust 压测脚本示例：**
```python
# tests/locustfile.py
from locust import HttpUser, task, between

class AgentUser(HttpUser):
    wait_time = between(0.1, 0.5)
    
    @task(3)
    def heartbeat(self):
        """心跳请求"""
        self.client.post("/agents/agent_001/heartbeat")
    
    @task(2)
    def query_agent(self):
        """查询 Agent"""
        self.client.get("/agents")
    
    @task(1)
    def execute_workflow(self):
        """执行工作流"""
        self.client.post("/workflows/wf_001/execute", json={})

class WorkflowUser(HttpUser):
    wait_time = between(0.5, 1)
    
    @task
    def create_workflow(self):
        """创建工作流"""
        payload = {
            "name": "test-workflow",
            "yaml_config": "name: test\nsteps: []"
        }
        self.client.post("/workflows", json=payload)
```

**压测命令：**
```bash
# Agent 注册压测
locust -f tests/locustfile.py --host http://localhost:8000 --users 1000 --spawn-rate 100 --run-time 60s

# 心跳压测
locust -f tests/locustfile.py --host http://localhost:8000 --users 5000 --spawn-rate 500 --run-time 300s

# 工作流执行压测
locust -f tests/locustfile.py --host http://localhost:8000 --users 500 --spawn-rate 50 --run-time 300s
```

---

## 四、测试执行计划

### 4.1 测试阶段

| 阶段 | 时间 | 测试类型 | 负责人 | 通过标准 |
|:-----|:-----|:---------|:-------|:---------|
| **单元测试** | Phase 1-2 结束 | 单元测试 | 开发 | 覆盖率>80% |
| **集成测试** | Phase 3 结束 | 集成测试 | 测试 | 核心用例 100% 通过 |
| **性能测试** | Phase 4 | 压测 | 测试 + 开发 | 性能指标达标 |
| **验收测试** | Phase 4 | 业务验证 | 产品 + 测试 | 业务流程通过 |

### 4.2 每日测试计划

| 日期 | 测试内容 | 用例数 | 负责人 |
|:-----|:---------|:------:|:-------|
| Day 1-2 | Agent 新建与管理测试 | 25 | 开发 |
| Day 3-4 | 心跳检测测试 | 10 | 开发 |
| Day 5-7 | 工作流编排测试 | 20 | 开发 |
| Day 8-9 | 工作流执行测试 | 25 | 开发 |
| Day 10 | 管理后台测试 | 15 | 测试 |
| Day 11 | 数据看板测试 | 8 | 测试 |
| Day 12 | 权限控制测试 | 8 | 测试 |
| Day 13 | 集成测试 | 20 | 测试 |
| Day 14 | 性能测试 | 6 | 测试 + 开发 |
| Day 15 | 验收测试 | 10 | 产品 + 测试 |

### 4.3 缺陷等级定义

| 等级 | 定义 | 响应时间 | 修复时限 |
|:-----|:-----|:---------|:---------|
| **P0 - 致命** | 系统崩溃、数据丢失、核心功能不可用 | 立即 | 4 小时 |
| **P1 - 严重** | 主要功能不可用、性能严重下降 | 1 小时 | 24 小时 |
| **P2 - 一般** | 次要功能不可用、UI 问题 | 4 小时 | 3 天 |
| **P3 - 轻微** | 文案错误、体验问题 | 24 小时 | 1 周 |

---

## 五、测试报告模板

### 5.1 单元测试报告

```markdown
# 单元测试报告

## 测试概况
- 测试时间：2026-04-16
- 测试范围：Agent 管理模块
- 用例总数：35
- 通过数：35
- 失败数：0
- 跳过数：0
- 通过率：100%

## 覆盖率统计
- 行覆盖率：85%
- 分支覆盖率：80%
- 函数覆盖率：90%

## 失败用例详情
无

## 风险与建议
无
```

### 5.2 集成测试报告

```markdown
# 集成测试报告

## 测试概况
- 测试时间：2026-04-16
- 测试范围：全链路集成测试
- 用例总数：50
- 通过数：48
- 失败数：2
- 跳过数：0
- 通过率：96%

## 失败用例详情
| 用例 ID | 用例名称 | 失败原因 | 缺陷 ID |
|:--------|:---------|:---------|:--------|
| TC-EX-008 | 超时处理 | 超时时间配置未生效 | BUG-001 |
| TC-AD-015 | 批量操作部分失败 | 返回码应为 207 但返回 200 | BUG-002 |

## 风险与建议
1. 超时处理逻辑需要修复
2. 批量操作返回码需要调整
```

### 5.3 性能测试报告

```markdown
# 性能测试报告

## 测试概况
- 测试时间：2026-04-16
- 测试场景：心跳压测
- 并发用户：5000
- 持续时间：300s

## 性能指标
| 指标 | 目标值 | 实际值 | 状态 |
|:-----|:------:|:------:|:----:|
| P50 响应时间 | <50ms | 35ms | ✅ |
| P95 响应时间 | <100ms | 85ms | ✅ |
| P99 响应时间 | <200ms | 150ms | ✅ |
| 错误率 | <0.1% | 0.05% | ✅ |
| TPS | >5000 | 5200 | ✅ |

## 资源使用
| 资源 | 峰值使用 | 阈值 | 状态 |
|:-----|:--------:|:----:|:----:|
| CPU | 65% | 80% | ✅ |
| 内存 | 1.5GB | 2GB | ✅ |
| 数据库连接 | 80 | 100 | ✅ |

## 结论
性能指标全部达标，系统可以承受 5000/s 心跳请求
```

---

## 六、附录

### 6.1 测试数据准备

```python
# tests/conftest.py
import pytest

@pytest.fixture
async def client():
    """测试客户端"""
    async with TestClient(app) as client:
        yield client

@pytest.fixture
async def test_agent(client):
    """测试 Agent"""
    payload = {"name": "test-agent", "type": "a2a", "config": {}}
    response = await client.post("/agents", json=payload)
    return response.json()

@pytest.fixture
async def test_workflow(client):
    """测试工作流"""
    payload = {
        "name": "test-workflow",
        "yaml_config": "name: test\nsteps: []"
    }
    response = await client.post("/workflows", json=payload)
    return response.json()
```

### 6.2 测试数据库配置

```python
# tests/test_config.py
TEST_DATABASE_URL = "sqlite:///./test.db"

@pytest.fixture
async def test_db():
    """测试数据库"""
    async with create_test_database() as db:
        yield db
        await drop_test_database()
```

### 6.3 测试用例编号规则

```
TC-{模块}-{序号}

模块代码:
- AG: Agent 管理
- HB: 心跳检测
- WF: 工作流编排
- EX: 工作流执行
- AD: 管理后台
- DB: 数据看板
- RB: 权限控制
- PR: 性能测试
```

---

**文档版本：v1.0**  
**创建时间：2026-04-16**  
**创建者：小宁（RAKkDm）**  
**状态：待执行**
