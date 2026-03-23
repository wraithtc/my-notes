# Query 重写技术方案

## 1. 概述

### 1.1 背景

Query 重写（Query Rewriting）是搜索和推荐系统中的核心技术之一，旨在将用户输入的原始查询转换为更易于检索和理解的形式，从而提升搜索结果的准确性和召回率。

### 1.2 核心价值

| 价值点 | 说明 |
|--------|------|
| **语义扩展** | 将简短查询扩展为更完整的语义表达 |
| **歧义消解** | 解决一词多义、多词一义的问题 |
| **意图对齐** | 将用户意图与系统理解进行对齐 |
| **召回增强** | 通过同义词、相关词扩展提升召回覆盖 |

### 1.3 应用场景

- 电商搜索：商品搜索、意图识别
- 内容搜索：文档检索、知识库问答
- 广告系统：关键词匹配、竞价优化
- 推荐系统：用户兴趣扩展

---

## 2. 技术目标

### 2.1 核心指标

```
目标指标：
├── 召回率提升：+15%~30%
├── 精准率保持：不低于基线
├── 延迟控制：<50ms (P99)
└── 覆盖率：>95% 的查询能被处理
```

### 2.2 质量要求

- **相关性**：重写后的 query 必须与原 query 语义相关
- **多样性**：提供多个候选重写，覆盖不同语义角度
- **可控性**：支持业务规则干预和配置

---

## 3. 技术方案架构

### 3.1 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                      Query 重写系统                          │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │   预处理层   │ -> │  重写引擎层  │ -> │  后处理层   │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│         │                  │                  │              │
│         v                  v                  v              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │  规则过滤   │    │  多策略融合  │    │  排序&过滤  │     │
│  │  归一化     │    │  LLM/向量   │    │  去重       │     │
│  │  纠错       │    │  同义词库   │    │  截断       │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 处理流程

```
输入 Query
    │
    v
┌───────────────┐
│  1. 文本预处理 │ ── 纠错、归一化、分词
└───────────────┘
    │
    v
┌───────────────┐
│  2. 意图识别   │ ── 分类、实体抽取
└───────────────┘
    │
    v
┌───────────────┐
│  3. 多策略重写 │ ── 同义词/LLM/向量/规则
└───────────────┘
    │
    v
┌───────────────┐
│  4. 候选排序   │ ── 相关性/多样性/业务权重
└───────────────┘
    │
    v
┌───────────────┐
│  5. 后处理输出 │ ── 去重、截断、格式化
└───────────────┘
    │
    v
输出重写结果
```

---

## 4. 核心技术模块

### 4.1 预处理模块

#### 4.1.1 文本纠错

```python
class QueryCorrector:
    """查询纠错模块"""

    def __init__(self):
        self.error_model = load_model("bert_correction")
        self.custom_dict = load_custom_dict()

    def correct(self, query: str) -> str:
        # 1. 拼写纠错
        corrected = self.spell_check(query)

        # 2. 同音字纠错
        corrected = self.homophone_correct(corrected)

        # 3. 形近字纠错
        corrected = self.shape_correct(corrected)

        return corrected
```

#### 4.1.2 文本归一化

```python
class QueryNormalizer:
    """查询归一化模块"""

    def normalize(self, query: str) -> str:
        # 1. 大小写归一化
        query = query.lower()

        # 2. 全半角转换
        query = self.fullwidth_to_halfwidth(query)

        # 3. 特殊字符处理
        query = self.remove_special_chars(query)

        # 4. 停用词处理（可选）
        query = self.handle_stopwords(query)

        return query
```

### 4.2 重写引擎模块

#### 4.2.1 同义词扩展

```python
class SynonymExpander:
    """同义词扩展器"""

    def __init__(self):
        self.synonym_dict = self.load_synonyms()
        self.word2vec = self.load_embeddings()

    def expand(self, query: str, top_k: int = 5) -> List[str]:
        """
        同义词扩展
        - 精确同义词：来自业务词表
        - 语义近义词：来自词向量相似度
        """
        terms = self.tokenize(query)
        expansions = []

        for term in terms:
            # 精确同义词
            exact_synonyms = self.synonym_dict.get(term, [])

            # 语义近义词
            semantic_synonyms = self.word2vec.most_similar(term, topn=3)

            expansions.extend(exact_synonyms + semantic_synonyms)

        return self.rank_and_filter(expansions, top_k)
```

#### 4.2.2 LLM 重写

```python
class LLMRewriter:
    """基于大模型的重写器"""

    SYSTEM_PROMPT = """你是一个专业的搜索查询重写助手。
你的任务是将用户的搜索查询重写为更清晰、更完整的形式。

重写规则：
1. 保持原始查询的核心意图
2. 扩展模糊或简短的表达
3. 添加必要的上下文信息
4. 生成 3-5 个不同角度的重写版本

输出格式：JSON 数组
"""

    def __init__(self, model_name: str = "gpt-4"):
        self.llm = LLMClient(model_name)

    def rewrite(self, query: str, context: dict = None) -> List[dict]:
        """
        LLM 重写主函数

        Args:
            query: 原始查询
            context: 用户上下文信息（历史、偏好等）

        Returns:
            重写结果列表，包含 query 和 confidence
        """
        prompt = self.build_prompt(query, context)

        response = self.llm.generate(
            system_prompt=self.SYSTEM_PROMPT,
            user_prompt=prompt,
            temperature=0.3,  # 较低温度保证稳定性
            response_format={"type": "json_object"}
        )

        return self.parse_response(response)
```

**Prompt 模板设计：**

```
原始查询：{query}
用户历史：{user_history}
业务场景：{domain}

请对上述查询进行重写，要求：
1. 语义扩展：补充完整表达
2. 歧义消解：明确查询意图
3. 同义替换：提供多种表达方式

输出 JSON 格式：
{
  "rewrites": [
    {
      "query": "重写后的查询",
      "type": "expansion|clarification|synonym",
      "confidence": 0.95,
      "reason": "重写理由"
    }
  ]
}
```

#### 4.2.3 向量检索重写

```python
class VectorRewriter:
    """基于向量检索的重写器"""

    def __init__(self):
        self.encoder = SentenceEncoder("bge-large-zh")
        self.index = FaissIndex(dimension=1024)

    def rewrite(self, query: str, top_k: int = 10) -> List[dict]:
        """
        向量检索重写

        原理：在历史查询库中检索语义相似的查询
        这些相似查询可以作为重写候选
        """
        # 编码
        query_vec = self.encoder.encode(query)

        # 检索相似历史查询
        results = self.index.search(query_vec, top_k)

        # 过滤和排序
        return [
            {
                "query": r["text"],
                "score": r["score"],
                "source": "vector_retrieval"
            }
            for r in results
            if r["score"] > 0.8  # 相似度阈值
        ]
```

#### 4.2.4 规则重写

```python
class RuleRewriter:
    """基于规则的重写器"""

    def __init__(self):
        self.rules = self.load_rules()

    def load_rules(self) -> List[Rule]:
        """
        规则类型：
        1. 精确匹配规则：query -> rewrite
        2. 模板规则：pattern -> template
        3. 正则规则：regex -> replacement
        """
        return [
            # 精确匹配
            ExactMatchRule("苹果", ["iPhone", "Apple 手机", "苹果手机"]),

            # 模板规则
            TemplateRule(
                pattern="{brand} {category}",
                templates=["{brand}官方{category}", "正品{brand}{category}"]
            ),

            # 正则规则
            RegexRule(
                pattern=r"(\d+)元?(以下|以内|内)",
                replacement=lambda m: f"价格低于{m.group(1)}元"
            ),
        ]

    def rewrite(self, query: str) -> List[dict]:
        results = []
        for rule in self.rules:
            if rule.match(query):
                results.extend(rule.apply(query))
        return results
```

### 4.3 多策略融合

```python
class QueryRewriter:
    """Query 重写主控制器"""

    def __init__(self):
        self.corrector = QueryCorrector()
        self.normalizer = QueryNormalizer()
        self.rewriters = {
            "synonym": SynonymExpander(),
            "llm": LLMRewriter(),
            "vector": VectorRewriter(),
            "rule": RuleRewriter(),
        }
        self.ranker = RewriteRanker()

    def rewrite(
        self,
        query: str,
        strategies: List[str] = None,
        top_k: int = 10
    ) -> List[dict]:
        """
        多策略重写主函数

        Args:
            query: 原始查询
            strategies: 使用的策略列表，默认全部
            top_k: 返回 top_k 个结果

        Returns:
            重写结果列表
        """
        # 1. 预处理
        query = self.corrector.correct(query)
        query = self.normalizer.normalize(query)

        # 2. 多策略重写
        all_candidates = []
        strategies = strategies or list(self.rewriters.keys())

        for strategy in strategies:
            rewriter = self.rewriters[strategy]
            candidates = rewriter.rewrite(query)
            for c in candidates:
                c["strategy"] = strategy
            all_candidates.extend(candidates)

        # 3. 排序融合
        ranked = self.ranker.rank(query, all_candidates)

        # 4. 多样性重排
        diverse = self.diversify(ranked)

        # 5. 截断返回
        return diverse[:top_k]

    def diversify(
        self,
        candidates: List[dict],
        lambda_param: float = 0.7
    ) -> List[dict]:
        """
        MMR 多样性重排

        Score = λ * Relevance - (1-λ) * Max_Similarity
        """
        selected = []
        remaining = candidates.copy()

        while remaining and len(selected) < self.max_results:
            best_score = -float('inf')
            best_idx = 0

            for i, cand in enumerate(remaining):
                # 相关性分数
                relevance = cand.get("score", 0)

                # 与已选结果的相似度
                max_sim = max(
                    [self.similarity(cand, s) for s in selected],
                    default=0
                )

                # MMR 分数
                mmr_score = lambda_param * relevance - (1 - lambda_param) * max_sim

                if mmr_score > best_score:
                    best_score = mmr_score
                    best_idx = i

            selected.append(remaining.pop(best_idx))

        return selected
```

---

## 5. 排序模型

### 5.1 相关性排序

```python
class RewriteRanker:
    """重写结果排序器"""

    def __init__(self):
        self.encoder = CrossEncoder("bge-reranker-large")
        self.feature_extractor = FeatureExtractor()

    def rank(
        self,
        original_query: str,
        candidates: List[dict]
    ) -> List[dict]:
        """
        对重写候选进行排序

        特征：
        1. 语义相似度（Cross-Encoder）
        2. 词重叠率
        3. 策略权重
        4. 长度惩罚
        5. 业务特征
        """
        for cand in candidates:
            # 语义相似度
            semantic_score = self.encoder.predict(
                [(original_query, cand["query"])]
            )[0]

            # 词重叠
            overlap_score = self.word_overlap(original_query, cand["query"])

            # 综合打分
            cand["final_score"] = (
                0.5 * semantic_score +
                0.2 * overlap_score +
                0.15 * self.strategy_weight(cand["strategy"]) +
                0.15 * self.business_features(cand)
            )

        # 按分数排序
        return sorted(candidates, key=lambda x: x["final_score"], reverse=True)
```

### 5.2 策略权重配置

```yaml
# strategy_weights.yaml
strategy_weights:
  llm:
    base_weight: 0.9
    confidence_threshold: 0.7
    max_candidates: 5

  synonym:
    base_weight: 0.8
    similarity_threshold: 0.75
    max_candidates: 3

  vector:
    base_weight: 0.7
    similarity_threshold: 0.8
    max_candidates: 5

  rule:
    base_weight: 1.0  # 规则优先级最高
    max_candidates: 10
```

---

## 6. 评估体系

### 6.1 离线评估

```python
class QueryRewriteEvaluator:
    """离线评估器"""

    def evaluate(self, test_set: List[dict]) -> dict:
        """
        评估指标

        Args:
            test_set: 测试集，包含 (query, ground_truth) 对
        """
        metrics = {
            "recall@k": [],     # 召回率
            "precision@k": [],  # 精准率
            "mrr": [],          # 平均倒数排名
            "ndcg": [],         # 归一化折损累计增益
            "latency": [],      # 延迟
        }

        for sample in test_set:
            query = sample["query"]
            ground_truth = sample["relevant_docs"]

            # 执行重写
            start_time = time.time()
            rewrites = self.rewriter.rewrite(query)
            latency = time.time() - start_time

            # 使用重写结果检索
            results = self.searcher.search(rewrites)

            # 计算指标
            metrics["recall@k"].append(
                self.compute_recall(results, ground_truth, k=10)
            )
            metrics["precision@k"].append(
                self.compute_precision(results, ground_truth, k=10)
            )
            metrics["mrr"].append(
                self.compute_mrr(results, ground_truth)
            )
            metrics["ndcg"].append(
                self.compute_ndcg(results, ground_truth, k=10)
            )
            metrics["latency"].append(latency)

        return {k: np.mean(v) for k, v in metrics.items()}
```

### 6.2 在线评估

```python
# A/B 测试指标
online_metrics = {
    "搜索点击率": "ctr",
    "无结果率": "zero_result_rate",
    "搜索转化率": "conversion_rate",
    "用户满意度": "user_satisfaction",
    "查询修改率": "query_reformulation_rate",
}
```

### 6.3 评估数据集构建

```
数据集结构：
├── query               # 原始查询
├── relevant_docs       # 相关文档列表
├── user_clicks         # 用户点击日志
├── satisfaction_label  # 满意度标注 (1-5)
└── rewrite_labels      # 人工标注的重写质量
```

---

## 7. 工程实现

### 7.1 服务架构

```yaml
# service architecture
query_rewrite_service:
  name: query-rewrite-service
  port: 8080
  replicas: 3

  components:
    - name: api_gateway
      type: nginx
      config:
        rate_limit: 1000/qps
        timeout: 100ms

    - name: rewrite_engine
      type: python_service
      framework: fastapi
      dependencies:
        - redis_cluster
        - llm_service
        - vector_db

    - name: llm_service
      type: inference_service
      model: qwen-72b
      batch_size: 32

    - name: vector_db
      type: milvus
      collection: query_history
      dimension: 1024
```

### 7.2 接口设计

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI(title="Query Rewrite Service")

class RewriteRequest(BaseModel):
    query: str
    user_id: str = None
    context: dict = None
    strategies: List[str] = None
    top_k: int = 10

class RewriteResponse(BaseModel):
    original_query: str
    rewrites: List[dict]
    latency_ms: float
    strategy_used: List[str]

@app.post("/v1/rewrite", response_model=RewriteResponse)
async def rewrite(request: RewriteRequest):
    """
    Query 重写接口

    示例请求：
    {
        "query": "苹果手机",
        "user_id": "user_123",
        "context": {"category": "electronics"},
        "top_k": 5
    }

    示例响应：
    {
        "original_query": "苹果手机",
        "rewrites": [
            {"query": "iPhone 苹果手机", "score": 0.95, "strategy": "llm"},
            {"query": "Apple 手机", "score": 0.92, "strategy": "synonym"},
            {"query": "苹果 iPhone 正品", "score": 0.88, "strategy": "rule"}
        ],
        "latency_ms": 45.2,
        "strategy_used": ["llm", "synonym", "rule"]
    }
    """
    start_time = time.time()

    rewriter = QueryRewriter()
    results = rewriter.rewrite(
        query=request.query,
        strategies=request.strategies,
        top_k=request.top_k
    )

    latency = (time.time() - start_time) * 1000

    return RewriteResponse(
        original_query=request.query,
        rewrites=results,
        latency_ms=latency,
        strategy_used=list(set(r["strategy"] for r in results))
    )
```

### 7.3 缓存策略

```python
class QueryRewriteCache:
    """多级缓存"""

    def __init__(self):
        # L1: 本地内存缓存 (热点 query)
        self.local_cache = LRUCache(max_size=10000)

        # L2: Redis 分布式缓存
        self.redis = RedisClient()

        # L3: 向量缓存 (语义相似)
        self.vector_cache = VectorCache()

    async def get(self, query: str) -> Optional[List[dict]]:
        # L1 查询
        if result := self.local_cache.get(query):
            return result

        # L2 查询
        cache_key = self.hash_query(query)
        if result := await self.redis.get(cache_key):
            self.local_cache.set(query, result)
            return result

        # L3 语义相似查询
        similar = await self.vector_cache.find_similar(query, threshold=0.95)
        if similar:
            return similar

        return None

    async def set(self, query: str, result: List[dict], ttl: int = 3600):
        # 写入多级缓存
        self.local_cache.set(query, result)
        await self.redis.set(self.hash_query(query), result, ttl=ttl)
        await self.vector_cache.add(query, result)
```

### 7.4 性能优化

```python
class AsyncQueryRewriter:
    """异步并行重写"""

    async def rewrite(self, query: str) -> List[dict]:
        # 并行执行多个策略
        tasks = [
            self.rewrite_with_synonym(query),
            self.rewrite_with_llm(query),
            self.rewrite_with_vector(query),
            self.rewrite_with_rule(query),
        ]

        results = await asyncio.gather(*tasks, return_exceptions=True)

        # 合并结果
        all_candidates = []
        for result in results:
            if not isinstance(result, Exception):
                all_candidates.extend(result)

        return self.rank_and_diversify(all_candidates)
```

---

## 8. 监控与运维

### 8.1 监控指标

```yaml
# Prometheus 指标
metrics:
  - name: query_rewrite_qps
    type: counter
    labels: [strategy, status]

  - name: query_rewrite_latency
    type: histogram
    buckets: [10ms, 25ms, 50ms, 100ms, 200ms]

  - name: query_rewrite_candidates_count
    type: histogram
    buckets: [1, 3, 5, 10, 20]

  - name: llm_call_latency
    type: histogram
    buckets: [50ms, 100ms, 200ms, 500ms]

  - name: cache_hit_rate
    type: gauge
```

### 8.2 告警规则

```yaml
# alerting_rules.yaml
groups:
  - name: query_rewrite_alerts
    rules:
      - alert: HighLatency
        expr: histogram_quantile(0.99, query_rewrite_latency) > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Query 重写延迟过高"

      - alert: LowCacheHitRate
        expr: cache_hit_rate < 0.5
        for: 10m
        labels:
          severity: warning

      - alert: LLMServiceDown
        expr: up{service="llm_service"} == 0
        for: 1m
        labels:
          severity: critical
```

### 8.3 日志规范

```python
import structlog

logger = structlog.get_logger()

def log_rewrite_request(
    query: str,
    rewrites: List[dict],
    latency: float,
    strategies: List[str]
):
    logger.info(
        "query_rewrite_complete",
        query=query,
        rewrite_count=len(rewrites),
        latency_ms=latency,
        strategies=strategies,
        top_score=rewrites[0]["score"] if rewrites else None,
    )
```

---

## 9. 业务适配

### 9.1 领域适配

```python
class DomainAdapter:
    """领域适配器"""

    def __init__(self, domain: str):
        self.domain = domain
        self.config = self.load_domain_config(domain)

    def load_domain_config(self, domain: str) -> dict:
        """加载领域特定配置"""
        configs = {
            "ecommerce": {
                "synonym_file": "ecommerce_synonyms.txt",
                "entity_types": ["brand", "category", "attribute"],
                "rewrite_templates": [
                    "{brand} {category} 正品",
                    "{category} {attribute}",
                ],
            },
            "content": {
                "synonym_file": "content_synonyms.txt",
                "entity_types": ["topic", "author", "keyword"],
            },
        }
        return configs.get(domain, {})

    def adapt_rewrites(self, rewrites: List[dict]) -> List[dict]:
        """根据领域特点调整重写结果"""
        # 应用领域规则
        # 添加领域特定实体
        return adapted_rewrites
```

### 9.2 用户个性化

```python
class PersonalizedRewriter:
    """个性化重写"""

    def __init__(self):
        self.user_profile = UserProfileService()

    def personalize(
        self,
        query: str,
        user_id: str,
        rewrites: List[dict]
    ) -> List[dict]:
        """
        根据用户画像调整重写

        考虑因素：
        1. 用户历史偏好
        2. 用户兴趣标签
        3. 用户行为序列
        """
        profile = self.user_profile.get(user_id)

        for rewrite in rewrites:
            # 用户偏好加权
            preference_boost = self.compute_preference_boost(
                rewrite, profile
            )
            rewrite["score"] += preference_boost

        return sorted(rewrites, key=lambda x: x["score"], reverse=True)
```

---

## 10. 迭代优化

### 10.1 数据闭环

```
用户行为数据收集
        │
        v
┌───────────────────┐
│   数据清洗 & 标注  │
└───────────────────┘
        │
        v
┌───────────────────┐
│   模型训练/更新    │
└───────────────────┘
        │
        v
┌───────────────────┐
│   离线评估验证     │
└───────────────────┘
        │
        v
┌───────────────────┐
│   灰度发布上线     │
└───────────────────┘
        │
        v
    效果监控
```

### 10.2 持续学习

```python
class ContinuousLearner:
    """持续学习模块"""

    def __init__(self):
        self.feedback_collector = FeedbackCollector()
        self.model_updater = ModelUpdater()

    async def collect_feedback(self, event: dict):
        """收集用户反馈"""
        # 点击日志
        # 转化日志
        # 负反馈（重新搜索、跳出）
        await self.feedback_collector.add(event)

    def train_incremental(self):
        """增量训练"""
        feedback_data = self.feedback_collector.get_batch()

        # 更新排序模型
        self.model_updater.update_ranker(feedback_data)

        # 更新同义词库
        new_synonyms = self.extract_synonyms(feedback_data)
        self.update_synonym_dict(new_synonyms)
```

---

## 11. 总结

### 11.1 技术选型建议

| 场景 | 推荐方案 | 说明 |
|------|----------|------|
| 快速上线 | 规则 + 同义词 | 实现简单，效果可控 |
| 中等规模 | 规则 + 同义词 + 向量 | 召回增强 |
| 大规模高精度 | 全策略融合 + LLM | 效果最优，成本较高 |

### 11.2 关键成功因素

1. **数据质量**：高质量的同义词库和标注数据
2. **策略平衡**：多策略融合时的权重调优
3. **性能保障**：延迟控制，特别是 LLM 调用
4. **业务对齐**：重写结果需符合业务语义
5. **持续迭代**：建立数据闭环，持续优化

### 11.3 风险与应对

| 风险 | 应对措施 |
|------|----------|
| LLM 幻觉 | 多策略交叉验证，置信度过滤 |
| 延迟过高 | 异步并行，缓存优化，降级策略 |
| 语义漂移 | 相关性阈值控制，人工审核机制 |
| 冷启动问题 | 规则兜底，迁移学习 |

---

## 参考资料

- [Query Understanding for Search Engines](https://example.com)
- [Learning to Rewrite Queries](https://example.com)
- [LLM for Query Rewriting](https://example.com)
- [Semantic Search at Scale](https://example.com)
