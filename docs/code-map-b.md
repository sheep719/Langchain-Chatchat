# 检索链路代码地图（fork 改动视角）

> 本文是二开改动的代码导航：链路逐跳（已逐一验证文件真实存在）→ 三处扩展锚点 → 改动文件清单。
> 对应上游基线 `49165d6`，改动全部在 `feature/excel-loader-and-hybrid-search` 分支。

## 一、检索链路（WebUI 发问 → 返回排序结果）

```
WebUI (Streamlit, :8501)
  │  POST /chat/kb_chat
  ▼
api_server 路由层
  kb_chat.py :: kb_chat()                          [入口路由，mode=local_kb]
  │  await run_in_threadpool(search_docs, ...)     [同步检索丢线程池，不堵事件循环]
  ▼
knowledge_base 模块
  kb_doc_api.py :: search_docs()                   [API 封装]
  │  KBServiceFactory.get_service_by_name(kb_name) [按库名拿服务实例]
  ▼
  kb_service/base.py :: KBService.search_docs()    [抽象入口，L201]
  │  do_search() 返回 (Document, score) 元组时统一剥分数——上层只见 Document
  ▼
  kb_service/faiss_kb_service.py :: do_search()    [实际执行，L66]
  │  ★ 改动2挂载点：with self.load_vector_store().acquire() as vs:
  │      hybrid_search(vectorstore=vs, ..., kb_key=(kb_name, vector_name))
  │    （SEARCH_METHOD=vector 时函数内部直接 similarity_search_with_score，等价原行为）
  ▼
  knowledge_base/hybrid_search.py :: hybrid_search()  [★ 新增模块，263 行]
  │  vector / ensemble / hybrid_rrf / hybrid_rerank 四模式分发
  ▼
向量库 FAISS
  kb_cache/faiss_cache.py :: KBFaissPool.load_vector_store()
  │  磁盘: data/data/knowledge_base/<kb>/vector_store/<embed_model>/index.faiss
  ▼
返回排序结果 → format_reference() → LLM 生成回答
```

**逐跳验证记录**（2026-10-28 复核）：链路涉及的 7 个文件全部真实存在（kb_chat.py / kb_doc_api.py / base.py / faiss_kb_service.py / file_rag/utils.py / retrievers/ensemble.py / knowledge_base/utils.py）。

## 二、三处扩展锚点（改动为什么挂在这）

| # | 锚点 | 文件 :: 函数 | 本 fork 的用法 |
|---|---|---|---|
| 1 | 文件解析器注册点 | `knowledge_base/utils.py` :: `LOADER_DICT`（扩展名→Loader 类映射）→ `get_loader()`（L177） | `"OpenpyxlExcelLoader": [".xlsx", ".xls", ".xlsd"]` 替换原 UnstructuredExcelLoader 键；import 白名单 +1 行 |
| 2 | 文本分块器 | `knowledge_base/utils.py` :: `make_text_splitter()`（L222） | 未改动——行级 Document 天然短小，默认分块器无再切分压力 |
| 3 | 检索执行入口 | `kb_service/base.py :: search_docs()`（L201）→ `faiss_kb_service.py :: do_search()`（L66） | do_search 改为调 hybrid_search() 统一入口；vector 模式内部直走 FAISS（原路径零改动） |

## 三、改动文件全景（git diff master...HEAD，27 文件 +1736 行）

**功能代码（3 个核心文件）**：
- `file_rag/document_loaders/openpyxl_loader.py`（108 行，新增）——Excel 行级解析器
- `knowledge_base/hybrid_search.py`（263 行，新增）——BM25 缓存 / RRF 融合 / rerank / 四模式分发
- `knowledge_base/kb_service/faiss_kb_service.py`（+20/-2）——do_search 挂接 + do_add_doc 触发缓存失效
- `knowledge_base/utils.py`（+3/-1）——LOADER_DICT 注册 + import 白名单
- `settings.py`（+16）——SEARCH_METHOD / HYBRID_RECALL_K / BM25_RECALL_K / RRF_K / RERANK_CANDIDATE_K 配置项

**评测（evaluation/ 独立目录）**：
- `eval_retrieval.py`（104 行）——三模式离线评测，Hit@1/Hit@3/MRR
- `testset.json`（20 题）——改写问答对 + expected_doc + keywords
- `retrieval_eval_results.json`——原始结果（README 数字唯一来源）
- `unit_tests.py`（133 行）——RRF 手算比对 + 解析器边界
- `test_degradation.py`（54 行）——rerank 降级双路径实测
- `make_fixture.py` / `check_chunks.py`——测试数据生成 + 分块核验

**文档（docs/ 6 个）**：architecture-before/after、extension-points、eval-results、acceptance-log、retrieval-design-qa

**根目录**：二次开发笔记.md（改动台账）、部署说明.md、启动服务.bat

## 四、RRF 手算验证记录（2026-10-28）

构造两路排名：向量路 `[A,B,C]`、BM25 路 `[B,D,A]`，k=60：

| doc | 手算 | 函数输出 | 判定 |
|---|---|---|---|
| A | 1/61 + 1/63 = 0.032266 | 0.032266 | ✅ |
| B | 1/62 + 1/61 = 0.032522 | 0.032522 | ✅ |
| C | 1/63 = 0.015873 | 0.015873 | ✅ |
| D | 1/62 = 0.016129 | 0.016129 | ✅ |

融合排序 **B > A > D > C**，与函数输出完全一致。

**观察**：B 在两路都靠前（2、1 名）融合后登顶；A 两路都出现（1、3 名）稳居第二；D 只在 BM25 出现但排名靠前，反超只排向量第 3 的 C——**RRF 的"多路共识"与"单路强信号"的平衡就在这两个 1/61 与 1/63 的加法里**。
