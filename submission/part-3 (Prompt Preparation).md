# Part 3: Prompt Preparation

---

## 3.1.1 Repository Context

MetaGPT is a Python framework that simulates a collaborative software company using Large Language Model (LLM) powered agents. The idea at its core is simple but powerful: instead of using a single AI to handle an entire software task, MetaGPT breaks the work into roles — Product Manager, Architect, Project Manager, Engineer, and QA Engineer — each of which is a separate autonomous agent. These agents communicate through a shared message bus and follow structured SOPs (Standard Operating Procedures) that mirror how real software teams operate. You give it a one-sentence requirement, and it produces user stories, architecture docs, data structures, APIs, and working code.

The intended users of MetaGPT span a wide range: AI researchers studying multi-agent cooperation, software developers who want to prototype projects quickly using natural language, and educators exploring how LLMs can model team collaboration. It lives in the broader problem domain of autonomous AI agents and natural language programming.

One of MetaGPT's most important subsystems is its RAG (Retrieval-Augmented Generation) module. This module enables agents to look up relevant information from a stored knowledge base — documents, code, specifications — rather than relying solely on what fits inside a single prompt. The RAG system uses `llama-index` under the hood to chunk documents, generate vector embeddings, store them in an index, and retrieve the most relevant chunks at query time. Because embedding generation involves calling external APIs (OpenAI, Gemini, etc.), efficiency in this subsystem directly affects both cost and response latency. Any bug that causes redundant embedding calls therefore has a real, measurable impact on production deployments of MetaGPT-based systems.

---

## 3.1.2 Pull Request Description

PR #1224 addresses a specific but consequential bug in the RAG module's `SimpleEngine` class. The problem manifests when a caller invokes `from_docs()` with one or more `retriever_configs` passed in. Internally, `from_docs()` delegates to `get_retriever()`, which was rebuilding the entire vector index — including re-running the embedding step — every time it was called, even when an index had already been built for the same set of documents during the same session.

The previous behaviour meant that if you initialised a `SimpleEngine` with documents and then requested multiple retrievers with different configurations, the embedding API would be called once per retriever config rather than once per unique set of documents. For large document sets or expensive embedding models, this translated directly into inflated API costs and unnecessary latency.

The new behaviour separates the concerns of **document transformation** (chunking and embedding) from **index construction and retrieval configuration**. Documents are embedded once, producing a set of `nodes` that are cached on the engine instance. Retriever construction then operates on those cached nodes, skipping the embedding step when nodes already exist. Additionally, the `ConfigBasedFactory.get_obj()` method was made more defensive by returning `None` for missing optional config keys instead of raising a `KeyError`, and the `RetrieverFactory` was refactored with decorators that implement "check-or-build" logic for index reuse. The example file `rag_pipeline.py` was also improved with docstrings and exception-handling decorators, making the RAG module easier to understand and more robust to surface-level errors.

---

## 3.1.3 Acceptance Criteria

The following criteria must all be satisfied for this PR to be considered successfully implemented:

1.**When `from_docs()` is called with one or more `retriever_configs`, the embedding model should be invoked exactly once for the set of input documents, regardless of how many retriever configurations are provided.**

2.**When `SimpleEngine.get_retriever()` is called on an instance that already has a built index, the system should reuse the existing index rather than rebuild it from raw documents.**

3.**When `ConfigBasedFactory._val_from_config_or_kwargs()` is called with a key that does not exist in the config or in kwargs, it should return `None` rather than raising a `KeyError`.**

4.**When `RetrieverFactory.create_retriever()` is called and no index yet exists for the given documents, it should build the index, store it internally, and return the retriever — subsequent calls with the same documents must not rebuild the index.**

5.**All existing unit tests in `tests/metagpt/rag/` must pass without modification to the test logic (only mock targets should change to reflect the new internal architecture).**

6.**The `rag_pipeline.py` example must execute end-to-end without errors, with docstrings on all public methods of `RAGExample` and exception handling that surfaces meaningful error messages rather than raw tracebacks.**

7.**When the `RAGExample` demonstrates adding documents and running a query, the flow must not trigger duplicate embedding calls as confirmed by mock call-count assertions in tests.**

---

## 3.1.4 Edge Cases

The following edge cases should be specifically considered during implementation:

**Edge Case 1 — Empty or Single-Document Input**  
`from_docs()` should handle being passed an empty list of documents or a list with exactly one document. The transformation pipeline must not crash on empty input, and the cached nodes list should remain an empty list in that scenario. Retriever creation from an empty index should raise a clear, informative error rather than silently returning a non-functional retriever.

**Edge Case 2 — Multiple Calls to `from_docs()` on the Same Engine Instance**  
If a developer calls `from_docs()` twice with different document sets on the same engine instance (e.g., incrementally adding documents), the implementation must clearly define and document the expected behaviour: does the second call replace the index, extend it, or raise an error? The bug fix must not silently serve stale nodes to the second call's retriever by over-aggressively caching.

**Edge Case 3 — `retriever_configs` with Conflicting Index Parameters**  
If two `retriever_config` objects in the same `from_docs()` call specify different similarity metrics or vector store backends, the implementation must decide whether to build separate indexes or raise a configuration error. The system should not silently discard one config's requirements in favour of the other.

**Edge Case 4 — Thread-Safety / Concurrent Retriever Requests**  
If two async tasks call `get_retriever()` on the same `SimpleEngine` instance simultaneously, the "check-or-build" logic must be race-condition-safe. Without a lock, both tasks could simultaneously determine that no index exists and each kick off an embedding job, recreating the duplicate-embedding problem under concurrent load.

---

## 3.1.5 Initial Prompt

```
NOTICE
Role: You are a professional Python engineer familiar with the llama-index library and
async design patterns. Your main goal is to write clean, well-typed, and testable code
that follows the existing style in the MetaGPT repository.
## Context
 
### Repository
MetaGPT (metagpt/) is a multi-agent LLM framework. Agents share a message bus and
follow SOPs. The RAG subsystem (metagpt/rag/) lets agents retrieve relevant chunks
from a document store using llama-index VectorStoreIndex and configurable retrievers.
 
### Bug being fixed  (PR #1224)
SimpleEngine.from_docs() calls get_retriever() once per retriever_config entry.
get_retriever() rebuilds the full VectorStoreIndex — including the embedding API call —
every time. Three configs = three embedding jobs on the same documents. This wastes
money and adds latency.
 
### Files to change
- metagpt/rag/engines/simple.py       ← main fix
- metagpt/rag/factories/base.py       ← return None instead of KeyError
- metagpt/rag/factories/retriever.py  ← check-or-build index guard
- examples/rag_pipeline.py            ← docstrings + async exception handling
- tests/metagpt/rag/engines/test_simple.py
- tests/metagpt/rag/factories/test_base.py
- tests/metagpt/rag/factories/test_retriever.py
 
## Task
 
1. Refactor SimpleEngine so documents are transformed (chunked + embedded) once into
   cached nodes via a new _from_nodes() classmethod. from_docs() stores these nodes;
   get_retriever() reads from them without re-running the embedding step.
 
2. In ConfigBasedFactory._val_from_config_or_kwargs(), change the missing-key behaviour
   from raising KeyError to returning None.
 
3. In RetrieverFactory, wrap create_retriever() with a check that reuses an existing
   index if one is already built for the current documents.
 
4. In examples/rag_pipeline.py, add Google-style docstrings to RAGExample and wrap
   async methods with an exception handler that logs errors cleanly.
 
5. Update all affected tests: remove VectorStoreIndex mocks, add a call-count assertion
   confirming the embed step fires exactly once regardless of how many retriever_configs
   are passed, and assert None (not KeyError) for missing config keys.
 
## Constraints
- Preserve the public API of SimpleEngine and the factory classes.
- If from_docs() is called with an empty document list, raise ValueError with a
  descriptive message — do not silently return a broken engine.
- If the embedding API fails mid-batch, do not cache partial nodes; leave the engine
  in a clean state so the caller can retry.
- Use asyncio.Lock inside SimpleEngine.__init__() to guard get_retriever() against
  concurrent async calls rebuilding the index simultaneously.
 
## Format example
 
## SimpleEngine patch
```python
# metagpt/rag/engines/simple.py  (full file)
class SimpleEngine(RetrieverQueryEngine):
    ...
    def __init__(self, ...):
        self._nodes = []
        self._index = None
        self._lock = asyncio.Lock()
        ...
 
    @classmethod
    def from_docs(cls, input_files, retriever_configs=None, ...):
        # 1. run transformation pipeline once
        # 2. cache nodes
        # 3. build retrievers from cached nodes
        ...
```
---
I declare that all written content in this assessment is my own work, created without the use of AI language models or automated writing tools. All technical analysis and documentation reflects my personal understanding and has been written in my own words.