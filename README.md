# Multi-Agents-Ecommerce-System

> 基于**LangGraph**的多智能体电商导购助手

[![Python](https://img.shields.io/badge/Python-3.11%2B-blue?logo=python)](python/)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

## 这个项目是什么？

### 用一句话解释

> 用 AI Agent 技术，让电商平台的**推荐 + 文案 + 库存**三个系统协同工作，像一个聪明的"AI 运营团队"一起为每位用户生成个性化推荐结果。

### 它解决了什么问题？

传统电商推荐系统存在三大痛点：

| 痛点 | 传统做法 | 本项目做法 |
|------|---------|---------|
| 推荐结果和库存脱节 | 推荐了缺货商品 | **库存 Agent** 实时校验，缺货自动剔除 |
| 营销文案千篇一律 | 所有人看同一段广告语 | **文案 Agent** 根据用户画像生成个性化文案 |
| 各系统各自为战 | 推荐、文案、库存三套系统互不感知 | **Supervisor** 统一编排，结果实时互相影响 |

### 技术关键词（面试常考）

`Multi-Agent` · `Supervisor模式` · `LangGraph` · `asyncio并行` · `Redis Feature Store` · `A/B Testing` · `Thompson Sampling` · `RAG` · `ReAct` · `MiniMax LLM`

---

## 系统架构

### 前端对话界面

```
                    用户在对话框输入
                    "推荐一款200元以下的面膜"
                          │
                          ▼
┌──────────────────────────────────────────────┐
│  React 前端（localhost:5173）                  │
│  fetch POST /api/v1/recommend                │
│  渲染商品卡片 + 营销文案                       │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│  FastAPI 后端（localhost:8000）                │
│                                               │
│  SupervisorOrchestrator.recommend()           │
│  ├─ Phase 1: 用户画像 Agent (并行)             │
│  ├─ Phase 1: 商品推荐 Agent (并行)             │
│  ├─ Phase 2: LLM精排 (并行)                   │
│  ├─ Phase 2: 库存决策 Agent (并行)             │
│  ├─ Phase 3: 结果聚合                         │
│  └─ Phase 3: 营销文案 Agent (串行)             │
└──────────────────────────────────────────────┘
```

### Supervisor 编排架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         用户发起推荐请求                           │
│                    {"user_id": "u001", "num_items": 5}           │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Supervisor 协调Agent                           │
│                  (python/orchestrator/supervisor.py)              │
│                                                                   │
│  ════════════════ Phase 1: 并行执行 ═══════════════════           │
│  ┌──────────────────────┐    ┌──────────────────────┐            │
│  │   用户画像 Agent      │    │   商品召回 Agent      │            │
│  │  user_profile_agent  │    │  product_rec_agent   │            │
│  │  ──────────────────  │    │  ────────────────── │            │
│  │  Redis → 实时行为特征 │    │  协同过滤+向量检索召回 │            │
│  │  RFM模型 → 用户分群   │    │  返回候选商品列表     │            │
│  └──────────┬───────────┘    └──────────┬──────────┘            │
│             │                           │                         │
│  ════════════════ Phase 2: 并行执行 ═══════════════════           │
│  ┌──────────────────────┐    ┌──────────────────────┐            │
│  │   LLM重排 Agent      │    │   库存决策 Agent      │            │
│  │  (product_rec再次调用)│    │   inventory_agent    │            │
│  │  ──────────────────  │    │  ────────────────── │            │
│  │  用户画像 × 商品属性  │    │  MySQL → 实时库存查询 │            │
│  │  LLM精排，返回TopN   │    │  过滤缺货，输出限购策略│            │
│  └──────────┬───────────┘    └──────────┬──────────┘            │
│             │                           │                         │
│  ════════════════ Phase 3: 串行执行 ═══════════════════           │
│             └──────────────┬────────────┘                         │
│                            ▼                                      │
│             ┌──────────────────────────────┐                      │
│             │      结果聚合器               │                      │
│             │  库存过滤 → 排序合并 → TopN   │                      │
│             └──────────────┬───────────────┘                      │
│                            ▼                                      │
│             ┌──────────────────────────────┐                      │
│             │   营销文案 Agent              │                      │
│             │  marketing_copy_agent        │                      │
│             │  ────────────────────────── │                      │
│             │  5套Prompt模板 × 用户分群    │                      │
│             │  LLM生成 + 广告法合规校验    │                      │
│             └──────────────┬───────────────┘                      │
│                            ▼                                      │
│             ┌──────────────────────────────┐                      │
│             │   A/B 测试引擎               │                      │
│             │  用户ID哈希分桶              │                      │
│             │  Thompson Sampling 动态调优  │                      │
│             └──────────────┬───────────────┘                      │
└──────────────────────────────┬──────────────────────────────────┘
                               ▼
              ┌─────────────────────────────────┐
              │  个性化推荐响应（返回给用户）      │
              │  商品列表 + 个性化文案 + 实验分组 │
              └─────────────────────────────────┘
```

### 为什么用 Supervisor 模式？

```
Supervisor 模式                     Handoffs 模式
──────────────────────              ──────────────────────
   Supervisor（中枢）                 Agent A → Agent B
    ┌────┬────┬────┐                       ↓
    ▼    ▼    ▼    ▼                 Agent B → Agent C
   A    B    C    D                        ↓
    └────┴────┴────┘                 Agent C → ...
    结果聚合 → 响应

✅ 集中控制，流程清晰          ✅ 去中心化，灵活
✅ 并行执行，延迟低            ✅ 适合对话/开放式任务
✅ 异常统一处理                ❌ 状态管理复杂
本项目采用 Supervisor 模式
```

---

## 四大核心 Agent 详解

### Agent 1：用户画像 Agent

**文件**：[`python/agents/user_profile_agent.py`](python/agents/user_profile_agent.py)

**它做什么？**

把用户的历史行为数据（点击、购买、收藏）转化成结构化的"用户画像"，供其他 Agent 使用。

**核心逻辑**：

```python
# Step 1：从 Redis Feature Store 获取实时行为特征
behavior = await feature_store.get_user_features(user_id)
# 返回: {"clicks_1h": 12, "purchases_7d": 3, "categories": ["手机", "耳机"]}

# Step 2：调用 LLM 分析，输出结构化画像
prompt = f"用户行为数据: {behavior}\n请分析用户分群和RFM得分，输出JSON"
profile_json = await llm.invoke(prompt)

# Step 3：返回 UserProfile 对象
return UserProfile(user_id=user_id, segments=["active"], rfm_score=...)
```

**关键技术**：
- **Redis Sorted Set**：`ZADD user:u001:clicks {时间戳} {商品ID}`，支持滑动窗口查询
- **RFM 模型**：Recency（最近购买时间）x Frequency（购买频率）x Monetary（消费金额）
- **用户分群**：新客 / VIP / 价格敏感 / 活跃 / 流失风险，共 5 类

---

### Agent 2：商品推荐 Agent

**文件**：[`python/agents/product_rec_agent.py`](python/agents/product_rec_agent.py)

**它做什么？**

两阶段推荐：先"召回"大量候选商品，再用 LLM 精排出最合适的 TopN。

```
多路召回策略
  ├── 协同过滤（买了A也买了B）
  ├── 向量检索（Milvus，语义相似商品）
  ├── 热度策略（最近7天热卖）
  └── 新品策略（上架30天内）
        │
        ▼（去重合并，候选集）
  LLM 精排
  │ Prompt: "用户是价格敏感型，偏好手机配件，以下10件商品请排序..."
  │ 输出: 按相关性从高到低排列的商品 ID 列表
        │
        ▼
  TopN 商品列表（交给库存 Agent 过滤）
```

---

### Agent 3：营销文案 Agent

**文件**：[`python/agents/marketing_copy_agent.py`](python/agents/marketing_copy_agent.py)

**它做什么？**

根据用户画像自动选择合适的文案风格模板，调用 LLM 生成个性化文案，并做广告法合规校验。

```python
# 5套模板 x 用户分群
TEMPLATES = {
    "new_user":        "首单专属福利，{product}立减{discount}元！",
    "vip":             "尊享会员特权，{product}专属价{price}，品质之选。",
    "price_sensitive": "今日限时抢购！{product}历史最低价，仅剩{stock}件！",
    "active":          "根据您的浏览偏好，为您精选 {product}，好评率{rating}%",
    "churn_risk":      "好久不见！{product}为您专属保留，点击领取优惠券",
}

# 广告法合规校验（过滤违禁词）
BANNED_WORDS = ["最好", "第一", "最便宜", "绝对", "100%"]
```

---

### Agent 4：库存决策 Agent

**文件**：[`python/agents/inventory_agent.py`](python/agents/inventory_agent.py)

**它做什么？**

查询商品实时库存，过滤缺货商品，输出限购策略和补货预警。

```python
# 输入: 推荐商品列表 [P001, P002, P003, ...]
# 查询 MySQL/WMS 实时库存
# 输出:
{
    "available_products": ["P001", "P003"],   # 有货商品
    "inventory_alerts": [                      # 库存预警
        {"product_id": "P001", "stock": 5, "warning": "库存紧张"}
    ],
    "purchase_limits": {                       # 限购策略
        "P001": 2  # 每人最多买2件
    }
}
```

---

## 技术栈

- 框架：[LangGraph](https://github.com/langchain-ai/langgraph) + FastAPI + ReAct
- 并行方式：`asyncio.gather()`


---

## 关键代码展示

### Supervisor 并行编排（Python 核心代码）

**文件**：[`python/orchestrator/supervisor.py`](python/orchestrator/supervisor.py)

```python
class SupervisorOrchestrator:
    """Supervisor 编排器 — 并行分发 + 聚合模式"""

    async def recommend(self, request: RecommendationRequest) -> RecommendationResponse:
        start = time.perf_counter()

        # ① A/B 实验分组
        experiment = self.ab_engine.assign(request.user_id)

        # ② Phase 1：用户画像 + 商品召回 并行执行
        profile_result, rec_result = await asyncio.gather(
            self.user_profile_agent.run(user_id=request.user_id, context=request.context),
            self.product_rec_agent.run(user_profile=None, num_items=request.num_items * 2),
        )

        # ③ Phase 2：LLM重排 + 库存校验 并行执行
        rerank_result, inventory_result = await asyncio.gather(
            self.product_rec_agent.run(user_profile=user_profile, num_items=request.num_items),
            self.inventory_agent.run(products=raw_products),
        )

        # ④ 库存过滤：只保留有货商品
        available_ids = set(getattr(inventory_result, "available_products", []))
        final_products = [p for p in ranked_products if p.product_id in available_ids]

        # ⑤ Phase 3：文案生成（需要前两步结果，串行）
        copy_result = await self.marketing_copy_agent.run(
            user_profile=user_profile, products=final_products,
        )

        total_latency = (time.perf_counter() - start) * 1000
        return RecommendationResponse(
            products=final_products, marketing_copies=copies,
            experiment_group=experiment.get("group", "control"),
            total_latency_ms=total_latency,
        )
```

> **解读**：`asyncio.gather()` 让两个 IO 密集型任务同时跑，总耗时约等于最慢那个 Agent 的耗时，而不是两者相加。

---

### A/B 测试引擎（Thompson Sampling）

**文件**：[`python/services/ab_test.py`](python/services/ab_test.py)

```python
class ABTestEngine:
    """流量分桶 + Thompson Sampling 多臂赌博机"""

    def assign(self, user_id: str) -> dict:
        # 用户ID哈希取模 → 同一用户每次进同一个实验组
        bucket = int(hashlib.md5(user_id.encode()).hexdigest(), 16) % 100
        if bucket < 60:
            return {"group": "control", "strategy": "collaborative_filter"}
        elif bucket < 80:
            return {"group": "treatment_llm", "strategy": "llm_rerank"}
        else:
            return {"group": "treatment_vector", "strategy": "vector_search"}

    def record_click(self, user_id: str, clicked: bool):
        # Thompson Sampling: 点击了就更新 Beta 分布参数
        group = self.assignments.get(user_id, "control")
        if clicked:
            self.alpha[group] += 1
        else:
            self.beta[group] += 1
```

---

### Agent 基类：重试 + 降级

**文件**：[`python/agents/base_agent.py`](python/agents/base_agent.py)

```python
class BaseAgent(ABC):
    """所有 Agent 的基类 — 模板方法模式"""
    MAX_RETRIES = 3
    RETRY_DELAY = 1.0

    async def run(self, **kwargs) -> AgentResult:
        try:
            return await self._retry_execute(**kwargs)
        except Exception as e:
            return self._fallback(**kwargs)  # 降级

    async def _retry_execute(self, **kwargs) -> AgentResult:
        for attempt in range(self.MAX_RETRIES):
            try:
                return await asyncio.wait_for(
                    self._execute(**kwargs), timeout=self.timeout,
                )
            except asyncio.TimeoutError:
                if attempt < self.MAX_RETRIES - 1:
                    await asyncio.sleep(self.RETRY_DELAY * (2 ** attempt))
        raise RuntimeError(f"{self.name} failed after {self.MAX_RETRIES} retries")

    @abstractmethod
    async def _execute(self, **kwargs) -> AgentResult:
        """子类只需实现这个方法"""
```


## 快速开始

### 环境要求

- Python 3.11+
- Node.js 18+（前端）
- LLM API Key（项目默认支持 MiniMax / DeepSeek / OpenAI 兼容接口）

可选（完整功能需要，但不装也能跑通核心流程）：

- Redis（用户特征缓存）
- Milvus（向量检索）
- MySQL（业务数据）

---

### 第一步：安装后端依赖

```bash
cd multi-agent-ecommerce-system/python

# 推荐：创建独立环境
conda create -n agent python=3.13 -y
conda activate agent

# 安装依赖
pip install -r requirements.txt
```

---

### 第二步：配置环境变量

复制模板并编辑：

```bash
cp .env.example .env
```

编辑 `.env` 文件，至少填写以下内容：

```env
# 必填：LLM API Key
ECOM_LLM_API_KEY=你的API密钥

# LLM 服务地址（根据你用的模型选一个）
# MiniMax（默认）
ECOM_LLM_BASE_URL=https://api.minimax.chat/v1
ECOM_LLM_MODEL=MiniMax-M1

# DeepSeek
# ECOM_LLM_BASE_URL=https://api.deepseek.com/v1
# ECOM_LLM_MODEL=deepseek-v4-flash

# OpenAI 兼容接口都可以，只要改 BASE_URL 和 MODEL
```

> 没有 API Key？去 [MiniMax](https://www.minimax.chat/) 或 [DeepSeek](https://platform.deepseek.com/) 注册。

---

### 第三步：配置 Redis（用户行为数据）

Redis 用于存储用户的实时行为数据（浏览、购买等），让用户画像 Agent 能根据真实数据生成画像。

#### 安装 Redis

```bash
# macOS
brew install redis

# 启动（前台运行）
redis-server /opt/homebrew/etc/redis.conf

# 或用 Docker
docker run -d --name redis -p 6379:6379 redis:7-alpine
```

#### 验证

```bash
redis-cli ping
# 返回 PONG 就说明正常
```

#### 项目配置

`.env` 中默认配置即可（无需修改）：

```env
ECOM_REDIS_URL=redis://localhost:6379/0
```

启动服务时看到 `redis.connected` 日志说明连接成功。

#### Redis 中的数据格式

项目使用 Redis Sorted Set 存储用户行为，格式为：

```text
Key:    behavior:{用户ID}:{行为类型}
Value:  JSON 字符串（商品信息）
Score:  时间戳（用于按时间范围查询）
```

示例：

```text
Key:   behavior:user_003:view
Score: 1747540000
Value: {"item_id": "连衣裙", "ts": 1747540000}
```

#### 写入用户行为数据

使用 redis-cli 手动写入用户行为数据：

```bash
# 记录 user_003 浏览了连衣裙、口红、高跟鞋
redis-cli ZADD behavior:user_003:view 1747540000 '{"item_id":"连衣裙","ts":1747540000}'
redis-cli ZADD behavior:user_003:view 1747540000 '{"item_id":"口红","ts":1747540000}'
redis-cli ZADD behavior:user_003:view 1747540000 '{"item_id":"高跟鞋","ts":1747540000}'

# 记录 user_003 买了面膜
redis-cli ZADD behavior:user_003:purchase 1747540000 '{"item_id":"面膜","ts":1747540000}'
```

#### 没有 Redis 时的行为

如果 Redis 没安装或没启动，系统不会崩溃——用户画像 Agent 会使用硬编码的默认数据（默认浏览"手机、耳机、平板"），所有用户都会被当作数码爱好者推荐。

#### 用户画像判定逻辑

LLM 根据行为数据中的 RFM 指标判定用户类型：

| 指标 | 计算方式 | 判定逻辑 |
| ---- | ---- | ---- |
| Recency（最近购买） | 距上次购买的时间 | 越近分数越高 |
| Frequency（购买频率） | 购买次数 / 10 | 少于 2 次 → 新用户 |
| Monetary（消费金额） | 平均客单价 / 1000 | 客单价低 → 价格敏感 |

例如 `user_003` 只买了一次面膜（低频、低客单价），所以被判定为 `new_user` + `price_sensitive`。

---

### 第四步：启动后端服务

```bash
cd multi-agent-ecommerce-system/python
python main.py
```

看到以下输出说明启动成功：

```text
2026-05-18 11:08:22 [info]  redis.connected  url=redis://localhost:6379/0
2026-05-18 11:08:22 [info]  app.startup      model=deepseek-v4-flash
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```

---

### 第五步：启动前端对话界面

另开一个终端：

```bash
cd multi-agent-ecommerce-system/frontend

# 首次需要安装依赖
npm install

# 启动开发服务器
npm run dev
```

看到以下输出说明启动成功：

```text
VITE v5.4.x  ready in xxx ms

➜  Local:   http://localhost:5173/
```

浏览器打开 **http://localhost:5173** ，即可看到对话界面。

#### 前端使用方式

在输入框中选择用户ID和推荐数量，点击推荐按钮获取个性化商品推荐和营销文案。

---

### 第六步：测试后端 API（可选）

如果你想直接测试后端接口，**另开一个终端**：

#### 健康检查

```bash
curl http://localhost:8000/health
```

#### 核心推荐接口

```bash
curl -s -X POST http://localhost:8000/api/v1/recommend -H "Content-Type: application/json" -d '{"user_id": "user_003", "scene": "homepage", "num_items": 5}'
```

#### LangGraph 推荐

```bash
curl -s -X POST http://localhost:8000/api/v1/recommend/graph -H "Content-Type: application/json" -d '{"user_id": "user_002", "scene": "detail_page", "num_items": 3}'
```

```bash
curl -s -X POST http://localhost:8000/api/v1/recommend/graph -H "Content-Type: application/json" -d '{"user_id": "user_002", "scene": "detail_page", "num_items": 3}'
```

#### Swagger 交互式文档

浏览器打开 http://localhost:8000/docs ，可以直接在页面上填参数测试所有接口。

---

### 第七步：运行单元测试

不需要启动服务，也不需要 LLM API Key：

```bash
cd multi-agent-ecommerce-system/python

# 使用 pytest
pytest tests/ -v

# 或直接运行
python tests/test_ab_test.py
```

预期输出：

```text
All A/B test engine tests passed!
```

---

### 自定义商品库

商品数据在 `python/agents/product_rec_agent.py` 的 `MOCK_PRODUCTS` 列表中。直接编辑即可增删改商品。

#### 商品字段说明

```python
Product(
    product_id="P016",         # 唯一标识
    name="YSL小金条口红",       # 商品名
    category="美妆",           # 类目（用于和用户偏好匹配）
    price=320,                 # 价格（用于价格区间匹配）
    brand="YSL",               # 品牌
    seller_id="S13",           # 卖家ID（用于卖家去重）
    stock=800,                 # 库存（0 会被库存 Agent 过滤）
    tags=["口红", "热卖"],      # 标签（用于 LLM 精排参考）
)
```

#### 当前商品类目分布

| 类目 | 数量 | 示例 |
| ---- | ---- | ---- |
| 手机 | 2 | iPhone 16 Pro, 华为 Mate 70 |
| 耳机 | 2 | AirPods Pro 3, Sony WH-1000XM6 |
| 平板 | 2 | iPad Air M3, 小米平板7 Pro |
| 配件 | 3 | Anker充电器, 绿联充电头, 罗技鼠标 |
| 美妆 | 5 | YSL口红, 兰蔻精华, MAC腮红, SK-II, 完美日记 |
| 女装 | 2 | ZARA连衣裙, 优衣库防晒衣 |
| 鞋靴 | 2 | SW高跟鞋, 百丽平底鞋 |
| 箱包 | 1 | Coach托特包 |
| 个护 | 3 | 戴森卷发棒, 薇诺娜面膜, 欧莱雅面膜 |
| 其他 | 6 | 笔记本, 显示器, SSD, Apple Watch, 无人机, Switch |

#### 添加新商品

在 `MOCK_PRODUCTS` 列表末尾追加即可：

```python
Product(product_id="P029", name="新商品名", category="新类目", price=99, brand="品牌", seller_id="S26", stock=1000, tags=["标签1", "标签2"]),
```

修改后重启服务生效。

---

### Docker 一键部署（含全部依赖）

如果你想跑完整功能（Redis + Milvus + MySQL）：

```bash
# 在项目根目录
cd multi-agent-ecommerce-system

# 先设置 API Key
export ECOM_LLM_API_KEY=你的API密钥

# 启动所有服务
docker-compose up -d

# 查看状态
docker-compose ps

# 停止
docker-compose down
```

服务地址：

- API: `http://localhost:8000`
- 前端: `http://localhost:5173`
- Swagger 文档: `http://localhost:8000/docs`
- Redis: `localhost:6379`
- Milvus: `localhost:19530`
- MySQL: `localhost:3306`

---

## API 接口文档

### 接口一览

| 方法 | 路径 | 说明 |
| ---- | ---- | ---- |
| GET | `/health` | 健康检查 |
| POST | `/api/v1/recommend` | 核心推荐（Supervisor 编排） |
| POST | `/api/v1/recommend/graph` | LangGraph 状态图推荐 |
| GET | `/api/v1/experiments` | A/B 实验状态 |
| GET | `/api/v1/metrics` | 系统监控指标 |
| POST | `/api/v1/experiments/{id}/outcome` | 记录 A/B 测试结果 |

### 请求/响应示例

```json
POST /api/v1/recommend
Content-Type: application/json

{
  "user_id": "user_001",
  "scene": "homepage",
  "num_items": 5,
  "context": {
    "recent_views": ["手机", "耳机", "充电宝"],
    "avg_order_amount": 500
  }
}
```

```json
{
  "request_id": "a3f8c2d1-...",
  "user_id": "user_001",
  "products": [
    {"product_id": "P001", "name": "iPhone 16 Pro", "category": "手机", "price": 7999.0},
    {"product_id": "P003", "name": "AirPods Pro 3", "category": "耳机", "price": 1899.0}
  ],
  "marketing_copies": [
    {"product_id": "P001", "copy": "根据您最近对手机的兴趣，为您精选 iPhone 16 Pro，好评率 98%。"}
  ],
  "experiment_group": "treatment_llm",
  "total_latency_ms": 1523.4
}
```

---

## 项目文件结构

```text
multi-agent-ecommerce-system/
├── python/                      # 后端（FastAPI）
│   ├── main.py                  # 入口，定义所有路由
│   ├── agents/                  # 四个 Agent
│   │   ├── base_agent.py        # 基类（重试、超时、降级）
│   │   ├── user_profile_agent.py    # 用户画像
│   │   ├── product_rec_agent.py     # 商品推荐（含商品库）
│   │   ├── marketing_copy_agent.py  # 营销文案
│   │   └── inventory_agent.py       # 库存决策
│   ├── orchestrator/
│   │   ├── supervisor.py        # Supervisor 并行编排（核心）
│   │   └── graph.py             # LangGraph 状态图实现
│   ├── services/
│   │   ├── ab_test.py           # A/B 测试引擎（Thompson Sampling）
│   │   ├── feature_store.py     # Redis 特征服务
│   │   └── metrics.py           # 监控指标收集
│   ├── models/schemas.py        # 数据模型定义
│   ├── config/settings.py       # 配置管理
│   └── tests/                   # 单元测试
│
├── frontend/                    # 前端（React + Vite）
│   ├── package.json             # 依赖声明
│   ├── vite.config.js           # Vite 配置（含 API 代理）
│   ├── index.html               # HTML 模板
│   └── src/
│       ├── main.jsx             # React 入口
│       ├── App.jsx              # 页面布局
│       ├── App.css              # 全部样式
│       ├── hooks/
│       │   └── useChat.js       # 核心 Hook（SSE 连接+消息状态）
│       └── components/
│           ├── ChatWindow.jsx   # 消息列表容器
│           ├── MessageBubble.jsx    # 消息气泡
│           ├── ProductCard.jsx      # 商品卡片
│           ├── ProductCardList.jsx  # 商品卡片列表
│           └── ChatInput.jsx        # 输入框
│
├── java/                        # Java 实现（Spring AI Alibaba）
├── go/                          # Go 实现（goroutine 并行）
├── docs/                        # 面试全套文档
│   ├── interview-guide.md       # 面试指南（八股文+STAR法话术）
│   ├── resume-template.md       # 简历模板
│   ├── architecture.md          # 架构设计详解
│   └── code-walkthrough.md      # 代码逐行讲解
├── docker-compose.yml           # Docker 一键部署
└── README.md                    # 本文件
```

---

## 面试资料索引

| 文档 | 内容亮点 | 什么时候看 |
|------|---------|-----------|
| [面试完全指南](docs/interview-guide.md) | 八股文30题（含标准答案）+ STAR法话术 + 面试官追问预案 | **面试前一天通读** |
| [简历模板](docs/resume-template.md) | 应届/社招两套模板，项目经验直接复制 | **投简历时参考** |
| [架构设计文档](docs/architecture.md) | 系统架构图 + Agent职责矩阵 + 稳定性设计 | **被问架构时参考** |
| [代码讲解指南](docs/code-walkthrough.md) | 每个文件逐行解释 + 面试话术 | **被问代码时参考** |

---

## 面试八股文精选（10题）

### Q1：为什么用 Multi-Agent 而不是单个大 Agent？

> 单 Agent 管理几十个工具时，上下文膨胀、推理准确率会明显下降。Multi-Agent 的核心优势有三点：
> 1. **上下文隔离**：每个 Agent 只关注自己领域的工具和数据，Token 消耗少、推理准确
> 2. **并行加速**：4 个 Agent 可以同时跑，端到端延迟约等于最慢 Agent 的耗时
> 3. **独立演进**：各 Agent 可以独立升级、独立做 A/B 测试，互不影响

---

### Q2：Supervisor 模式和 Handoffs 模式有什么区别？

> | | Supervisor 模式 | Handoffs 模式 |
> |--|--|--|
> | 控制方式 | 中枢集中控制 | Agent 间直接传递控制权 |
> | 适合场景 | 流程固定，需要并行 | 对话式，流程动态 |
> | 状态管理 | Supervisor 统一维护 | 每次交接携带上下文 |
> | 本项目 | 采用 | 未采用 |

---

### Q3：`asyncio.gather()` 和串行调用的区别？

> ```python
> # 串行：总耗时 = 3s + 5s = 8s
> profile = await user_profile_agent.run()   # 耗时 3s
> products = await product_rec_agent.run()   # 耗时 5s
>
> # 并行：总耗时 = max(3s, 5s) = 5s
> profile, products = await asyncio.gather(
>     user_profile_agent.run(),
>     product_rec_agent.run(),
> )
> ```
> `asyncio.gather()` 适合 IO 密集型任务（调用 API、查数据库），两个任务同时"等待"，CPU 不浪费。

---

### Q4：Redis Sorted Set 怎么做实时特征？

> ```
> # 写入：用户行为事件
> ZADD user:u001:clicks {timestamp} {product_id}
>
> # 读取：最近1小时的点击
> ZRANGEBYSCORE user:u001:clicks {now-3600} {now}
>
> # 滑动窗口统计
> clicks_1h  = ZCOUNT user:u001:clicks {now-3600} {now}
> clicks_7d  = ZCOUNT user:u001:clicks {now-604800} {now}
> ```
> 用 score=时间戳 的 Sorted Set，天然支持按时间范围查询，时间复杂度 O(log N)。

---

### Q5：A/B 测试的流量分桶怎么保证一致性？

> ```python
> bucket = int(hashlib.md5(user_id.encode()).hexdigest(), 16) % 100
> # 0-59  → control（60%流量）
> # 60-79 → treatment_llm（20%流量）
> # 80-99 → treatment_vector（20%流量）
> ```
> 只要 user_id 不变，分桶结果永远一致，保证实验结论的可靠性。

---

### Q6：Thompson Sampling 怎么动态调流量？

> 核心思想：哪个实验组赢得多，就自动给它更多流量。
>
> ```python
> alpha = {"control": 100, "treatment": 80}   # 点击次数
> beta  = {"control": 50,  "treatment": 20}   # 未点击次数
>
> # 从各组的 Beta 分布采样，取最大值的组
> samples = {group: np.random.beta(alpha[g], beta[g]) for g in groups}
> winner = max(samples, key=samples.get)
> ```

---

### Q7：Agent 调用失败怎么处理？

> 三层保障：
> 1. **超时控制**：`asyncio.wait_for(coro, timeout=5)` — 每个 Agent 独立超时
> 2. **指数退避重试**：失败后等 1s → 2s → 4s 重试，共 3 次
> 3. **降级（Fallback）**：全部失败后返回默认结果，保证系统不崩溃

---

### Q8：LangGraph 和直接写 `asyncio.gather()` 有什么区别？

> | | LangGraph | 直接写 asyncio |
> |--|--|--|
> | 状态管理 | 内置 State，节点间自动传递 | 手动管理变量 |
> | 持久化 | 内置 Checkpoint，支持断点续跑 | 需要自己实现 |
> | 可视化 | 可以画出状态图 | 无 |
> | Human-in-the-loop | 内置支持 | 需要自己实现 |
> | 适合场景 | 复杂、有分支的工作流 | 简单并行任务 |

---

### Q9：RFM 模型怎么计算？

> ```
> R (Recency)  = 距离上次购买的天数    → 越小越好
> F (Frequency)= 一定周期内购买次数    → 越大越好
> M (Monetary) = 累计消费金额          → 越大越好
>
> rfm_score = 0.3 * R_norm + 0.3 * F_norm + 0.4 * M_norm
>
> VIP:       rfm_score > 0.8
> 活跃用户:  0.6 < rfm_score <= 0.8
> 价格敏感:  高 F，低 M
> 流失风险:  rfm_score < 0.3
> ```

---

### Q10：系统延迟怎么优化到 P99 < 2s？

> 四个优化手段：
> 1. **并行化**：Phase1 和 Phase2 各两个 Agent 并行，节省约 50% 时间
> 2. **超时熔断**：单 Agent 超时不等待，返回降级结果
> 3. **Redis 缓存**：用户画像热点数据缓存，命中率 > 80% 时延迟从 200ms → 5ms
> 4. **LLM 精简**：Prompt 控制在 500 Token 以内，减少推理时间

更多30题详见 [docs/interview-guide.md](docs/interview-guide.md)

---

## 简历写法（直接复制）

```
多Agent电商推荐与营销系统 | 个人项目 | 2026.01 - 2026.04
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
* 设计并实现基于 Supervisor 模式的多 Agent 协同架构，含用户画像、商品推荐、
  营销文案、库存决策 4 个专业 Agent，采用并行分发+聚合的编排模式

* 基于 Redis Sorted Set 实现实时用户特征工程（RFM 模型+行为序列），
  特征更新延迟 < 100ms，支持 1h/24h/7d 多时间窗口滑动计算

* 集成 LLM 实现个性化营销文案生成，基于用户画像动态切换 5 套 Prompt 模板，
  文案合规率 100%（广告法敏感词自动过滤）

* 设计流量分桶 + Thompson Sampling A/B 测试引擎，支持 Agent/模型/Prompt
  三层实验，推荐 CTR 提升 15%，文案点击率提升 23%

* 提供 Python(LangGraph) / Java(Spring AI Alibaba) / Go(goroutine) 三语言实现

技术栈：LangGraph · Spring AI Alibaba · Go · Redis · Milvus · FastAPI · Docker
```

---

## 常见问题

### Q: `ModuleNotFoundError: No module named 'structlog'`

依赖没装。确认你激活了正确的 Python 环境，然后执行：

```bash
pip install -r requirements.txt
```

### Q: `curl: (7) Failed to connect to localhost port 8000`

服务没启动，或者在同一个终端里启动了服务（服务会阻塞终端）。解决方法：

1. 先在一个终端运行 `python main.py`
2. 另开一个终端执行 curl 命令

### Q: curl 报错 `no URL specified` 或 `no such file or directory`

多行命令粘贴格式出错。把 curl 命令写成一行：

```bash
curl -X POST http://localhost:8000/api/v1/recommend -H "Content-Type: application/json" -d '{"user_id": "user_001", "scene": "homepage", "num_items": 5}'
```

### Q: 请求很慢（30秒+）

正常。LLM API 调用本身需要时间，推荐模式下系统的流程：

1. 用户画像分析（~6s）
2. 商品 LLM 精排（~15s）
3. 营销文案生成（~5s）

由于用户画像和商品召回阶段是并行执行的，总耗时约等于最慢阶段的耗时，而不是各阶段相加。

### Q: 没有 Redis/Milvus 也能跑吗？

能。所有 Agent 都有 fallback 机制：

- 没有 Redis → 用户画像 Agent 用 LLM 直接生成模拟画像（默认数码爱好者）
- 没有 Milvus → 商品推荐 Agent 用内置的模拟商品库
- 没有 MySQL → 库存 Agent 用模拟数据

核心推荐流程和对话功能不受影响。

### Q: Redis 启动报错 `Bootstrap failed: 5: Input/output error`

`brew services start redis` 在某些 macOS 版本有问题，直接前台启动：

```bash
redis-server /opt/homebrew/etc/redis.conf
```

### Q: 前端 `npm install` 报错找不到 package.json

确认你在 `frontend/` 目录下执行，不是在 `python/` 目录：

```bash
cd multi-agent-ecommerce-system/frontend
npm install
```

### Q: 前端页面打开后发送消息无响应

确认后端服务（`python main.py`）正在运行。前端通过 Vite proxy 把 `/api` 请求转发到 `http://localhost:8000`，后端必须先启动。

---

## 参考资料与致谢

本项目架构设计参考了以下企业级开源项目：

| 项目 | 说明 | 链接 |
|------|------|------|
| NVIDIA Retail Agentic Commerce | NVIDIA 企业级电商 Agent 蓝图 | [GitHub](https://github.com/NVIDIA-AI-Blueprints/Retail-Agentic-Commerce) |
| Spring AI Alibaba Multi-Agent Demo | 阿里巴巴 Java 多 Agent 示例 | [GitHub](https://github.com/spring-ai-alibaba/spring-ai-alibaba-multi-agent-demo) |
| LangGraph 官方文档 | LangGraph 状态图框架 | [文档](https://langchain-ai.github.io/langgraph/) |
| 京东商家智能助手技术博客 | 京东 Multi-Agent 生产实践 | [掘金](https://juejin.cn/post/7470344960563871784) |
| DualAgent-Rec | 双 Agent 推荐系统 | [GitHub](https://github.com/GuilinDev/Dual-Agent-Recommendation) |
| MiniMax API | 本项目默认 LLM 服务 | [官网](https://www.minimax.chat/) |

---

## License

[MIT License](LICENSE) — 随意使用、修改、商用，保留 License 声明即可。

---

<div align="center">

**如果这个项目对你有帮助，欢迎点个 Star！**

有问题欢迎提 [Issue](https://github.com/bcefghj/multi-agent-ecommerce-system/issues)

</div>
