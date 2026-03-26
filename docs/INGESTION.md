# Document Ingestion

## Overview

OpenDsStar provides a document ingestion layer that converts raw data files into structured descriptions stored in a vector database (Milvus). These descriptions enable the DS-Star agent to discover and understand data files at query time via semantic search.

The ingestion layer sits upstream of the agent: it runs once per corpus and produces a searchable index of file descriptions that the agent's tools can query during execution.

```
Raw Files  -->  Ingestion Pipeline  -->  Milvus Vector Store  -->  Agent (via search tool)
```

## Architecture

All ingestion implementations extend `DocumentDescriptionBuilder` (`src/ingestion/document_description_builder.py`), which defines the interface:

```python
class DocumentDescriptionBuilder(ABC):
    def process_directory(dir_path) -> (Milvus, analysis_results, document_streams)
    def process_corpus(corpus) -> (Milvus, analysis_results, document_streams)
```

There are two implementations, each using a fundamentally different strategy.

## Implementation 1: DoclingDescriptionBuilder (Primary)

**Location:** `src/ingestion/docling_based_ingestion/`

This is the primary, production-ready ingestion path. It uses a three-stage pipeline:

### Stage 1: Document Conversion (Docling)

The `DoclingConverter` (`docling_converter.py`) converts files to markdown using IBM's [Docling](https://github.com/DS4SD/docling) library. It handles 30+ file formats natively:

- **Docling-native:** PDF, DOCX, PPTX, XLSX, XML, HTML, CSV, JSON, YAML, code files, etc.
- **Parquet:** Converted to CSV first, then processed through Docling.
- **Text fallback:** Plain text, logs, markdown, RST files read directly with encoding detection.
- **Graceful degradation:** If Docling fails, falls back to raw text extraction.

```
File  -->  classify_suffix()  -->  Docling / Parquet-to-CSV / Text Fallback  -->  Markdown
```

### Stage 2: Markdown Shortening

The `MarkdownShortener` (`markdown_shortener.py`) compresses the markdown to fit within prompt limits:

- Truncates tables to `max_table_rows` (default: 50)
- Truncates lists to `max_list_items` (default: 100)
- Caps total content at `max_content_length` (default: 20,000 chars)
- Returns truncation statistics for downstream tracking

### Stage 3: LLM Description Generation

The `FileDescriptionGenerator` (`file_description_generator.py`) sends the shortened markdown to an LLM with a structured prompt (`prompts.py`) that produces retrieval-optimized descriptions with these sections:

- **Overview** -- purpose and use-cases
- **Document Type** -- format classification
- **Topics** -- main subjects covered
- **Key Terms** -- important entities and acronyms
- **Structured Data - Exact Column Names** -- verbatim column/field names
- **Structured Data - Column Descriptions** -- meaning, units, allowed values
- **Sampled rows/data** -- 2-3 representative examples
- **Likely Queries** -- what users might search for
- **Keywords** -- search terms and synonyms

The prompt explicitly instructs the LLM to copy column names verbatim and not hallucinate structure.

### Caching

Two-layer caching minimizes recomputation:

- **Analysis cache** (`DoclingAnalysisCache`): Caches the Docling-to-markdown conversion per file (keyed by path + modification time or content hash).
- **Description cache** (`FileDescriptionCache`): Caches LLM-generated descriptions per (doc_id, content_hash, model, prompt_template). Error descriptions are not cached and will be regenerated.

### Vector Store

Descriptions are stored in Milvus via `MilvusManager`, which handles:
- Deduplication (skips documents already in the collection by `doc_id`)
- Embedding via HuggingFace models (default: `ibm-granite/granite-embedding-english-r2`)

### Key Properties

- **Format coverage:** Best-in-class -- handles PDF, Office, structured data, code, etc.
- **Robustness:** Multiple fallback paths; Docling failure does not block ingestion.
- **Caching:** Extensive; re-running on the same corpus is nearly free.
- **Description quality:** Structured, retrieval-optimized, includes exact column names.
- **Batched LLM calls:** Configurable batch size for throughput.
- **Weakness:** Column names and schema details are LLM-inferred from markdown, not programmatically extracted from the actual data.

## Implementation 2: AnalyzerDescriptionBuilder (Code-Based)

**Location:** `src/ingestion/analyzer.py` + `src/agents/analyzer/`

This implementation uses a code-generation agent to programmatically analyze files -- closer to the approach described in the original DS-Star paper by Google.

### How It Works

The `AnalyzerGraph` (`src/agents/analyzer/analyzer_graph.py`) is a small LangGraph agent:

```
CoderNode  -->  ExecutorNode  -->  [DebuggerNode --> ExecutorNode]  -->  FinalizerNode
```

1. **CoderNode** generates Python code to load and describe a file (column names, types, samples, structure).
2. **ExecutorNode** runs the code in the same sandboxed environment used by DS-Star.
3. **DebuggerNode** fixes errors and retries (up to `max_debug_tries`, default: 3).
4. **FinalizerNode** produces the final description from execution output.

The generated code has direct filesystem access to the target file and can use pandas, numpy, etc. to extract exact schema information.

### Caching

The `AnalyzerDescriptionCache` (`src/ingestion/docling_cache.py`) caches final description results per file. Enable it by passing `cache_dir` to the builder:

```python
builder = AnalyzerDescriptionBuilder(
    llm="watsonx/mistralai/mistral-medium-2505",
    cache_dir="./cache",
    enable_caching=True,
)
```

Cache behavior:
- **Cache key:** File path + modification time (mtime). Modifying a file automatically invalidates its cache entry.
- **Cache directory:** Namespaced by a hash of `llm_model`, `code_timeout`, and `max_debug_tries`. Changing any of these parameters produces a separate cache, so results from different configurations never collide.
- **Only successful results are cached.** Failed analyses are always re-run on the next invocation, giving the agent another chance to generate correct code.
- **Storage:** Uses `diskcache` via the shared `FileCache` utility (same infrastructure as the Docling caches).

### Key Properties

- **Schema accuracy:** High -- code reads actual column names, data types, value ranges.
- **Adaptive:** The LLM generates file-type-specific code (different for CSV vs JSON vs Excel).
- **Self-healing:** Debug loop handles format surprises.
- **Caching:** File-level description cache keyed by path + mtime + config hash; re-running on the same corpus is fast.
- **Weakness:** Less robust for non-tabular formats (PDF, DOCX, presentations).

## Comparison with the Original DS-Star Paper

The DS-Star paper (Nam et al., arXiv:2509.21825) describes a **Data File Analyzer** as a core component. The paper's ablation study shows it is the single most impactful component: removing it drops accuracy on hard tasks by 18 percentage points.

| Dimension | Paper's Analyzer | AnalyzerDescriptionBuilder | DoclingDescriptionBuilder |
|-----------|-----------------|---------------------------|--------------------------|
| **Method** | Generate & execute Python per file | Generate & execute Python per file | Docling conversion + LLM summarization |
| **Data access** | Direct programmatic | Direct programmatic | Indirect (Docling markdown) |
| **Schema accuracy** | High (code-extracted) | High (code-extracted) | Medium (LLM-inferred from markdown) |
| **Format coverage** | Depends on generated code | Depends on generated code | 30+ formats via Docling |
| **Robustness** | Debug loop | Debug loop (3 retries) | Multi-fallback chain |
| **Caching** | Not described | File-level (path + mtime + config hash) | Two-layer (analysis + description) |
| **Output structure** | Raw printed text | Raw printed text | Structured sections with keywords |
| **Retrieval optimization** | Not described | Stored in Milvus | Retrieval-optimized descriptions in Milvus |

### Critical Architectural Difference

In the paper, file descriptions are injected **directly into every agent prompt** (Planner, Coder, Debugger, Verifier, Router) as ambient context. In OpenDsStar, descriptions live in Milvus and are accessed via a search tool at the agent's discretion.

**Implications:**

- The paper's approach guarantees the Planner always sees all data context.
- OpenDsStar's approach scales better (no prompt bloat with many files) but depends on the agent searching effectively.
- The paper uses an embedding-based retrieval module when N > 100 files, selecting top-K most relevant -- a middle ground.

## Usage

### Running DoclingDescriptionBuilder Standalone

```python
from pathlib import Path
from ingestion.docling_based_ingestion.docling_description_builder import (
    DoclingDescriptionBuilder,
)

builder = DoclingDescriptionBuilder(
    cache_dir="./cache",
    model="watsonx/mistralai/mistral-medium-2505",  # or any LiteLLM model
    temperature=0.0,
    embedding_model="ibm-granite/granite-embedding-english-r2",
    batch_size=8,
    enable_caching=True,
)

# Process a directory of files
vector_db, analysis_results, path_to_bytes_factory = builder.process_directory(
    Path("./data/my_corpus")
)

# Or process a corpus (list of Document objects from experiments framework)
vector_db, analysis_results, path_to_bytes_factory = builder.process_corpus(corpus)
```

**Returns:**
- `vector_db` -- Milvus instance with embedded descriptions, ready for similarity search.
- `analysis_results` -- dict mapping `doc_id` to metadata (success, description, truncation stats, file path).
- `path_to_bytes_factory` -- dict mapping file paths to callables that return file bytes (used by file content tools).

### Running AnalyzerDescriptionBuilder Standalone

```python
from pathlib import Path
from ingestion.analyzer import AnalyzerDescriptionBuilder

builder = AnalyzerDescriptionBuilder(
    llm="watsonx/mistralai/mistral-medium-2505",  # or a BaseChatModel instance
    code_timeout=30,
    max_debug_tries=3,
    embedding_model="ibm-granite/granite-embedding-english-r2",
    db_uri="./milvus_analyzer.db",
    cache_dir="./cache",       # enables caching (omit or None to disable)
    enable_caching=True,        # default: True (requires cache_dir)
)

# Process a directory
vector_db, analysis_results, document_streams = builder.process_directory(
    Path("./data/my_corpus")
)

# Or process a list of file paths
vector_db, analysis_results, document_streams = builder.process_corpus(
    [Path("./data/file1.csv"), Path("./data/file2.json")]
)
```

On the first run the analyzer graph executes for each file. On subsequent runs, cached results are returned immediately for unchanged files. The cache is keyed by `(file_path, mtime)` and namespaced by `(llm_model, code_timeout, max_debug_tries)`, so switching models or parameters starts a fresh cache.

### Wiring Ingestion Output into Agent Tools

The ingestion output connects to the agent through four shared tools defined in `src/experiments/benchmarks/shared_tools.py`. The `build_file_tools()` function creates all four from the ingestion output:

```python
from experiments.benchmarks.shared_tools import build_file_tools

# vector_db and path_to_bytes_factory come from the builder above
tools = build_file_tools(
    vector_db=vector_db,
    path_to_bytes_factory=path_to_bytes_factory,
)

# tools is a list of 4 LangChain BaseTool instances, ready to pass to the agent
```

The four tools produced:

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `search_files` | Semantic search over file descriptions | `query: str, top_k: int` | List of `{path, filename, description}` |
| `get_file_info` | Lightweight metadata for a file path | `path: str` | `{path, filename, extension, likely_format, is_tabular}` |
| `get_file_content` | Raw file bytes as a BytesIO stream | `path: str` | `BinaryIO` |
| `load_dataframe` | Load tabular file into pandas DataFrame | `path: str, format: str` | `pd.DataFrame` |

**Typical agent workflow:**
1. Agent calls `search_files(query)` to find relevant files.
2. Agent reads descriptions to choose the right file(s).
3. Agent calls `load_dataframe(path)` for tabular data, or `get_file_content(path)` for raw access.
4. Agent processes the data in generated code.

### Using Ingestion in Experiments

Both benchmark implementations (KramaBench, DataBench) wire ingestion automatically via their `ToolsBuilder`. The pattern is identical in both:

```python
# Inside a ToolsBuilder.build_tools() method:
builder = DoclingDescriptionBuilder(
    cache_dir=self.cache_dir,
    model=self.llm,
    temperature=self.temperature,
    embedding_model=self.embedding_model,
    batch_size=self.batch_size,
    enable_caching=True,
)
vector_db, analysis_results, path_to_bytes_factory = builder.process_corpus(corpus)
tools = build_file_tools(vector_db=vector_db, path_to_bytes_factory=path_to_bytes_factory)
```

See `src/experiments/benchmarks/kramabench/tools_builder.py` and `src/experiments/benchmarks/databench/tools_builder.py` for the full implementations.

### End-to-End Example: Custom Corpus with DS-Star Agent

```python
from pathlib import Path
from agents.ds_star.open_ds_star_agent import OpenDsStarAgent
from agents.utils.model_builder import ModelBuilder
from ingestion.docling_based_ingestion.docling_description_builder import (
    DoclingDescriptionBuilder,
)
from experiments.benchmarks.shared_tools import build_file_tools

# Step 1: Ingest the corpus
builder = DoclingDescriptionBuilder(
    cache_dir="./cache",
    model="watsonx/mistralai/mistral-medium-2505",
    embedding_model="ibm-granite/granite-embedding-english-r2",
)
vector_db, analysis_results, path_to_bytes_factory = builder.process_directory(
    Path("./data/my_corpus")
)

# Step 2: Build tools from ingestion output
tools = build_file_tools(
    vector_db=vector_db,
    path_to_bytes_factory=path_to_bytes_factory,
)

# Step 3: Create the agent with the tools
model, _ = ModelBuilder.build("gpt-4o-mini", temperature=0.0)
agent = OpenDsStarAgent(
    model=model,
    tools=tools,
    max_steps=10,
    code_timeout=60,
)

# Step 4: Query
result = agent.invoke("What are the top 5 customers by revenue?")
print(result["answer"])
```

## When to Use Which

| Scenario | Recommendation |
|----------|---------------|
| Mixed corpus (PDF, DOCX, CSV, JSON, code) | `DoclingDescriptionBuilder` -- best format coverage |
| Primarily structured data (CSV, parquet, JSON) | Consider `AnalyzerDescriptionBuilder` for higher schema fidelity |
| Re-running on same corpus frequently | Either -- both support caching when `cache_dir` is provided |
| Need exact column names for downstream code | `AnalyzerDescriptionBuilder` -- programmatic extraction |

## Code References

- `src/ingestion/document_description_builder.py` -- abstract interface
- `src/ingestion/docling_based_ingestion/docling_description_builder.py` -- primary builder
- `src/ingestion/docling_based_ingestion/docling_converter.py` -- file-to-markdown conversion
- `src/ingestion/docling_based_ingestion/file_description_generator.py` -- LLM description generation
- `src/ingestion/docling_based_ingestion/prompts.py` -- description generation prompt
- `src/ingestion/docling_based_ingestion/markdown_shortener.py` -- markdown truncation
- `src/ingestion/docling_based_ingestion/milvus_manager.py` -- vector store management
- `src/ingestion/analyzer.py` -- code-based builder
- `src/agents/analyzer/analyzer_graph.py` -- analyzer agent graph
- `src/agents/analyzer/nodes/coder.py` -- code generation for file analysis
- `src/ingestion/docling_cache.py` -- caching utilities
