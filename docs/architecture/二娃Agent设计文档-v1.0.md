# 二娃Agent设计文档

**版本：v1.0**
**创建日期：2026-04-28**
**状态：MVP阶段**

---

## 一、身份定位

| 维度 | 内容 |
|------|------|
| **代号** | Outreach Agent |
| **昵称** | 二娃 |
| **职责** | 从情报到执行，把已验证目标变成转化用户 |
| **上游** | 大娃（采集 + 清洗 + 验证） |
| **下游** | 进件系统（获客结果转化） |

---

## 二、智能体架构

### 2.1 完整文件结构

```
outreach_agent_v1.0/
│
├── config/
│   ├── settings.py              # 全局参数配置
│   │   ├── 飞书配置（app_id/secret/触达表ID）
│   │   ├── 状态机参数（48h/72h/3次/24h）
│   │   ├── 风控参数（15人/天、3-5min间隔）
│   │   ├── 账号池（ACCOUNTS = [{profile_id, daily_limit}]）
│   │   └── 链接配置（落地页URL + UTM模板）
│   │
│   ├── templates.py             # 话术模板库
│   │   ├── DM话术（按语义标签分类，每类2-3套）
│   │   ├── 评论回复话术（3套通用）
│   │   ├── 异议处理话术（HESITANT用户专用）
│   │   └── 二次触达话术（跟进专用）
│   │
│   └── prompts.py               # LLM提示词
│       └── RESPONSE_CLASSIFIER_PROMPT（响应分类，唯一一个）
│
├── scripts/
│   ├── handoff.py               # Skill: outreach-handoff
│   │   ├── 读大娃采集表（状态=已验证）
│   │   ├── 复制9个字段到触达表
│   │   ├── 触达表状态=待触达
│   │   ├── 采集表状态→已移交
│   │   └── 防重复（检查采集记录ID是否已存在）
│   │
│   ├── outreach_run.py          # Skill: outreach-run
│   │   ├── 选人引擎（优先级排序 + 冷却过滤 + 通道过滤）
│   │   ├── 策略引擎（语义→话术 + 变量填充 + 链接决策）
│   │   ├── 执行调度（账号选择 + AdsPower启动 + 发送）
│   │   ├── 结果处理（更新触达表 + 设置冷却）
│   │   └── 文件锁（防并发）
│   │
│   ├── check_responses.py       # Skill: outreach-followup（第一步）
│   │   ├── 遍历触达表（状态=触达中 + 超过48h）
│   │   ├── AdsPower打开Messenger逐个检查
│   │   ├── 提取新消息内容
│   │   ├── 关键词快速判断（60%场景）
│   │   ├── LLM分类（40%模糊场景）
│   │   └── 更新触达表（状态/响应内容/响应时间）
│   │
│   └── follow_up.py             # Skill: outreach-followup（第二步）
│       ├── 遍历触达表（状态=未响应 + 触达次数<3 + 冷却已过）
│       ├── 选二次触达话术（换模板）
│       ├── 执行发送
│       ├── 触达次数≥3 → 状态=已关闭
│       └── 更新触达表
│
├── tools/
│   ├── adspower_api.py          # AdsPower封装
│   │   ├── start_browser(profile_id) → ws_url
│   │   ├── stop_browser(profile_id)
│   │   └── pick_account(accounts) → 可用账号
│   │
│   ├── fb_messenger.py          # DM发送能力
│   │   ├── send_dm(page, user_url, text) → 成功/失败
│   │   ├── check_dm_response(page, user_url) → 新消息列表
│   │   └── has_message_button(page) → bool
│   │
│   ├── fb_comment.py            # 评论回复能力
│   │   ├── reply_comment(page, post_url, user_name, text) → 成功/失败
│   │   └── check_comment_reply(page, post_url, our_comment) → 回复列表
│   │
│   ├── response_classifier.py   # 响应分类器
│   │   ├── classify_by_keywords(text) → 标签或None
│   │   ├── classify_by_llm(text, context) → 标签
│   │   └── classify(text, context) → 关键词优先，未命中走LLM
│   │
│   ├── feishu_reader.py         # 飞书读取
│   │   ├── get_pending_users() → 待触达列表
│   │   ├── get_in_progress_users() → 触达中列表
│   │   └── get_record(record_id) → 单条记录
│   │
│   └── feishu_writer.py         # 飞书写入
│       ├── create_record(fields) → record_id
│       ├── update_record(record_id, fields)
│       └── record_exists(source_record_id) → bool（防重复移交）
│
├── skills/                      # Kiro Skill定义
│   ├── outreach-handoff/
│   │   └── SKILL.md             # 触发: #outreach-handoff
│   ├── outreach-run/
│   │   └── SKILL.md             # 触发: #outreach-run
│   └── outreach-followup/
│       └── SKILL.md             # 触发: #outreach-followup
│
├── data/
│   ├── .outreach_run.lock       # 文件锁
│   ├── .outreach_handoff.lock
│   └── .outreach_followup.lock
│
├── .env                         # 凭证（飞书API/AdsPower）
└── .env.example                 # 凭证模板
```

### 2.2 模块职责

| 层 | 包含文件 | 定位 | 是否依赖LLM |
|----|---------|------|------------|
| **config/** | settings.py / templates.py / prompts.py | 配置层：所有参数和模板集中管理 | prompts.py（仅响应分类） |
| **scripts/** | handoff.py / outreach_run.py / check_responses.py / follow_up.py | 业务层：每个脚本对应一个Skill入口 | check_responses.py（模糊场景走LLM） |
| **tools/** | adspower_api.py / fb_messenger.py / fb_comment.py / response_classifier.py / feishu_reader.py / feishu_writer.py | 能力层：封装外部系统和操作能力 | response_classifier.py（关键词兜底 + LLM兜底） |
| **skills/** | outreach-handoff/ / outreach-run/ / outreach-followup/ | 入口层：Skill描述，定义触发方式和参数 | 无 |

### 2.3 调用链路

```
                  ┌──────────────────────────────────┐
                  │      飞书多维表格（4张表）        │
                  │  触达用户表 / 话术模板表          │
                  │  账号管理表 / 触达统计表          │
                  └──────────┬───────────┬───────────┘
                             │           │
              ┌──────────────┘           └──────────────┐
              │                                          │
         ┌────▼─────┐                          ┌───────▼───────┐
         │ handoff  │                          │  outreach_run  │
         │ 移交脚本  │                          │  触达执行脚本   │
         └────┬─────┘                          └───────┬───────┘
              │                                        │
              │ 读采集表(已验证) → 写入触达表(待触达)     │ 选人→选话术→账号调度→发送
              │                                        │
              └────────────┬───────────────────────────┘
                           │
                    ┌──────▼──────┐
                    │ check_responses  │
                    │ 响应检查 (48h后) │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  follow_up  │
                    │  二次触达     │
                    └─────────────┘
```

### 2.4 LLM使用范围（最小化原则）

二娃仅在**一处**使用 LLM，且做了两层兜底：

```
用户回复
  ↓
  关键词快速判断（覆盖 60% 明确场景）
  ├── 命中拒绝词 → 已关闭
  ├── 命中强兴趣词 → 已响应，发进件链接
  └── 未命中 → 调 LLM
        ↓
        LLM 分类（处理 40% 模糊场景）
        ├── INTERESTED  → 已响应，发进件链接
        ├── HESITANT    → 发异议处理模板
        ├── REJECTED    → 已关闭
        └── IRRELEVANT  → 不发模板，记录即可
```

LLM 只做一件事：**四分类**。不生成话术、不做决策、不分析情绪。

---

## 三、数据边界

### 3.1 领地划分

| 领地 | Agent | 操作权限 |
|------|--------|---------|
| 大娃领地 | 大娃 | 读写（采集表、采集用户表） |
| 二娃领地 | 二娃 | 读写（触达用户表） |
| 共用资源 | 大娃 + 二娃 | 只读（账号矩阵、IP矩阵） |

```
大娃领地（只读）          二娃领地（读写）
┌────────────┐          ┌────────────┐
│ KOL采集表   │          │ 触达用户表  │  ← 二娃唯一操作的表
│ 采集用户表  │          │            │
└────────────┘          └────────────┘
        ↓ 已验证                ↑
    移交脚本（handoff.py）──────┘
```

### 3.2 触达用户表字段（22个）

#### 继承字段（9个，移交时写入，二娃只读）

| 字段 | 类型 | 来源 |
|------|------|------|
| 用户名 | 文本 | 大娃移交 |
| 用户主页 | 超链接 | 大娃移交 |
| 用户编号 | 文本 | 大娃移交 |
| 来源KOL | 文本 | 大娃移交 |
| 语义标签 | 文本 | 大娃移交 |
| 意图评分 | 数字 | 大娃移交 |
| 联系方式 | 文本 | 大娃移交 |
| 原始内容 | 文本 | 大娃移交 |
| 采集记录ID | 文本 | 大娃移交 |

#### 触达过程字段（7个，二娃读写）

| 字段 | 类型 | 说明 |
|------|------|------|
| 触达状态 | 单选 | 待触达/触达中/已响应/已转化/已关闭 |
| 触达次数 | 数字 | 累计0-N |
| 首次触达时间 | 日期 | YYYY-MM-DD HH:MM |
| 最后触达时间 | 日期 | YYYY-MM-DD HH:MM |
| 触达通道 | 单选 | FB_DM/评论回复 |
| 最后触达内容 | 文本 | 发送的话术 |
| 响应时间 | 日期 | 用户回复时间 |

#### 补充字段（3个，二娃读写）

| 字段 | 类型 | 说明 |
|------|------|------|
| 响应内容 | 文本 | 用户回复了什么（归因分析用） |
| 失败原因 | 文本 | 被封/被拦/不存在/超时 |
| 冷却截止 | 日期 | 下次可触达最早时间（风控用） |

#### 策略字段（3个，二娃读写）

| 字段 | 类型 | 说明 |
|------|------|------|
| 话术模板ID | 文本 | 用了哪个模板 |
| 账号ID | 文本 | 用哪个账号发的 |
| 优先级分数 | 数字 | 选人排序用 |

---

## 四、状态机

### 4.1 状态流转图

```
待触达 ──→ 触达中 ──→ 已响应 ──→ 已转化
              │           │
              ↓           ↓
           未响应      未转化
              │           │
              ↓           ↓
          二次触达    二次触达/已关闭
              │
              ↓
        超过N次无响应 ──→ 已关闭
```

### 4.2 状态参数（配置化）

| 参数 | 值 | 配置字段名 | 说明 |
|------|-----|-----------|------|
| 响应判定窗口 | 48h | `response_window_hours` | 触达中多久没回复算未响应 |
| 二次触达间隔 | 72h | `followup_interval_hours` | 未响应后多久发二次触达 |
| 触达上限 | 3次 | `max_attempts` | 同用户最多触达次数 |
| 冷却时间 | 24h | `cooldown_hours` | 同用户两次触达间隔 |

### 4.3 时间线示例

```
Day 0     首次触达 → 状态=触达中
          │
Day 2     48h无响应 → 状态=未响应
          │ 冷却24h
Day 3     二次触达 → 状态=触达中，触达次数=2
          │
Day 5     48h无响应 → 状态=未响应
          │ 冷却24h
Day 6     三次触达 → 状态=触达中，触达次数=3
          │
Day 8     48h无响应 → 触达次数=3 ≥ 上限 → 状态=已关闭
```

最坏情况8天关闭一个用户，不会无限占用资源。

### 4.4 明确拒绝判定

用户回复包含以下信号直接关闭，不等N次：

- "no me interesa" / "no gracias" / "no quiero"
- 拉黑/屏蔽（消息发送失败）
- "deja de escribirme" / "no me escribas"

---

## 五、核心流程

### 5.1 全链路总览

```
大娃已验证用户
      ↓
① 移交（handoff）
  采集表 → 触达表，状态=待触达
      ↓
② 选人（优先级排序 + 冷却过滤）
  意图评分 × 语义权重 → 排序
      ↓
③ 触达执行
  ├── 评论回复（有评论记录）→ 可发链接
  └── DM发送（无评论记录）→ 首条不发链接
      ↓
④ 响应处理（check_responses）
  ├── INTERESTED → 状态=意向客户
  ├── HESITANT → 异议处理 → 跟进
  ├── REJECTED → 状态=已拒绝
  └── IRRELEVANT → 状态=已冷却
      ↓
⑤ 跟进（follow_up）
  48h无响应 / HESITANT异议处理 → 二次/三次触达
      ↓
  触达统计表更新 → 账号池重置
```

### 5.2 各环节详情

| 环节 | 输入 | 执行者 | 输出 | 关键参数 |
|------|------|--------|------|---------|
| ① 移交 | 采集表（状态=已验证） | handoff.py | 触达表（状态=待触达） | 继承9个字段 |
| ② 选人 | 触达表（状态=待触达） | outreach_run.py | 排序队列 | 意图评分×语义权重 |
| ③ 触达 | 排序队列 + 话术模板 | fb_messenger.py / fb_comment.py | FB消息/评论发送 | 15人/天/账号 |
| ④ 响应 | FB用户回复 | check_responses.py | 状态变更 | 四分类 |
| ⑤ 跟进 | 响应结果 | follow_up.py | 二次/三次触达 | 48h/72h冷却 |

### 5.3 数据流

```
采集表（大娃）
  │ handoff.py
  ▼
触达表（待触达）
  │ outreach_run.py（选人）
  ▼
触达表（触达中）
  │ fb_messenger.py → FB用户
  ▼
用户回复
  │ check_responses.py（分类）
  ▼
触达表（意向/拒绝/冷却）
  │ follow_up.py（跟进）
  ▼
触达表（二次触达...）
  │
  └──→ 触达统计表（累计数据）
```

### 5.4 时间线

```
T+0:   移交完成 → 触发outreach-run
T+5min: 选人完成 → 首条DM发出
T+48h:  无响应 → 自动触发follow-up（二次触达）
T+120h: 二次触达后仍无响应 → 三次触达（最后一次）
T+168h: 三次触达后无响应 → 状态=冷却（冷却期24h）
T+192h: 冷却期结束 → 状态重置为待触达（再次进入选人池）

若用户在任意时间点回复：
  INTERESTED → 状态=意向客户（不再触达）
  HESITANT → 异议处理话术 → 72h后跟进
  REJECTED → 状态=已拒绝（不再触达）
```

### 5.5 与数据边界和状态机的关系

| 关联章节 | 关系 |
|---------|------|
| **三、数据边界** | 核心流程中流转的数据字段定义在数据边界章节 |
| **四、状态机** | 核心流程中每个环节触发状态变更，状态机定义流转规则 |
| **六、通道策略** | 核心流程③触达执行环节根据通道策略选择触达方式 |

> 核心流程是"怎么跑"，数据边界是"跑什么"，状态机是"跑到哪"。三者构成完整的执行坐标系。

---

## 六、通道策略

### 6.1 通道优先级逻辑

```
有评论记录？
  ├── 是 → 评论回复优先（可发链接）
  │        ↓
  │        用户响应 → DM跟进（发进件链接）
  │
  └── 否 → 检查是否有FB主页
           ├── 有 → DM首条（不发链接）
           │        ↓
           │        用户响应 → 发进件链接
           │        ↓
           │        无响应 → 评论回复补充
           │
           └── 否 → 跳过（无可用通道）
```

### 6.2 链接策略

| 时机 | 链接 | 原因 |
|------|------|------|
| DM首条 | 不发 | FB拦截率高，先建立对话 |
| DM用户响应后 | 发 | 已建立对话，不会被拦截 |
| 评论回复 | 可以发 | 公开场合FB不拦截 |

### 6.3 UTM追踪参数

```
基础链接：https://xxx.com/apply
完整链接：https://xxx.com/apply?utm_source=fb_comment&utm_medium=outreach&utm_content={用户ID}&utm_term={话术ID}
```

| 参数 | 值 | 用途 |
|------|-----|------|
| `utm_source` | fb_comment / fb_dm | 渠道来源 |
| `utm_medium` | outreach | 标记触达流量 |
| `utm_content` | 用户ID | 具体哪个用户 |
| `utm_term` | 话术模板ID | 用了哪个话术 |

---

## 七、选人优先级

### 7.1 优先级算法

```
优先级分数 = 意图评分 × 语义权重
```

### 7.2 语义权重表

| 语义标签 | 权重 | 理由 |
|---------|------|------|
| 求购 | 强烈 | 1.5 | 最高优先，明确要买 |
| 求购 | 中等 | 1.3 | 有意向但不急 |
| 咨询 | 强烈 | 1.2 | 主动问问题，接近求购 |
| 咨询 | 中等 | 1.0 | 基准线 |
| 投诉 | 强烈 | 1.1 | 强切换意愿 |
| 投诉 | 中等 | 0.9 | 有不满但不急 |
| 求助 | * | 0.8 | 有需求但不确定是目标 |
| 分享/推荐 | 0.5 | 低优先 |
| 闲聊 | 0.3 | 基本不触达 |

### 7.3 示例计算

| 用户 | 意图评分 | 语义权重 | 优先级分数 | 排序 |
|------|---------|---------|-----------|------|
| A | 0.8 | 1.5（求购强烈） | 1.20 | 1 |
| B | 0.7 | 1.1（投诉强烈） | 0.77 | 2 |
| C | 0.5 | 1.0（咨询中等） | 0.50 | 3 |
| D | 0.4 | 0.8（求助） | 0.32 | 4 |

### 7.4 时间衰减（暂不加）

触发条件：待触达队列 > 50人时再加

```
优先级分数 = 意图评分 × 语义权重 × max(0.3, 1 - 天数/30)
```

---

## 八、话术模板

### 8.1 DM话术模板（按语义标签分多套）

```python
# config/templates.py

TEMPLATES = {
    # 求购强烈 - 直接推产品
    "求购_强烈": [
        {
            "id": "buy_hot_01",
            "text": "Hola {用户名}, vi tu comentario \"{原始内容}\" en el post de {来源KOL}. Podemos ayudarte con eso. ¿Te interesa que te envíe más info?",
            "tone": "direct",
        },
        {
            "id": "buy_hot_02",
            "text": "{用户名}, justo vi que comentaste \"{原始内容}\" en {来源KOL}. Tenemos solución para lo que buscas. ¿Quieres que te cuente cómo funciona?",
            "tone": "direct",
        },
        {
            "id": "buy_hot_03",
            "text": "Hola {用户名} 👋 Leí tu mensaje \"{原始内容}\" en {来源KOL}. Creo que podemos ayudarte. Si te interesa, puedo enviarte los detalles ahora.",
            "tone": "direct",
        },
    ],
    
    # 求购中等 - 提供信息引导
    "求购_中等": [
        {
            "id": "buy_mid_01",
            "text": "Hola {用户名}, vi tu comentario \"{原始内容}\" en {来源KOL}. Parece que estás buscando algo similar. ¿Te gustaría saber más opciones?",
            "tone": "informative",
        },
        {
            "id": "buy_mid_02",
            "text": "{用户名}, vi que mencionaste \"{原始内容}\" en el post de {来源KOL}. Hay varias alternativas que podrían funcionarte. ¿Quieres que te explique?",
            "tone": "informative",
        },
    ],
    
    # 咨询强烈 - 回答问题建立信任
    "咨询_强烈": [
        {
            "id": "ask_hot_01",
            "text": "Hola {用户名}, vi tu pregunta \"{原始内容}\" en {来源KOL}. Es una duda común. Te puedo explicar cómo funciona, ¿tienes 2 minutos?",
            "tone": "helpful",
        },
        {
            "id": "ask_hot_02",
            "text": "{用户名}, leí \"{原始内容}\" en {来源KOL}. Buena pregunta. Mucha gente tiene la misma duda. ¿Quieres que te aclare los detalles?",
            "tone": "helpful",
        },
        {
            "id": "ask_hot_03",
            "text": "Hola {用户名} 👋 Tu comentario \"{原始内容}\" en {来源KOL} me pareció interesante. Te respondo esa duda si quieres, es rápido.",
            "tone": "helpful",
        },
    ],
    
    # 咨询中等 - 回答问题
    "咨询_中等": [
        {
            "id": "ask_mid_01",
            "text": "Hola {用户名}, vi \"{原始内容}\" en {来源KOL}. Si tienes dudas sobre eso, puedo ayudarte. ¿Qué necesitas saber?",
            "tone": "helpful",
        },
        {
            "id": "ask_mid_02",
            "text": "{用户名}, noté tu mensaje \"{原始内容}\" en {来源KOL}. Hay información que te puede servir. ¿Te interesa?",
            "tone": "helpful",
        },
    ],
    
    # 投诉强烈 - 共情 + 替代方案
    "投诉_强烈": [
        {
            "id": "comp_hot_01",
            "text": "Hola {用户名}, leí \"{原始内容}\" en {来源KOL}. Entiendo tu frustración, eso no debería pasar. Hay alternativas mejores. ¿Te cuento?",
            "tone": "empathic",
        },
        {
            "id": "comp_hot_02",
            "text": "{用户名}, vi tu comentario \"{原始内容}\" en {来源KOL}. Qué mal experiencia. Te entiendo perfectamente. Si quieres, puedo mostrarte otra opción.",
            "tone": "empathic",
        },
    ],
    
    # 投诉中等 - 轻度共情 + 引导
    "投诉_中等": [
        {
            "id": "comp_mid_01",
            "text": "Hola {用户名}, \"{原始内容}\" en {来源KOL} - entiendo que fue difícil. Hay opciones que funcionan mejor. ¿Te interesa conocerlas?",
            "tone": "empathic_light",
        },
        {
            "id": "comp_mid_02",
            "text": "{用户名}, leí \"{原始内容}\" en {来源KOL}. Siento que pasó eso. Si quieres algo diferente, puedo sugerirte alternativas.",
            "tone": "empathic_light",
        },
    ],
    
    # 求助 - 提供帮助 + 顺带引导
    "求助": [
        {
            "id": "help_01",
            "text": "Hola {用户名}, vi \"{原始内容}\" en {来源KOL}. Parece que necesitas ayuda con eso. Te puedo orientar si quieres.",
            "tone": "supportive",
        },
        {
            "id": "help_02",
            "text": "{用户名}, tu mensaje \"{原始内容}\" en {来源KOL} me llamó la atención. Si tienes ese problema, hay soluciones. ¿Quieres que te explique?",
            "tone": "supportive",
        },
    ],
    
    # 兜底默认
    "default": [
        {
            "id": "default_01",
            "text": "Hola {用户名}, vi tu comentario \"{原始内容}\" en {来源KOL}. ¿Hay algo en lo que pueda ayudarte?",
            "tone": "neutral",
        },
    ],
}
```

### 8.2 评论回复话术模板（MVP版本3套通用）

```python
COMMENT_TEMPLATES = {
    "通用_求购意向": [
        {
            "id": "comment_buy_01",
            "text": "Hola {用户名}! 关于 \"{原始内容}\" - 我们有相关信息可以参考 👉 {链接}",
        },
    ],
    "通用_咨询求助": [
        {
            "id": "comment_help_01",
            "text": "Hola {用户名}! 看到你的问题 \"{原始内容}\" - 这里有详细信息可以看看 👉 {链接}",
        },
    ],
    "通用_吐槽投诉": [
        {
            "id": "comment_complaint_01",
            "text": "Hola {用户名}! 理解你的感受。关于 \"{原始内容}\" - 这里有替代方案可以参考 👉 {链接}",
        },
    ],
}
```

### 8.3 间接意向评论回复

针对没有直接求购但有潜在意向的评论：

| 场景 | 话术风格 |
|------|---------|
| 讨论相关话题 | "Hola {用户名}! 看到你聊到{话题}, 这里有相关信息可以参考 👉 {链接}" |
| 提到品牌/行业 | "正好看到你提到的这个话题, 我们有更多选择 👉 {链接}" |
| 吐槽竞品/行业 | "理解你的感受, 这里有替代方案可以看看 👉 {链接}" |

### 8.4 模板变量

| 变量 | 来源 | 用途 |
|------|------|------|
| `{用户名}` | 触达表.用户名 | 个性化称呼 |
| `{来源KOL}` | 触达表.来源KOL | 建立关联感 |
| `{原始内容}` | 触达表.原始内容（截取前30字） | 证明不是群发 |
| `{链接}` | UTM生成 | 进件链接 |

### 8.5 轮换逻辑

```python
import random

def pick_template(semantic_label, templates_dict):
    """根据语义标签随机选一套话术"""
    # "投诉 | 强烈 | ES" → "投诉_强烈"
    parts = semantic_label.split(" | ")
    key = f"{parts[0]}_{parts[1]}" if len(parts) >= 2 else parts[0]
    
    templates = templates_dict.get(key, templates_dict.get("default", []))
    if not templates:
        return None
    return random.choice(templates)
```

---

## 九、AdsPower集成

### 9.1 架构分层

| 层级 | 负责方 | 职责 |
|------|--------|------|
| 账号层 | AdsPower | 启动/关闭浏览器Profile，管理N个账号 |
| 操作层 | Playwright | 打开主页 → 点击消息 → 输入 → 发送 → 验证 |
| 逻辑层 | 二娃脚本 | 选人 → 选账号 → 选话术 → 记录 → 风控 |

### 9.2 AdsPower API能力

| 能做的 | 不能做的 |
|--------|---------|
| 启动/关闭浏览器实例 | 点击按钮 |
| 返回WebSocket连接地址 | 输入文本 |
| 管理多个Profile | 发送消息 |
| 切换代理IP | 任何页面内操作 |

**方案：AdsPower管浏览器，Playwright管页面操作。**

### 9.3 发消息流程（7步）

```
步骤1: 启动浏览器
  AdsPower API → 启动Profile → 返回WebSocket URL
  Playwright → connect_over_cdp(ws_url) → 获取page

步骤2: 打开用户主页
  page.goto("https://www.facebook.com/wzambrano1")
  等待加载（3-5秒）

步骤3: 找到消息按钮
  page.click('div[aria-label="Message"]')  # 或 "Mensaje"（西班牙语）
  等待Messenger窗口弹出（2-3秒）

步骤4: 输入文本
  page.fill('div[aria-label="Message"]', "Hola, vi tu comentario...")
  # 或用type()模拟逐字输入（更像真人）

步骤5: 发送
  page.keyboard.press("Enter")
  等待发送确认（1-2秒）

步骤6: 验证
  检查消息是否出现在对话框里
  如果出现"无法发送"→ 记录失败原因

步骤7: 记录
  更新触达表：触达状态=触达中，触达时间=now，触达内容=话术
```

### 9.4 评论回复流程

```
步骤1: 启动浏览器（同DM）

步骤2: 打开帖子
  page.goto("https://www.facebook.com/{KOL}/posts/{帖子ID}")

步骤3: 找用户评论
  定位评论区域，搜索用户名或评论内容

步骤4: 点击回复按钮
  在用户评论下找到"Reply"/"Responder"按钮

步骤5: 输入回复文本
  page.fill('textarea[placeholder="Write a reply..."]', 话术)

步骤6: 发送
  page.keyboard.press("Enter")

步骤7: 验证 + 记录
```

### 9.5 风险点和应对

| 风险 | 应对 |
|------|------|
| 陌生人消息进"请求"文件夹 | 无法避免，记录在触达表，48h无响应就跟进 |
| 消息按钮不存在（用户关闭DM） | 检测按钮是否存在，不存在→标记"无可用通道" |
| 发送频率过高被限制 | 每账号每日上限15条，两条之间间隔3-5分钟随机 |
| 页面DOM变化导致选择器失效 | 用aria-label（相对稳定），失败时记录错误不崩溃 |
| 需要登录/验证码 | AdsPower预先手动登录好，保持session |

---

## 十、风控参数

### 10.1 参数配置

```python
# config/settings.py

OUTREACH = {
    # 状态机参数
    "response_window_hours": 48,
    "followup_interval_hours": 72,
    "max_attempts": 3,
    "cooldown_hours": 24,
    
    # 账号风控
    "daily_limit_per_account": 15,
    "min_interval_seconds": 180,  # 3分钟
    "max_interval_seconds": 300,  # 5分钟
    
    # 链接配置
    "landing_page_url": "https://xxx.com/apply",
    "send_link_after_response": True,
}

ACCOUNTS = [
    {"profile_id": "xxx", "daily_limit": 15},
    {"profile_id": "yyy", "daily_limit": 15},
    # 后续加账号只需加一行
]
```

### 10.2 账号轮换逻辑

```python
def pick_account(accounts):
    """选一个可用账号"""
    for account in accounts:
        if account["daily_count"] < account["daily_limit"]:
            return account
    return None  # 所有账号都满了，今天停止触达

def execute_outreach(user, account, template):
    """执行触达"""
    # 1. 启动浏览器
    ws_url = adspower_api.start_browser(account["profile_id"])
    page = playwright.connect_over_cdp(ws_url)
    
    # 2-7. 执行发消息流程
    # ...
    
    # 8. 完成后更新计数
    account["daily_count"] += 1
    
    # 9. 随机间隔
    interval = random.randint(OUTREACH["min_interval_seconds"], OUTREACH["max_interval_seconds"])
    time.sleep(interval)
```

### 10.3 冷却控制

每次触达后，设置冷却截止时间：

```python
cooldown_until = datetime.now() + timedelta(hours=OUTREACH["cooldown_hours"])
# 写入触达表.冷却截止字段
```

下次选人时，跳过冷却截止 > 当前时间的用户。

---

## 十一、Skill设计

### 11.1 Skill清单

| Skill | 触发命令 | 功能 |
|-------|---------|------|
| outreach-handoff | `#outreach-handoff` | 移交已验证用户到触达表 |
| outreach-run | `#outreach-run` | 执行触达（选人 → 发消息） |
| outreach-followup | `#outreach-followup` | 检查响应 + 二次触达 |

### 11.2 触发方式（MVP阶段）

| Skill | 触发方式 | 时机 |
|-------|---------|------|
| outreach-handoff | 手动 | 大娃验证后执行一次 |
| outreach-run | 手动 | 建议UTC-5 9-12点（哥伦比亚上午） |
| outreach-followup | 手动 | 触达后48小时执行 |

**改成定时的条件：**
- 触达流程跑通3轮以上
- 每日待触达 > 30人
- AdsPower登录状态能保持24h+

### 11.3 并发控制

用文件锁防止重复触发：

```python
from pathlib import Path

def acquire_lock(skill_name):
    lock_file = Path(f"data/.{skill_name}.lock")
    if lock_file.exists():
        print(f"另一个 {skill_name} 正在执行，跳过")
        return False
    lock_file.touch()
    return True

def release_lock(skill_name):
    lock_file = Path(f"data/.{skill_name}.lock")
    lock_file.unlink(missing_ok=True)

# 脚本开始
if not acquire_lock("outreach_run"):
    sys.exit(0)

try:
    main()
finally:
    release_lock("outreach_run")
```

---

## 十二、与大娃的接口约定

### 12.1 移交条件

```
采集用户表.状态 = "已验证"
```

### 12.2 移交字段

| 字段 | 说明 |
|------|------|
| 用户名 | — |
| 用户主页 | — |
| 用户编号 | — |
| 来源KOL | — |
| 语义标签 | — |
| 意图评分 | — |
| 联系方式 | — |
| 原始内容 | — |
| 采集记录ID | — |

### 12.3 移交后处理

```
采集用户表.状态 → "已移交"（大娃不再处理）
```

---

## 十三、度量指标

| 指标 | 公式 | 目标值 |
|------|------|--------|
| 触达率 | 已触达 / 待触达 | > 90% |
| 响应率 | 已响应 / 已触达 | > 15% |
| 转化率 | 已转化 / 已响应 | > 30% |
| 平均触达次数 | 总触达次数 / 已转化 | < 2 |

---

## 十四、后续迭代计划

| 版本 | 新功能 | 触发条件 |
|------|--------|---------|
| v1.1 | 定时触发（cron） | 触达流程跑通3轮 + 日待触达 > 30人 |
| v1.2 | WhatsApp通道 | FB DM触达率稳定50% + 有20+ WhatsApp用户被跳过 |
| v1.3 | 时间衰减优先级 | 待触达队列 > 50人 |
| v1.4 | 评论回复话术细分 | 评论回复转化数据积累50+ |
| v1.5 | 间接意向自动识别 | 有间接意向评论数据积累 |

---

## 十五、提示词设计

> 📝 **提示词待补充**
>
> 此章节用于记录二娃使用的LLM提示词。
>
> 计划包含：
> - 响应分类提示词（RESPONSE_CLASSIFIER_PROMPT，四分类）
> - 异议处理辅助提示词（HESITANT用户场景）
> - 用户意图重分类提示词（模糊场景LLM兜底）
> - 评论内容相关性判断提示词
>
> *Diwei 后续补充具体内容*

**文档完成。**