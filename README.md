# 🚗 AutoDrive RAG — 自动驾驶研发知识检索系统

> 面向自动驾驶研发团队的 RAG 知识检索系统，接入算法文档、测试规范、传感器手册、法规标准，基于 RAG + MCP 提供引用式问答与工具协同能力。

[![Python](https://img.shields.io/badge/Python-3.11+-blue.svg)](https://www.python.org/downloads/)
[![Tests](https://img.shields.io/badge/Tests-89%20passing-brightgreen.svg)]()
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

## 项目背景

自动驾驶研发团队日常面对海量技术文档：传感器规格书、感知/规划/控制算法设计文档、GB/T 国家标准、ISO 26262 功能安全标准、测试场景库……这些文档分散、更新频繁，工程师查找信息效率极低。

本项目构建了一套基于 RAG 的知识检索系统，将这些文档统一接入，支持自然语言查询，每条回答都带有可追溯的文档引用。同时通过 MCP 协议暴露为工具服务，可被外部 AI Agent 或研发工具直接调用。

## 系统架构

```
用户查询
    │
    ▼
┌─────────────────┐
│  Query Analyzer  │  复杂度检测 / 意图分类 / AD 术语识别
└────────┬────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌────────┐ ┌──────────────┐
│  BM25  │ │ Dense Embed  │  双路并行检索
└────────┘ └──────────────┘
    │              │
    └──────┬───────┘
           ▼
    ┌─────────────┐
    │  RRF Fusion │  倒数排名融合，平衡精确率与召回率
    └──────┬──────┘
           ▼
    ┌──────────────────┐
    │ Metadata Booster │  查询类型感知权重提升
    │                  │  传感器查询 → sensor_doc ×1.5
    │                  │  法规查询   → regulation_doc ×1.6
    └──────┬───────────┘
           ▼
    ┌──────────────┐
    │   Reranker   │  Cross-Encoder 精排
    └──────┬───────┘
           ▼
    ┌──────────────────┐
    │ Response Builder │  多文档综合 / 传感器对比 / 算法对比
    └──────┬───────────┘
           ▼
    ┌──────────────────┐
    │ Boundary Validator│  拒绝预测性查询 / 实时诊断请求
    └──────┬───────────┘
           ▼
    引用式回答 + MCP 工具服务 + Streamlit Dashboard
```

## 核心技术亮点

**1. 混合检索引擎**

BM25 稀疏检索与 Dense Embedding 稠密检索双路并行，通过 RRF（倒数排名融合）算法合并结果。BM25 擅长精确关键词匹配（如标准编号 "ISO 26262-3"），Dense 擅长语义理解（如 "感知模块如何处理遮挡"），两者互补。

**2. Metadata Boost — 查询类型感知的权重提升**

检测查询意图后，对目标文档类型动态加权：

```python
BOOST_CONFIG = {
    "sensor_query":     {"sensor_doc": 1.5, "algorithm_doc": 0.8},
    "regulation_query": {"regulation_doc": 1.6, "test_doc": 1.2},
    "algorithm_query":  {"algorithm_doc": 1.3},
}
```

确保传感器查询的 top-3 结果中至少 2 个来自传感器文档，避免通用文档稀释结果。

**3. 可插拔模块化架构**

基于抽象接口 + 工厂模式 + YAML 配置驱动，LLM、Embedding、Vector Store、Reranker 均可独立替换：

```yaml
llm:
  provider: openai   # 可换 azure / ollama / deepseek
embedding:
  provider: openai   # 可换 azure / local
reranker:
  provider: cross_encoder  # 可换 llm_reranker
```

**4. MCP 协议服务化**

将知识检索能力封装为 MCP 工具，外部 AI Agent 或研发工具可直接调用：

```python
# 外部 Agent 调用示例
result = mcp_client.call("query_knowledge_hub", {
    "query": "激光雷达的外参标定步骤",
    "collection": "ad_knowledge_v01"
})
# 返回：带引用的结构化回答
```

**5. 五阶段 Ingestion Pipeline**

```
Load → Split → Transform → Embed → Upsert
```

基于 SHA256 哈希增量去重，支持 PDF、图片（自动生成描述）、纯文本多格式，处理完成生成摄取报告。

## 查询示例

**传感器参数查询**
```
Q: 激光雷达 VLS-128 的水平视场角和角分辨率是多少？

A: VLS-128 激光雷达的水平视场角为 360°，垂直视场角为 ±15°，
   水平角分辨率为 0.1°（10Hz 模式）。[1]

[1] lidar_spec_vls128.pdf · 第 3 页 · 相关性 0.94
```

**传感器方案对比**
```
Q: 激光雷达和毫米波雷达在自动驾驶中各有什么优缺点？

A: ## 传感器方案对比

   ### 激光雷达 [1][2]
   - 探测距离：200m，角分辨率 0.1°
   - 优点：高精度三维点云，目标识别准确
   - 缺点：成本高（$5000+），受雨雪天气影响

   ### 毫米波雷达 [3]
   - 探测距离：150m，全天候工作
   - 优点：成本低，穿透雨雾能力强
   - 缺点：分辨率低，无法识别目标形状
```

**边界控制**
```
Q: 预测一下未来三年激光雷达的价格趋势

A: 本系统仅提供基于现有文档的事实性检索，不提供预测性分析。
   建议改为查询：当前激光雷达的技术规格和成本构成。
```

## 快速开始

```bash
# 1. 安装依赖
pip install -r requirements.txt

# 2. 配置 API Key
cp .env.example .env
# 填入 OPENAI_API_KEY

# 3. 摄取自动驾驶文档
python scripts/ingest.py \
  --collection ad_knowledge_v01 \
  --source demo-data-ad/ \
  --config config/settings.ad_knowledge.yaml

# 4. 查询
python scripts/query.py \
  --collection ad_knowledge_v01 \
  --query "激光雷达的外参标定步骤是什么？"

# 5. 启动 Dashboard
python scripts/start_dashboard.py
# 访问 http://localhost:8501
```

## 测试

```bash
pytest tests/unit/ -v      # 89 tests passing
pytest tests/integration/  # 集成测试
pytest tests/e2e/          # 端到端测试
```

## 项目结构

```
src/
├── core/
│   ├── query_engine/
│   │   ├── query_analyzer.py      # 查询分析 + AD 术语识别
│   │   ├── hybrid_search.py       # BM25 + Dense 混合检索
│   │   ├── fusion.py              # RRF 融合 + 文档分组
│   │   ├── metadata_booster.py    # 查询类型感知权重提升
│   │   └── scope_provider.py      # 知识库范围感知
│   └── response/
│       ├── response_builder.py    # 多文档综合 + 对比响应
│       └── citation_enhancer.py   # 引用元数据增强
├── ingestion/                     # 五阶段摄取流水线
├── libs/                          # 可插拔组件（LLM/Embedding/Reranker）
├── mcp_server/                    # MCP 协议服务
└── observability/                 # Dashboard + 评估 + Trace
```

## 技术栈

| 层次 | 技术 |
|------|------|
| 编排框架 | LangChain |
| 向量数据库 | Chroma |
| 稀疏检索 | BM25 (rank-bm25) |
| 重排序 | Cross-Encoder (sentence-transformers) |
| LLM | OpenAI GPT-4o-mini / Azure / Ollama |
| Embedding | OpenAI text-embedding-3-small |
| 服务协议 | MCP (Model Context Protocol) |
| 可视化 | Streamlit |
| 评估 | Ragas + 自定义指标 |

## License

MIT
