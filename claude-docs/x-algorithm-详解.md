# X（Twitter）"For You" 推荐算法详解

## 一、项目概述

这是 X（原 Twitter）的 **"For You" 信息流推荐系统**的核心代码。它的目标是为每个用户生成个性化的内容推荐，决定用户打开 X 时看到哪些推文。

系统的核心特点是：**完全基于 Grok 大模型进行推荐，几乎没有人工设计的特征工程**。

---

## 二、系统架构

整个系统分为四个主要组件：

```
┌─────────────────────────────────────────────────────────────────┐
│                      用户请求 For You 信息流                      │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                    HOME MIXER（编排层）                           │
│  负责协调各个组件，组装最终的推荐结果                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   候选内容来源：                                                  │
│   ┌───────────────────┐    ┌───────────────────┐                │
│   │     THUNDER       │    │  PHOENIX RETRIEVAL│                │
│   │   （网内内容）      │    │  （网外内容）      │                │
│   │ 你关注的人的推文    │    │ ML检索发现的推文   │                │
│   └───────────────────┘    └───────────────────┘                │
│                                                                 │
│   打分排序：                                                      │
│   ┌───────────────────────────────────────────────────────────┐ │
│   │                   PHOENIX SCORER                           │ │
│   │            Grok Transformer 预测用户行为概率                 │ │
│   └───────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                        排序后的推荐结果                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 三、四大核心组件详解

### 1. Home Mixer（编排层）

**位置**：`home-mixer/`

这是整个系统的"指挥中心"，负责协调所有组件的工作流程。它基于 **Candidate Pipeline** 框架，按以下阶段执行：

| 阶段 | 功能 |
|------|------|
| Query Hydration | 获取用户上下文（历史互动、关注列表） |
| Sources | 从 Thunder 和 Phoenix 获取候选推文 |
| Hydrators | 补充推文的详细信息（作者、媒体等） |
| Filters | 过滤不合适的内容 |
| Scorers | 用 ML 模型预测并计算分数 |
| Selector | 按分数排序，选择 Top K |
| Post-Selection Filters | 最终的可见性检查 |

### 2. Thunder（网内内容服务）

**位置**：`thunder/`

Thunder 是一个**内存级实时推文存储系统**，专门处理用户关注的人发布的内容（称为"网内内容"）。

工作原理：
- 从 Kafka 消费推文的创建/删除事件
- 在内存中维护每个用户的推文索引
- 查询延迟在**亚毫秒级**，无需访问数据库
- 自动清理过期推文

### 3. Phoenix（ML 核心）

**位置**：`phoenix/`

Phoenix 是整个系统的**机器学习大脑**，有两个核心功能：

#### （1）检索（Retrieval）- 双塔模型

用于从全局数百万推文中快速筛选出相关内容：

```
用户塔 (User Tower)          候选塔 (Candidate Tower)
     │                              │
     ▼                              ▼
用户特征 + 历史行为          所有推文的特征
     │                              │
     ▼                              ▼
  用户向量 [D维]              推文向量 [D维]
     │                              │
     └──────────┬───────────────────┘
                │
                ▼
         点积相似度搜索
                │
                ▼
         返回 Top-K 候选
```

#### （2）排序（Ranking）- Grok Transformer

这是系统的核心创新点。使用基于 **xAI Grok-1** 改造的 Transformer 模型进行精细排序。

**候选隔离机制（Candidate Isolation）**：

```
注意力矩阵可视化：

              Keys（被关注的位置）
              ────────────────────────────────────────────►
              │ 用户 │    历史行为序列    │     候选推文      │
         ┌────┼──────┼──────────────────┼──────────────────┤
         │用户│  ✓   │       ✓          │        ✗         │
   Q     │历史│  ✓   │       ✓          │        ✗         │
   u     │    │  ✓   │       ✓          │        ✗         │
   e     ├────┼──────┼──────────────────┼──────────────────┤
   r     │    │  ✓   │       ✓          │ ✓(自己) ✗ ✗ ✗   │
   i     │候选│  ✓   │       ✓          │   ✗  ✓(自己) ✗ ✗ │
   e     │推文│  ✓   │       ✓          │   ✗  ✗  ✓(自己) ✗│
   s     │    │  ✓   │       ✓          │   ✗  ✗  ✗  ✓    │
         └────┴──────┴──────────────────┴──────────────────┘

   ✓ = 可以关注    ✗ = 不能关注
```

**为什么要候选隔离？**
- 每个候选推文只能看到用户上下文，不能看到其他候选
- 这保证了推文的分数**不依赖于批次中有哪些其他推文**
- 使得分数**可缓存、可复用**，大幅提升系统效率

### 4. Candidate Pipeline（管道框架）

**位置**：`candidate-pipeline/`

这是一个**可复用的推荐管道框架**，定义了标准化的组件接口：

```rust
// 核心 Trait 定义
trait Source<Q, C>      // 获取候选
trait Hydrator<Q, C>    // 补充数据
trait Filter<Q, C>      // 过滤候选
trait Scorer<Q, C>      // 计算分数
trait Selector<Q, C>    // 排序选择
trait SideEffect<Q, C>  // 副作用（日志、缓存）
```

管道执行流程：

```
hydrate_query → fetch_candidates → hydrate → filter → score → select → filter_post_selection
```

---

## 四、打分机制详解

Phoenix 模型会预测**多种用户行为的概率**：

| 正向行为 | 负向行为 |
|---------|---------|
| P(点赞) | P(不感兴趣) |
| P(回复) | P(拉黑作者) |
| P(转发) | P(静音作者) |
| P(引用) | P(举报) |
| P(点击) | |
| P(分享) | |
| P(关注作者) | |

最终分数计算公式：

```
最终分数 = Σ (权重_i × P(行为_i))

其中：
- 正向行为权重 > 0（鼓励）
- 负向行为权重 < 0（惩罚）
```

这意味着：如果模型预测你可能会拉黑某个作者，这条推文的分数会被大幅拉低。

### 分数计算代码示例

来自 `home-mixer/scorers/weighted_scorer.rs`：

```rust
fn compute_weighted_score(candidate: &PostCandidate) -> f64 {
    let s: &PhoenixScores = &candidate.phoenix_scores;

    let combined_score =
        Self::apply(s.favorite_score, FAVORITE_WEIGHT)      // 点赞
        + Self::apply(s.reply_score, REPLY_WEIGHT)          // 回复
        + Self::apply(s.retweet_score, RETWEET_WEIGHT)      // 转发
        + Self::apply(s.share_score, SHARE_WEIGHT)          // 分享
        + Self::apply(s.follow_author_score, FOLLOW_WEIGHT) // 关注
        // 负向行为（负权重）
        + Self::apply(s.not_interested_score, NOT_INTERESTED_WEIGHT)
        + Self::apply(s.block_author_score, BLOCK_AUTHOR_WEIGHT)
        + Self::apply(s.mute_author_score, MUTE_AUTHOR_WEIGHT)
        + Self::apply(s.report_score, REPORT_WEIGHT);

    combined_score
}
```

---

## 五、过滤机制

系统有两轮过滤：

### 打分前过滤

位置：`home-mixer/filters/`

| 过滤器 | 功能 |
|--------|------|
| `drop_duplicates_filter` | 去重 |
| `age_filter` | 过滤太旧的推文 |
| `self_tweet_filter` | 过滤自己的推文 |
| `muted_keyword_filter` | 过滤静音关键词 |
| `author_socialgraph_filter` | 过滤拉黑/静音的作者 |
| `previously_seen_posts_filter` | 过滤已看过的推文 |
| `previously_served_posts_filter` | 过滤本次会话已展示的推文 |
| `ineligible_subscription_filter` | 过滤无权访问的付费内容 |
| `retweet_deduplication_filter` | 转发去重 |
| `core_data_hydration_filter` | 过滤元数据获取失败的推文 |

### 选择后过滤

| 过滤器 | 功能 |
|--------|------|
| `vf_filter` | 可见性过滤（删除/垃圾/暴力/血腥等） |
| `dedup_conversation_filter` | 对话线程去重 |

---

## 六、关键设计决策

### 1. 零人工特征

完全依赖 Grok Transformer 从用户行为序列中学习，不做人工特征工程。

**优点**：
- 大大简化了数据管道
- 减少了维护成本
- 模型可以学习到人类难以设计的复杂模式

### 2. 候选隔离

在 Transformer 推理时，候选推文之间不能互相关注（attention）。

**优点**：
- 保证分数的一致性
- 分数可缓存、可复用
- 不会因为批次组成不同而导致分数变化

### 3. 哈希嵌入

使用多个哈希函数进行嵌入查找。

**优点**：
- 处理超大词汇表（用户ID、推文ID等）
- 无需维护显式的嵌入表
- 自动处理冷启动问题

### 4. 多行为预测

同时预测多种行为概率，而非单一"相关性"分数。

**优点**：
- 可以灵活调整各行为的权重
- 通过负向行为惩罚降低低质量内容
- 更精细地理解用户偏好

### 5. 并行执行

- Sources 和 Hydrators 可**并行**执行
- Filters 和 Scorers **顺序**执行

**原因**：
- 数据获取阶段相互独立，可以并行加速
- 过滤和打分依赖前序结果，必须顺序执行

---

## 七、技术栈总结

| 组件 | 语言/框架 | 用途 |
|------|----------|------|
| home-mixer | Rust + gRPC + Tonic | 服务编排层 |
| thunder | Rust + Kafka | 实时推文存储 |
| phoenix | Python 3.11 + JAX + dm-haiku | ML 模型 |
| candidate-pipeline | Rust | 通用管道框架 |

### 开发工具

| 工具 | 用途 |
|------|------|
| uv | Python 包管理 |
| ruff | Python 代码检查 |
| pyright | Python 类型检查 |
| pytest | Python 测试 |

---

## 八、代码结构

```
x-algorithm/
├── home-mixer/                 # 编排层（Rust）
│   ├── candidate_pipeline/     # 管道配置
│   ├── candidate_hydrators/    # 数据补充器
│   ├── filters/                # 过滤器
│   ├── scorers/                # 打分器
│   ├── selectors/              # 选择器
│   ├── sources/                # 数据源
│   ├── query_hydrators/        # 查询补充器
│   ├── side_effects/           # 副作用处理
│   ├── server.rs               # gRPC 服务
│   └── main.rs                 # 入口
│
├── thunder/                    # 网内内容服务（Rust）
│   ├── kafka/                  # Kafka 消费
│   ├── posts/                  # 推文存储
│   └── thunder_service.rs      # 服务实现
│
├── phoenix/                    # ML 模型（Python）
│   ├── grok.py                 # Grok Transformer 架构
│   ├── recsys_model.py         # 排序模型
│   ├── recsys_retrieval_model.py # 检索模型
│   ├── runners.py              # 推理运行器
│   ├── run_ranker.py           # 排序入口
│   ├── run_retrieval.py        # 检索入口
│   └── test_*.py               # 测试文件
│
└── candidate-pipeline/         # 管道框架（Rust）
    ├── candidate_pipeline.rs   # 核心管道逻辑
    ├── source.rs               # Source trait
    ├── hydrator.rs             # Hydrator trait
    ├── filter.rs               # Filter trait
    ├── scorer.rs               # Scorer trait
    └── selector.rs             # Selector trait
```

---

## 九、运行命令

### Phoenix ML 模型

```bash
cd phoenix

# 安装依赖
uv sync

# 运行排序模型
uv run python run_ranker.py

# 运行检索模型
uv run python run_retrieval.py

# 运行测试
uv run pytest

# 类型检查
uv run pyright

# 代码检查
uv run ruff check .
```

---

## 十、总结

X 的 "For You" 推荐算法是一个**端到端的深度学习推荐系统**：

1. **输入**：用户的历史行为序列（点赞、转发、回复等）
2. **处理**：Grok Transformer 理解用户偏好
3. **输出**：对每条候选推文预测多种行为概率
4. **排序**：加权求和得到最终分数

核心创新在于**完全依赖大模型**，摒弃了传统推荐系统中大量的人工特征工程，让模型自己学习什么是"好内容"。
