# 大娃Agent设计文档

**版本：v2.0**
**创建日期：2026-04-28**
**状态：生产运行中**

---

## 一、身份定位

| 维度 | 内容 |
|------|------|
| **代号** | Intelligence Agent |
| **昵称** | 大娃 |
| **职责** | 情报采集与目标筛选，发现目标 → 清洗目标 → 上报飞书 |
| **上游** | Facebook KOL 帖子评论 |
| **下游** | 二娃（触达 Agent，通过移交脚本对接） |

---

## 二、智能体架构

### 2.1 完整文件结构

```
intelligence_agent_v1.0/
│
├── config/
│   ├── settings.py              # 全局参数配置
│   │   ├── 飞书配置（app_id/secret/app_token/表ID）
│   │   ├── Cookie文件路径
│   │   └── AdsPower配置（API地址/Profile ID）
│   │
│   ├── kol_sources.json         # KOL数据源配置（20个KOL）
│   ├── feishu_field_map.json    # 用户表字段映射
│   └── kol_fields_map.json      # KOL表字段映射
│
├── scripts/
│   ├── hunter_run.py            # Skill: hunter-collect（主流程）
│   │   ├── 读飞书KOL表（分类：有帖子/无帖子）
│   │   ├── 启动AdsPower浏览器
│   │   ├── 有帖子KOL → 逐帖采集评论
│   │   ├── 无帖子KOL → 访问主页获取帖子（最多5条）→ 回写飞书
│   │   ├── 采集评论（DOM解析用户名/主页/内容）
│   │   ├── 跨帖子去重（用户名+用户ID）
│   │   ├── 上传飞书用户表（状态=待清洗）
│   │   └── 本地备份JSON
│   │
│   ├── clean_collected_data.py  # Skill: hunter-clean（基础清洗）
│   │   ├── 垃圾账号过滤（GARBAGE_KEYWORDS）
│   │   ├── 意图分类（4词库：HIGH/MEDIUM/COMPLAINT/LOW）
│   │   ├── 情感分析（正面/中性/负面）
│   │   ├── 语义识别（行为意图|情绪强度|语言）
│   │   ├── 联系方式提取（WhatsApp/Telegram/电话/邮箱）
│   │   └── 评分计算（加权/联系方式+0.3）
│   │
│   ├── deep_clean.py            # Skill: hunter-clean（深度清洗v2.0）
│   │   ├── KOL表5项检查（链接池/reel/主页/粉丝/完整性）
│   │   ├── 用户表13项检查（用户名/内容/去重/意图/语义/编号/来源/URL/主页）
│   │   ├── 所有记录走完整分类（数据完整性保证）
│   │   ├── 评分驱动状态（<0.2过滤，高中→待分配，低→待观察）
│   │   └── 单选字段写入失败时回退只写文本字段
│   │
│   ├── verify_users.py          # Skill: hunter-verify（僵尸检测）
│   │   ├── 跨KOL水军检测（3+KOL下评论→标记）
│   │   ├── AdsPower访问用户主页
│   │   ├── 5维活跃度评分（可访问/头像/好友/帖子/活跃时间）
│   │   ├── 评分判定（≥70已验证/40-69待人工/<40僵尸）
│   │   └── 支持--dry-run和--limit参数
│   │
│   ├── clean_kol_v2.py          # Skill: hunter-kol-manage（KOL清洗）
│   │   ├── FB ID自动从主页提取修正
│   │   ├── reel链接自动清理
│   │   ├── 主页与名称不匹配→飞书标注⚠️
│   │   ├── 状态自动判断（启用/待补充/暂停/失效）
│   │   └── 粉丝数/类型/完整性检查
│   │
│   ├── audit_kol_data.py        # Skill: hunter-kol-manage（KOL审计）
│   │   └── 名称/类型/FB ID/粉丝/主页一致性检查
│   │
│   ├── sync_status.py           # 状态同步（KOL表+用户表）
│   ├── add_new_kols.py          # KOL新增
│   ├── mark_kol_priority.py     # 贷款优先级标记
│   ├── count_kols.py            # KOL统计
│   ├── resolve_kol_urls.py      # KOL URL解析（数字ID→用户名）
│   ├── fb_login.py              # Skill: hunter-login（Cookie登录）
│   ├── fb_cookie_refresh.py     # Cookie刷新
│   └── delete_empty_records.py  # 清理空记录
│
├── tools/
│   ├── fb_user_id_extractor.py  # FB用户ID提取
│   │   ├── extract_facebook_id(url) → (id, type)
│   │   ├── normalize_user_profile(url) → 标准化URL
│   │   └── get_user_display_info(url, name) → 完整用户信息
│   │
│   ├── feishu_cli.py            # 飞书CLI工具
│   └── feishu_updater.py        # 飞书批量更新
│
├── skills/                      # Kiro Skill定义
│   ├── hunter-collect/
│   │   └── SKILL.md             # 触发: #hunter-collect → hunter_run.py
│   ├── hunter-clean/
│   │   └── SKILL.md             # 触发: #hunter-clean → deep_clean.py
│   ├── hunter-login/
│   │   └── SKILL.md             # 触发: #hunter-login → fb_login.py
│   └── hunter-kol-manage/
│       └── SKILL.md             # 触发: #hunter-kol-manage → clean_kol_v2.py
│
├── identity/
│   ├── SOUL.md                  # Agent人格（稳定，不常改）
│   └── PROFILE.md               # Agent能力档案（稳定，不常改）
│
├── knowledge/
│   ├── MEMORY.md                # 任务经验积累
│   ├── reports/                 # 历史报告（5份）
│   └── docs/                    # 使用指南（2份）
│
├── docs/
│   └── 大娃Agent设计文档-v2.0.md  # 完整设计文档
│
├── data/
│   ├── fb_cookies_latest.json   # Facebook Cookie
│   └── dedup_cache.json         # 去重缓存
│
├── .env                         # 凭证（飞书API/AdsPower）
└── .env.example                 # 凭证模板
```

### 2.2 模块职责

| 层 | 包含文件 | 定位 | 是否依赖LLM |
|----|---------|------|------------|
| **config/** | settings.py / kol_sources.json / feishu_field_map.json / kol_fields_map.json | 配置层：飞书凭证、KOL数据源、字段映射集中管理 | 否 |
| **scripts/** | hunter_run.py / clean_collected_data.py / deep_clean.py / verify_users.py / clean_kol_v2.py / audit_kol_data.py / sync_status.py / add_new_kols.py / mark_kol_priority.py / count_kols.py / resolve_kol_urls.py / fb_login.py / fb_cookie_refresh.py / delete_empty_records.py | 业务层：采集→清洗→验证→管理的完整管线，每个核心Skill对应一个入口脚本 | clean_collected_data.py（关键词分类，无LLM） |
| **tools/** | fb_user_id_extractor.py / feishu_cli.py / feishu_updater.py | 能力层：封装FB用户ID提取、飞书API读写 | 否 |
| **skills/** | hunter-collect/ / hunter-clean/ / hunter-login/ / hunter-kol-manage/ | 入口层：Skill描述，定义触发方式和参数 | 否 |
| **identity/** | SOUL.md / PROFILE.md | 身份层：Agent人格与能力档案（大娃特有，二娃无此层） | 否 |
| **knowledge/** | MEMORY.md / reports/ / docs/ | 记忆层：历史报告与任务经验积累（大娃特有） | 否 |

### 2.3 调用链路

```
                    ┌──────────────────────────────────┐
                    │      飞书多维表格（3张表）        │
                    │  KOL采集表 / 采集用户表 / 触达表  │
                    └──────────┬───────────┬───────────┘
                               │           │
                ┌──────────────┘           └──────────────┐
                │                                          │
           ┌────▼─────┐                           ┌───────▼───────┐
           │  采集     │                           │   KOL管理      │
           │ hunter_run│                           │ clean_kol_v2  │
           │ .py       │                           │ audit_kol_data│
           └────┬─────┘                           └───────┬───────┘
                │                                          │
                ▼                                          │
           ┌────▼─────┐                                    │
           │  清洗     │                                    │
           │ deep_clean│                                    │
           │ .py       │                                    │
           └────┬─────┘                                    │
                │                                           │
                ▼                                           │
           ┌────▼─────┐                                    │
           │  验证     │                                    │
           │ verify_   │                                    │
           │ users.py  │                                    │
           └────┬─────┘                                    │
                │                                           │
                ▼                                           ▼
           ┌────▼─────┐                           ┌───────▼───────┐
           │  移交     │                           │   技能辅助     │
           │ handoff   │                           │ fb_login.py    │
           │ .py(二娃) │                           │ sync_status.py │
           └──────────┘                           └───────────────┘
```

### 2.4 LLM使用范围

与二娃不同，**大娃在MVP阶段完全不依赖LLM**。所有分类和判断均为关键词规则驱动：

| 判断环节 | 方式 | 覆盖精度 |
|---------|------|---------|
| 意图分类 | 4词库：HIGH/MEDIUM/COMPLAINT/LOW | 90%+ |
| 语义识别 | 正则+词典匹配 | 85%+ |
| 情感分析 | 正面/中性/负面词库 | 80%+ |
| 僵尸检测 | 5维活跃度评分规则 | 75%+ |

**后续迭代方向（v2.5+）：**
- 意图分类的模糊场景引入LLM补充（参考二娃关键词→LLM双层架构）
- 语义识别中相似意图聚类改用embedding
- 联系方式提取中非结构化文本用LLM兜底

---

## 三、数据边界

### 3.1 领地划分

| 领地 | Agent | 操作权限 |
|------|--------|---------|
| KOL 采集表 | 大娃 | 读写 |
| 采集用户表 | 大娃 | 读写 |
| 触达用户表 | 二娃 | 大娃不碰 |

```
大娃领地（读写）                二娃领地（大娃不碰）
┌────────────┐                ┌────────────┐
│ KOL 采集表  │                │ 触达用户表  │
│ 采集用户表  │                │            │
└────────────┘                └────────────┘
        ↓ 状态=已验证                ↑
    移交脚本（handoff.py）──────────┘
        ↓
  采集用户表.状态 → "已移交"
```

### 3.2 飞书表格

| 表 | ID | 用途 | 操作权限 |
|----|-----|------|---------|
| KOL 采集表 | tbljhYo4MomkOuu6 | KOL 资料 + 帖子链接池 | 读写 |
| 采集用户表 | tbljLvRkD0jGhboB | 目标用户 + 评论 | 读写 |

### 3.3 KOL 采集表字段

| 字段 | 类型 | 说明 |
|------|------|------|
| KOL 名称 | 文本 | — |
| KOL 类型 | 单选 | 银行/借贷/金融科技/消费/电信/娱乐 |
| Facebook ID | 文本 | 自动从主页链接提取 |
| 主页链接 | 超链接 | — |
| 粉丝数量 | 文本 | — |
| 帖子链接池 | 文本 | 每行一个 URL，自动过滤 reel |
| 状态 | 单选 | 启用/待补充/暂停/失效 |
| 文本 | 文本 | ⚠️ 人工确认标注 |
| 备注 | 文本 | — |
| 采集权重 | 数字 | — |
| 目标数量 | 数字 | 每个 KOL 采集目标数 |

### 3.4 采集用户表字段（24个）

| 字段 | 类型 | 写入阶段 | 说明 |
|------|------|---------|------|
| 用户名 | 文本 | 采集 | — |
| 用户编号 | 文本 | 采集 | FB 数字 ID 或用户名 |
| 用户主页 | 超链接 | 采集 | — |
| 内容 | 文本 | 采集 | 评论原文 |
| 互动类型 | 单选 | 采集 | 评论 |
| 来源 KOL_文本 | 文本 | 采集 | 统一为 KOL 表标准名称 |
| 来源KOL | 超链接 | 采集 | — |
| 帖子 URL_文本 | 文本 | 采集 | — |
| 帖子URL | 超链接 | 采集 | — |
| 平台 | 文本 | 采集 | Facebook |
| 发现时间 | 日期 | 采集 | — |
| 状态 | 单选 | 清洗 | 待清洗/待分配/待观察/已过滤/已验证/已移交 |
| 意图 | 单选 | 清洗 | 高/中/低 |
| 语义 | 文本 | 清洗 | 行为意图 \| 情绪强度 \| 语言 |
| 情感 | 文本 | 清洗 | 正面/中性/负面 |
| 评分 | 数字 | 清洗 | 0-1.0 |
| 优先级 | 单选 | 清洗 | 高/中/低 |
| 联系方式 | 文本 | 清洗 | WhatsApp/Telegram/Tel/Email |
| 信号 | 文本 | 采集 | — |
| 消费信号 | 文本 | 采集 | — |
| 备注 | 文本 | 清洗 | 过滤原因/验证结果 |
| 目标ID | 文本 | 采集 | — |
| 原始截图 | 超链接 | 采集 | — |

---

## 四、状态机

### 4.1 KOL 状态

| 状态 | 条件 | 自动/手动 |
|------|------|----------|
| 启用 | 粉丝 ≥ 5000 + 帖子链接池有有效链接 | 自动 |
| 待补充 | 粉丝 ≥ 5000 + 帖子链接池为空 | 自动 |
| 暂停 | 粉丝 < 5000 / 主页无帖子 | 自动 |
| 失效 | 主页链接为空 / FB ID 无效 | 自动 |

### 4.2 采集用户状态流转

```
待清洗 ──→ 待分配（高/中意向 + 评分≥0.2）
  │          │
  │          ├──→ 已验证（verify_users 通过）──→ 已移交（handoff 到二娃）
  │          │
  │          └──→ 待人工（verify_users 可疑）──→ 已验证 / 已过滤
  │
  ├──→ 待观察（低意向 + 评分≥0.2）
  │
  └──→ 已过滤（垃圾/重复/评分<0.2/僵尸）
```

### 4.3 状态判定规则

| 条件 | 状态 |
|------|------|
| 用户名无效（未知/groups/profile.php） | 已过滤 |
| 内容 < 15 字符 | 已过滤 |
| 重复用户 | 已过滤 |
| 评分 < 0.2 | 已过滤 |
| 高/中意向 + 评分 ≥ 0.2 | 待分配 |
| 低意向 + 评分 ≥ 0.2 | 待观察 |
| 无意向 + 评分 ≥ 0.2（有联系方式加权） | 待观察 |
| verify_users 评分 ≥ 70 | 已验证 |
| verify_users 评分 40-69 | 待人工 |
| verify_users 评分 < 40 | 已过滤（僵尸） |
| 受保护状态（已分配/已触达/已转化/已关闭） | 跳过不处理 |

---

## 五、核心流程

### 5.1 数据流总览

```
飞书 KOL 表
    ↓ hunter_run.py 读取
    ├── 有帖子链接池 → 直接采集评论
    └── 无帖子链接池 → AdsPower 自动获取帖子 → 回写飞书 → 采集评论
    ↓ 跨帖子去重
飞书采集用户表（状态=待清洗）
    ↓ deep_clean.py（18项检查 + 语义识别）
意图分类 + 评分 + 语义标签 + 联系方式提取 → 待分配 / 待观察 / 已过滤
    ↓ verify_users.py（僵尸检测）
跨KOL水军检测 + 主页活跃度验证 → 已验证 / 待人工 / 僵尸
    ↓ handoff.py（移交给二娃）
已验证 → 触达用户表 → 采集用户表状态改为"已移交"
```

### 5.2 采集流程（hunter_run.py）

```
1. 从飞书 KOL 表读取所有 KOL
2. 分类：有帖子链接 vs 无帖子链接
3. 启动 AdsPower 指纹浏览器
4. 有帖子链接的 KOL → 逐帖采集评论
5. 无帖子链接的 KOL → 访问主页自动获取帖子 → 回写飞书 → 采集评论
6. 跨帖子去重（用户名 + 用户ID）
7. 上传飞书采集用户表（状态=待清洗）
8. 本地备份 JSON
```

**步骤 5 帖子自动获取参数：**

| 参数 | 值 | 说明 |
|------|-----|------|
| 每个 KOL 最多获取帖子数 | 5 | 取最近 5 条，避免过多请求 |
| 每个 KOL 最多采集帖子数 | 2 | 获取 5 条但只采集前 2 条评论 |
| 过滤条件 | 非 reel | /reel/ 链接自动跳过 |
| 优先级 | posts > videos > permalink > photo | photo 作为兜底 |
| 排序 | 页面自然顺序（最近发布优先） | Facebook 默认按时间倒序 |
| 获取失败处理 | 主页无可采集帖子 → KOL 状态改为"暂停" | 回写飞书 |
| 回写策略 | 全量写入帖子链接池 | 获取到的有效链接全部回写，KOL 状态改为"启用" |

### 5.3 清洗流程（deep_clean.py v2.0）

18 项检查，分三层：

**KOL 表检查（5项）：**

| # | 检查项 | 行为 |
|---|--------|------|
| 1 | 帖子链接池不能为空（启用KOL） | 标记 |
| 2 | 帖子链接池不能有 reel | 自动清理 |
| 3 | 启用 KOL 必须有主页链接 | 标记 |
| 4 | 粉丝数不能为 0 | 标记 |
| 5 | 名称/类型/FB ID 完整 | 标记 |

**用户表检查（13项）：**

| # | 检查项 | 行为 |
|---|--------|------|
| 6 | 用户名不能是无效值 | 过滤 |
| 7 | 用户名时间残留清理 | 自动修复 |
| 8 | 内容 ≥ 15 字符 | 过滤 |
| 9 | 来源 KOL 不能为空 | 标记 |
| 10 | 帖子 URL 不能为空 | 标记 |
| 11 | 同一用户去重 | 过滤 |
| 12 | 意图分类 + 评分 + 语义识别 | 写入字段 |
| 13 | 帖子 URL 可追溯到 KOL 链接池 | 标记 |
| 14 | 来源 KOL 存在于 KOL 表 | 标记 |
| 15 | 用户编号完整性 | 自动修复/标记 |
| 16 | 来源 KOL 格式统一 | 自动修正 |
| 17 | 帖子 URL 格式校验 | 标记 |
| 18 | 用户主页链接完整性 | 标记 |

**所有记录都走完整分类流程，确保数据完整性（意图/评分/情感/语义字段不为空）。**

---

## 六、意图分类系统

### 6.1 分类逻辑

```
评论内容
  ↓ 关键词匹配（4 个词库）
  ├── HIGH_INTENT    ×3 权重   借贷需求、联系意向
  ├── MEDIUM_INTENT  ×2 权重   咨询信息、价格询问
  ├── COMPLAINT      ×2 权重   投诉类（有真实消费行为）
  └── LOW_INTENT     ×1 权重   浏览/泛咨询
  ↓
  评分 = min(总加权 / 10, 1.0)
  ↓ 有联系方式 → 评分 +0.3
  ↓
  评分 < 0.2 → 已过滤
  high/medium → 待分配
  low → 待观察
```

### 6.2 关键词库

**HIGH_INTENT（高意向）：**
- 借贷需求：necesito dinero, urgente, necesito prestamo, quiero prestamo, solicito
- 联系意向：whatsapp, wa.me, t.me, mi numero, contactame, escribeme, llamame
- 金额表达：necesito 100/200/500/1000/5000/10000, mil pesos
- 时间紧迫：para hoy, ya mismo, inmediato

**MEDIUM_INTENT（中意向）：**
- 咨询：informacion, como funciona, donde puedo, requisitos, que necesito
- 价格：cuanto cuesta, precio, cuanto es, costo, valor, tasa, cuota
- 兴趣：interesado, me gustaria, quisiera saber, consulta

**COMPLAINT_INTENT（投诉类 → 算中意向）：**
- 服务投诉：no sirve, pesimo, terrible, horrible, estafa, fraude
- 产品问题：dañada, dañado, mal estado, mal sellado, no responden
- 改善诉求：arreglen, actualizar, mejorar, queja, reclamo

**LOW_INTENT（低意向）：**
- 泛咨询：problema, error, falla, ayuda, busco
- 浏览：opinion, experiencia, como les fue, vale la pena

---

## 七、语义识别系统

### 7.1 三维通用标签（不绑定行业）

输出格式：`行为意图 | 情绪强度 | 语言`

### 7.2 行为意图（7类）

| 标签 | 匹配逻辑 | 示例 |
|------|---------|------|
| 求购 | quiero comprar, necesito, solicitar, adquirir | "Necesito un prestamo urgente" |
| 咨询 | como funciona, precio, requisitos, cuanto cuesta | "Como puedo solicitar?" |
| 投诉 | no sirve, pesimo, estafa, dañada, no responden | "La app no sirve para nada" |
| 求助 | ayuda, no puedo, no funciona, error, bloqueado | "No puedo transferir dinero" |
| 推荐 | excelente, recomiendo, perfecto, genial, lo mejor | "Excelente servicio" |
| 分享 | mi experiencia, confiable, vale la pena, yo compre | "Confiable pero se demora" |
| 闲聊 | 以上都不匹配 | "Ya lo dijo Meryl Streep..." |

### 7.3 情绪强度（3级）

| 级别 | 判定条件 |
|------|---------|
| 强烈 | 大写占比>40% / 感叹号≥3 / 重复字符≥3 / 极端词≥2 |
| 中等 | 有感叹号或情绪词 |
| 平淡 | 短文本(<30字) + 无情绪信号 |

### 7.4 语言检测

基于高频词匹配：ES（西班牙语）/ PT（葡萄牙语）/ EN（英语）

### 7.5 与意图分类的关系

| | 意图分类 | 语义识别 |
|---|---|---|
| 决定什么 | 状态（待分配/已过滤） | 描述标签 |
| 绑定行业 | 是（借贷关键词） | 否（通用） |
| 输出字段 | 意图 + 评分 + 状态 | 语义 |
| 换行业要改 | 要改关键词库 | 不用改 |

---

## 八、联系方式提取

### 8.1 支持格式

| 类型 | 匹配规则 | 示例 |
|------|---------|------|
| WhatsApp | wa.me/数字, whatsapp+数字 | wa.me/573757647212 |
| Telegram | t.me/用户名, @用户名 | t.me/Dln75, @Sofia_Elena12 |
| 电话 | 哥伦比亚手机号(3xx开头10位), +57前缀 | +57 3554672296 |
| 邮箱 | 标准邮箱格式 | carlos@gmail.com |

### 8.2 评分加权

有联系方式的用户评分 +0.3（上限 1.0）。

### 8.3 输出

写入飞书"联系方式"字段，格式：`WhatsApp: xxx | Telegram: xxx | Tel: xxx | Email: xxx`

---

## 九、用户验证（僵尸检测）

### 9.1 两个检测维度

**跨 KOL 水军检测（不需要浏览器）：**
- 同一用户在 3+ 个不同 KOL 的帖子下评论 → 标记水军嫌疑
- 水军用户评分 -20

**主页活跃度验证（需要 AdsPower）：**

| 维度 | 满分 | 判定 |
|------|------|------|
| 可访问性 | 20 | 主页能打开 |
| 头像 | 10 | 非默认头像 |
| 好友数 | 25 | ≥500=25, ≥100=20, ≥20=10 |
| 帖子数 | 25 | ≥10=25, ≥5=20, ≥2=10 |
| 活跃度 | 20 | 今天=20, 本月=15, 3月内=10 |

### 9.2 评分判定

| 评分 | 判定 | 状态 |
|------|------|------|
| ≥ 70 | 真实用户 | 已验证 |
| 40-69 | 可疑 | 待人工 |
| < 40 | 僵尸 | 已过滤 |

### 9.3 人工复核流程

**适用对象**：verify_users 评分 40-69 的"待人工"用户

**当前阶段（v2.0）**：

| 步骤 | 操作人 | 动作 |
|------|--------|------|
| 1 | 大娃脚本 | verify_users.py 将评分 40-69 的用户标记为"待人工"，备注写入具体原因（如"0好友 \| 无头像"） |
| 2 | 运营人员 | 在飞书采集用户表筛选"状态=待人工"，逐条查看备注 |
| 3 | 运营人员 | 打开用户 FB 主页，人工判断是否真实用户 |
| 4 | 运营人员 | 手动改状态：真实→"已验证"，虚假→"已过滤" |

**复核标准**：
- 有真实头像 + 有近期动态 → 已验证
- 主页全空但评论内容有明确需求 → 已验证（内容优先）
- 主页不存在 / 明显机器人特征 → 已过滤

**后续优化（v2.3+）**：当待人工用户积累 20+ 条时，考虑在飞书表加审批流，自动通知运营人员复核。

### 9.4 性能边界

| 用户数 | 预估耗时 | 说明 |
|--------|---------|------|
| 20 | 2-3 分钟 | 当前阶段，可接受 |
| 50 | 5-8 分钟 | 可接受 |
| 100+ | 10-15 分钟 | 需要优化 |

**当前限制**：单 Profile 串行访问，每个用户主页加载 5-10 秒。

**优化方向（100+ 用户时启用）**：
- 超时跳过：单个主页加载超过 15 秒自动跳过，标记"待人工"
- 批量分页：`--limit N` 参数分批跑，避免单次运行过长
- 并发 Profile：多个 AdsPower Profile 同时验证（需要多账号支持）

---

## 十、AdsPower 集成

### 10.1 架构分层

| 层级 | 负责方 | 职责 |
|------|--------|------|
| 账号层 | AdsPower | 启动/关闭浏览器 Profile |
| 操作层 | Playwright | 页面操作（滚动/点击/提取） |
| 逻辑层 | 大娃脚本 | 采集流程/清洗逻辑/验证逻辑 |

### 10.2 浏览器方案

| 方案 | 用途 | 说明 |
|------|------|------|
| AdsPower 指纹浏览器 | 主力采集 | 自动获取帖子 + 采集评论，FB 不检测 |
| Playwright + Cookie | 备用 | AdsPower 不可用时 fallback |

### 10.3 配置

```
.env 中：
ADSPOWER_API=http://localhost:50325
ADSPOWER_PROFILE_ID=xxx
```

---

## 十一、KOL 管理

### 11.1 清洗策略 v2.0

| 检查项 | 行为 |
|--------|------|
| FB ID 与主页链接不匹配 | 自动从主页提取修正 |
| 帖子链接池有 reel | 自动清理 |
| 主页与名称不匹配 | 飞书"文本"字段标注 ⚠️ 人工确认 |
| 粉丝数为 0 | 状态改为暂停 |
| 类型不在有效范围 | 标记 |

### 11.2 贷款优先级

| 优先级 | KOL 类型 |
|--------|---------|
| 🔴 高 | 银行/借贷/金融科技 |
| 🟡 中 | 消费/零售 |
| 🟢 低 | 电信/娱乐 |

### 11.3 帖子格式兼容性

| 格式 | 评论采集 | 说明 |
|------|---------|------|
| /posts/ | ✅ 最好 | 20+ 条评论 |
| /videos/ | ✅ 可以 | 2-4 条评论 |
| /photo/ | ⚠️ 部分 | 取决于帖子 |
| /reel/ | ❌ 跳过 | DOM 不兼容 |

---

## 十二、Skill 设计

### 12.1 Skill 清单

| Skill | 触发命令 | 功能 | 对应脚本 |
|-------|---------|------|---------|
| hunter-collect | `#hunter-collect` | AdsPower 采集 + 飞书上传 | `scripts/hunter_run.py` |
| hunter-clean | `#hunter-clean` | NLP 清洗 + 语义识别 | `scripts/deep_clean.py`（主）/ `scripts/clean_collected_data.py`（基础） |
| hunter-login | `#hunter-login` | Cookie 更新 | `scripts/fb_login.py` |
| hunter-kol-manage | `#hunter-kol-manage` | KOL 管理（清洗/新增/标记/审计） | `scripts/clean_kol_v2.py`（主）/ `scripts/audit_kol_data.py` / `scripts/add_new_kols.py` |

### 12.2 触发方式

全部手动触发（`#命令`），暂无定时任务。

---

## 十三、与二娃的接口约定

### 13.1 移交条件

```
采集用户表.状态 = "已验证"
```

### 13.2 移交字段（9个）

| 字段 | 来源字段 | 说明 |
|------|---------|------|
| 用户名 | 用户名 | — |
| 用户主页 | 用户主页 | — |
| 用户编号 | 用户编号 | — |
| 来源KOL | 来源 KOL_文本 | 统一为标准名称 |
| 语义标签 | 语义 | 行为意图 \| 情绪强度 \| 语言 |
| 意图评分 | 评分 | 0-1.0 |
| 联系方式 | 联系方式 | WhatsApp/Tel/Telegram/Email |
| 原始内容 | 内容 | 评论原文 |
| 采集记录ID | record_id | 飞书记录 ID，用于溯源 |

### 13.3 移交后处理

```
采集用户表.状态 → "已移交"（大娃不再处理）
```

### 13.4 边界规则

- 大娃**永远不碰**触达用户表
- 二娃**永远不碰**采集用户表和 KOL 表
- 移交脚本（handoff.py）是唯一的桥梁

---

## 十四、风控

### 14.1 三问

1. 这个操作的风险是什么？
2. 这个操作如果不做，会影响什么关键指标？做了但失败，最大损失是多少？
3. 如果出问题怎么办？

### 14.2 四不

- 不把鸡蛋放在一个篮子
- 不在风险区恋战
- 不用主号冒险
- 不做无预案的操作

---

## 十五、度量指标

| 指标 | 公式 | 当前值 |
|------|------|--------|
| KOL 总数 | — | 38 |
| 启用 KOL | 状态=启用 | 12 |
| 采集用户总数 | — | 104 |
| 待分配 | 状态=待分配 | 31 |
| 待观察 | 状态=待观察 | 5 |
| 已过滤 | 状态=已过滤 | 68 |
| 有联系方式 | 联系方式非空 | 2 |
| 语义覆盖率 | 有语义/总数 | 104/104 (100%) |

### 15.1 语义分布

| 行为 | 数量 |
|------|------|
| 闲聊 | 57 |
| 求助 | 16 |
| 投诉 | 12 |
| 咨询 | 10 |
| 求购 | 7 |
| 分享 | 1 |
| 推荐 | 1 |

---

## 十六、后续迭代计划

| 版本 | 新功能 | 触发条件 |
|------|--------|---------|
| v2.1 | reel 评论 DOM 适配 | 有 reel 帖子采集需求 |
| v2.2 | photo 评论采集优化 | photo 帖子 0 评论问题解决 |
| v2.3 | 僵尸检测正式运行 | AdsPower 可用 + 待分配用户 > 20 |
| v2.4 | 意图分类准确率优化 | 触达闭环数据积累 50+ |
| v2.5 | 定时采集（cron） | 日均新增 KOL > 5 |

---

## 十七、提示词设计

> 📝 **提示词待补充**
>
> 此章节用于记录大娃各Skill和流程中使用的LLM提示词。
>
> 计划包含：
> - 意图分类提示词
> - 语义识别提示词
> - 僵尸检测复核提示词（人工复核辅助）
> - 清洗流程中的LLM兜底提示词
> - KOL清洗辅助判断提示词
>
> *Diwei 后续补充具体内容*

---

**文档完成。**
