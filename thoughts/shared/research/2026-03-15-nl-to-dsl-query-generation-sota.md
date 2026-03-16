---
date: 2026-03-15T17:30:00+00:00
researcher: claude
git_commit: ab1aa5631754aa69ffc14e57cdbf5c48a3f253d0
branch: claude/research-nl-hogql-agent-8xsVo
repository: jstammers/posthog
topic: 'NL to DSL Query Generation: Best Practices and State of the Art'
tags: [research, nl-to-sql, text-to-sql, dsl, hogql, llm, rag, grammar-constrained-decoding, query-generation]
status: complete
last_updated: 2026-03-15
last_updated_by: claude
---

# Research: NL to DSL Query Generation — Best Practices and State of the Art

**Date**: 2026-03-15T17:30:00+00:00
**Researcher**: claude
**Git Commit**: ab1aa5631754aa69ffc14e57cdbf5c48a3f253d0
**Branch**: claude/research-nl-hogql-agent-8xsVo
**Repository**: jstammers/posthog

## Research Question

What are best-practice and state-of-the-art methods for using LLMs to generate syntactically
correct and performant queries in domain-specific query languages that are functionally similar
to, but not standard SQL dialects — with specific relevance to HogQL?

## Summary

The field has matured rapidly. On standard SQL benchmarks (Spider), accuracy has nearly saturated
at ~86% with prompting alone. On harder real-world benchmarks (BIRD at 76%, Spider 2.0 at 24%),
significant gaps remain — gaps that are most pronounced for non-standard dialects.
For custom DSLs like HogQL, KQL, Cypher, and PromQL, the gap versus standard SQL is dramatic:
the PARROT benchmark shows LLMs average below 38.5% accuracy on cross-dialect SQL translation.

The current state of the art combines five major techniques:

1. **Dynamic schema-linked context** — inject only the relevant schema slice, enriched with
   descriptions and sample values, into the prompt
2. **Dual-similarity few-shot retrieval** — select example queries matching both NL question
   semantics and SQL/query structural skeleton
3. **Grammar-constrained output** — constrain the final generated tokens against a formal
   grammar (but allow free reasoning beforehand)
4. **Multi-candidate generation with execution-guided selection** — generate 3–5 candidates,
   execute all, select by result consistency
5. **Iterative execution-error refinement** — feed query engine error messages back to the LLM
   for up to 3 rounds of self-repair

For non-standard DSLs specifically, the highest-leverage additions are:
a grammar/function reference injected into the prompt (analogous to NL2KQL's Semantic Data
Catalog), synthetic data generation + LoRA fine-tuning on the target dialect, and applying
CRANE-style alternating reasoning/constraint to preserve chain-of-thought quality.

---

## Benchmarks and the Performance Landscape

### Standard SQL

| Benchmark   | Description                                        | SOTA Score                 | Human |
| ----------- | -------------------------------------------------- | -------------------------- | ----- |
| Spider 1.0  | 10K questions, 200 DBs, 138 domains                | ~86.6% EX (DAIL-SQL)       | ~91%  |
| BIRD        | 12.7K questions, 95 real-world DBs, 33.4 GB        | ~76.1% EX (Gemini 2.5 Pro) | ~93%  |
| Spider 2.0  | 632 enterprise workflows, 1,000+ column schemas    | ~23.8% (o3-mini)           | N/A   |
| BIRD-CRITIC | SQL debugging across MySQL/PostgreSQL/MSSQL/Oracle | ~34.5% (o3-mini)           | —     |

Spider 1.0 is largely saturated; the community now focuses on BIRD and Spider 2.0. The 60-point
drop from Spider to Spider 2.0 exposes how far research systems are from enterprise production
readiness (multi-step planning, 100+ line queries, reading external documentation).

### Custom DSLs and Dialect Translation

The **PARROT Benchmark** ([arXiv:2509.23338](https://arxiv.org/pdf/2509.23338)) measured
cross-system SQL translation accuracy (including Postgres→ClickHouse) and found LLMs average
below **38.53% accuracy** — roughly half the Spider score for the same model class.

Root causes of the DSL gap:

- **Training data scarcity**: LLMs see extensive standard SQL in pre-training;
  KQL, PromQL, Cypher, and HogQL appear rarely or never
- **Function name divergence**: ClickHouse functions (`toStartOfInterval`, `arrayJoin`),
  Cypher's `MATCH/RETURN` pattern, KQL's pipe syntax — none map to SQL patterns
- **Semantic paradigm shift**: graph pattern matching, time-range metrics, and event-property
  access require different reasoning from tabular JOIN logic
- **Schema specificity**: custom DSLs are deployed against instance-specific schemas;
  models hallucinate column/property names without schema context

No formal paper exists yet for NL-to-HogQL specifically. The closest analogs are NL2KQL (KQL
is also a custom analytics/log query language on top of a columnar engine) and dialect
fine-tuning work (Snowflake SQL, GoogleSQL).

---

## Technique 1: Schema-Aware Context Injection

Schema injection — how the database schema is represented in the prompt — is the single
highest-leverage prompt-engineering decision.

### Schema Serialization Format

**CREATE TABLE DDL format** ("SimpleDDL") is the dominant best practice:

- Mirrors SQL pre-training data (Stack Overflow, GitHub)
- Most token-efficient rich format
- Naturally includes column names, types, and constraints

Critical additions beyond bare DDL:

- **Foreign key hints**: explicitly state FK relationships; many systems omit these and suffer
- **Column descriptions**: business-language annotations for non-intuitive names (`col1`,
  abbreviations, domain jargon). NL2KQL uses a YAML Semantic Data Catalog with per-column
  type, enumerated values, and semantic description
- **Sample cell values**: including 2–5 representative values per string/enum column is the
  single most cost-effective technique for cross-domain generalization without fine-tuning.
  Enables value-level accuracy (knowing a column contains `'USD'` vs `'usd'`)
- **Dialect-specific annotations**: for non-SQL DSLs, add inline notes explaining syntax
  differences (e.g., for HogQL: "use `properties.$browser` to access event properties;
  `person` is a virtual table, not a physical JOIN")

**DFS serialization** for multi-table schemas preserves relational structure:
depth-first traversal of the schema graph improves schema routing by 1–6% over unordered
serialization ([DBCopilot, EDBT 2025](https://www.openproceedings.org/2025/conf/edbt/paper-209.pdf)).

### Schema Linking / Pruning (Two-Pass Approach)

For schemas with more than ~20 tables, feeding the full schema in every prompt degrades both
accuracy (noise) and cost. The CHESS system reports **5× token reduction** with only ~2%
accuracy drop using schema pruning.

**Two-pass pipeline**:

1. **Fast schema linker** (cheap LLM call or retrieval): select the top-k relevant tables
   and columns for the given question. Methods:
   - Zero-shot correlation scoring (C3)
   - Few-shot schema linking (DIN-SQL)
   - Preliminary-SQL-based filtering (PET-SQL, RSL-SQL)
   - LSH (locality-sensitive hashing) for value matching + vector DB for description
     similarity (CHESS)

2. **Main generation call**: feed only the linked schema slice

Schema linking is "becoming optional" ([arXiv:2408.07702](https://arxiv.org/html/2408.07702v2))
for frontier models when the schema fits in context, but remains essential for enterprise
databases with 100+ tables.

Key papers:

- **RSL-SQL** ([arXiv:2411.00073](https://arxiv.org/html/2411.00073v1)): bidirectional schema
  linking + contextual augmentation + binary candidate selection + self-correction; SOTA with
  GPT-4o and DeepSeek. Improves accuracy by up to 7%
- **CHESS** ([arXiv:2405.16755](https://arxiv.org/abs/2405.16755)): 4-agent pipeline with
  LSH-indexed cell value lookup + vector DB column descriptions. Schema linking recall 97%
  but 30% false positive rate

---

## Technique 2: Dynamic Few-Shot Retrieval

Static few-shot examples (hand-written once, fixed for all queries) consistently underperform
dynamic retrieval. For non-SQL DSLs, NL2KQL found that static few-shots can actively hurt
accuracy when examples don't match the target cluster's schema.

### Dual-Similarity Selection (DAIL-SQL)

**DAIL-SQL** ([arXiv:2308.15363](https://arxiv.org/abs/2308.15363), VLDB 2024) is the most
systematic study of few-shot strategy for query generation. Key finding: the best example
selection combines two signals:

1. **NL question similarity**: semantic embedding (cosine similarity over SBERT/ada-002
   embeddings)
2. **SQL skeleton similarity**: abstract out literals and identifiers → compare structural
   patterns (JOIN types, subquery nesting, aggregation presence)

This "dual-similarity" selection outperforms either signal alone because the LLM learns
mappings between question structure and query structure, not just surface phrasing.

DAIL-SQL also found that stripping schema context from in-context examples (presenting only
question-SQL pairs) cuts token cost significantly without hurting accuracy.

### Masked Question Similarity (OpenSearch-SQL)

**OpenSearch-SQL** ([arXiv:2502.14913](https://arxiv.org/abs/2502.14913), current BIRD SOTA
at 72.28%) introduced **Masked Question Similarity (MQs)**: mask entity tokens (table names,
column values) before computing NL similarity, so structural patterns dominate over surface
entities. This prevents retrieval from conflating questions about different tables that happen
to use similar entity names.

### Self-Taught CoT Augmentation

OpenSearch-SQL also introduced **self-taught chain-of-thought augmentation**: convert raw
NL-SQL pairs in the example library into NL-CoT-SQL triples by having an LLM add reasoning
traces, offline and once. This enriches few-shots with logical decomposition without requiring
human annotation.

### Historical Workload Retrieval (TailorSQL)

**TailorSQL** ([VLDB 2025](https://arxiv.org/html/2505.23039v1)) stores past executed queries
as "query hint documents" and retrieves them at inference time. Also synthesizes documents
capturing **common query patterns** across the historical workload. Results:

- +10–22% accuracy over baselines
- 2–15× smaller context windows than raw query libraries
- Pattern documents are more compact and more generalizable than verbatim past queries

This is the highest-ROI enhancement for analytics domains with repetitive query workloads
(like PostHog's analytics interface).

### Skeleton-Based Caching (ASKSQL)

**ASKSQL** ([ScienceDirect 2025](https://www.sciencedirect.com/science/article/pii/S2666827025000246))
extracts SQL structural skeletons (tables + clause types, not literals) for approximate cache
lookup. A question structurally similar to a past query reuses the cached LLM output.
Results: +5.83% accuracy and -32.6% latency as the cache grows.

### Diminishing Returns in Specialized Domains

RAG for EHR/specialized SQL ([arXiv:2403.09226](https://arxiv.org/html/2403.09226v1)) found
that **top-1 retrieved example** gives most of the gain; top-2 to top-5 show diminishing
returns or degradation. Entity masking (replacing domain terms with placeholders like `<EVENT>`)
before embedding improves retrieval quality in specialized domains.

---

## Technique 3: Grammar-Constrained Decoding

Grammar-constrained decoding (GCD) filters invalid tokens at each decoding step using a
formal grammar, guaranteeing syntactic validity from the first token. This eliminates an entire
class of errors with no additional LLM calls.

### Evolution of GCD

| System                                                                              | Year | Key Contribution                                                                           |
| ----------------------------------------------------------------------------------- | ---- | ------------------------------------------------------------------------------------------ |
| PICARD ([arXiv:2109.05093](https://arxiv.org/abs/2109.05093))                       | 2021 | Incremental SQL parser alongside T5; canonical reference                                   |
| Outlines ([Willard & Louf, 2023](https://arxiv.org/abs/2307.09702))                 | 2023 | Regex/CFG constrained decoding; integrates with Transformers/vLLM                          |
| GCD without Fine-tuning ([arXiv:2305.13971](https://arxiv.org/abs/2305.13971))      | 2023 | Input-dependent grammars where allowed tokens vary with schema                             |
| XGrammar                                                                            | 2024 | 100× faster grammar constraint preprocessing vs. prior libs                                |
| DOMINO                                                                              | 2024 | Near-zero overhead via speculative pre-computation; can increase throughput                |
| ITERGEN ([ICLR 2025](https://openreview.net/pdf?id=ac93gRzxxV))                     | 2025 | Grammar symbols as abstractions for forward/backward navigation; +18.5% over prior methods |
| Flexible and Efficient GCD ([ICML 2025](https://icml.cc/virtual/2025/poster/45613)) | 2025 | 17.71× faster offline preprocessing                                                        |

**Grammar-Aligned Decoding / ASAp** ([NeurIPS 2024](https://proceedings.neurips.cc/paper_files/paper/2024/file/2bdc2267c3d7d01523e2e17ac0a754f3-Paper-Conference.pdf)):
standard GCD distorts the LLM's token probability distribution (masking invalid tokens and
renormalizing). ASAp uses speculative sampling to preserve the model's conditional distribution
while guaranteeing grammaticality — better quality output at the same syntactic correctness.

**Practical tools**:

- **llama.cpp GBNF** ([README](https://github.com/ggml-org/llama.cpp/blob/master/grammars/README.md)):
  define a grammar in GGML Backus-Naur Form and pass at inference. JSON Schema→GBNF conversion
  is built in. For HogQL: derive a GBNF from HogQL's ANTLR grammar
- **Outlines**: Python library for Transformers/vLLM; supports regex, JSON Schema, CFG
- **2025 benchmark**: Guidance (8ms TPOT) > llama.cpp GBNF > XGrammar > Outlines (81ms TPOT)

**Grammar Prompting** (without constrained decoding):
**Grammar Prompting** ([Wang et al., NeurIPS 2023](https://proceedings.neurips.cc/paper_files/paper/2023/file/cd40d0d65bfebb894ccc9ea822b47fa8-Paper-Conference.pdf)):
augment each few-shot example with the minimal BNF grammar subset needed to produce it.
At inference, the model first predicts relevant grammar rules, then generates the query
constrained to those rules — in the prompt alone, without token masking. Tested on GeoQuery,
PDDL, and SMILES. The grammar context alone improves syntactic validity even without constrained
decoding hardware.

### CRANE: The Critical Insight for Complex Queries

**CRANE** ([arXiv:2502.09061](https://arxiv.org/abs/2502.09061), ICML/ICLR 2025) identifies
a critical failure mode of standard GCD: **strict grammar constraints during decoding suppress
chain-of-thought reasoning**, reducing functional correctness even when syntactic correctness
improves.

Solution: **alternate** between:

- Unconstrained generation phases (for reasoning/planning: "I need to join events to persons
  using the virtual table...")
- Constrained generation phases (for final output tokens: the actual query string)

Results: up to 10 percentage point improvement over both pure constrained and pure unconstrained
baselines on symbolic reasoning benchmarks.

**Implication for HogQL**: Do not apply grammar masks to the entire generation. Let the model
reason freely first (chain-of-thought), then emit the final query under constraint.

---

## Technique 4: Multi-Candidate Generation and Re-Ranking

Generate multiple candidate queries, execute them, and select the best by result consistency
or a trained selection model.

### Self-Consistency (Execution-Result Voting)

**SQL-PaLM** ([arXiv:2306.00739](https://arxiv.org/abs/2306.00739)): generate N candidates
at temperature > 0, execute all, group by result set, pick the majority cluster. Simple and
effective for ambiguous question intent. Ceiling: the most-consistent answer is not always
correct; upper bound is ~14% higher than self-consistency achieves.

### Multi-Path + Preference-Optimized Selection (CHASE-SQL)

**CHASE-SQL** ([ICLR 2025](https://proceedings.iclr.cc/paper_files/paper/2025/file/974ff7b5bf08dbf9400b5d599a39c77f-Paper-Conference.pdf)):

1. Generate candidates via multiple diverse strategies (divide-and-conquer decomposition,
   QDecomp, OS prompt)
2. Run iterative fix loops (up to 3 rounds using syntax errors and empty results as signals)
3. Use a trained **pairwise selection agent** (preference optimization / DPO) rather than
   majority voting

The selection agent is trained to distinguish "better" vs. "worse" SQL using pairwise examples
— moves beyond the limitation of voting-based selection.

### MCTS-SQL

**MCTS-SQL** ([arXiv:2501.16607](https://arxiv.org/pdf/2501.16607v3)): Monte Carlo tree search
for iterative refinement with prefix-cache mechanism to amortize repeated schema/prompt costs.

**Practical recommendation**: For a production system, generate 3–5 candidates using varied
few-shot examples or temperature, execute all, use execution result consistency as a fast filter.
Add an LLM re-ranker only when the consistency signal is ambiguous.

---

## Technique 5: Iterative Execution-Error Refinement

All top systems now incorporate execution feedback. The fundamental pattern:

```text
generate query
  → execute against engine
  → if error: feed error message to LLM + ask to fix
  → repeat up to N iterations
  → if empty result: flag for user review OR retry with alternative approach
```

### Systems Using Execution Refinement

- **DIN-SQL**: human-written self-correction guidelines applied post-generation
- **CHASE-SQL**: syntax errors + empty result sets as signals; up to 3 rounds
- **MAC-SQL Refiner**: execution errors as primary refinement signal in multi-agent pipeline
- **RSL-SQL**: multi-turn self-correction driven by execution outcomes
- **ReFoRCE** ([arXiv:2502.00675](https://arxiv.org/pdf/2502.00675)): bounded retry agent;
  80% of examples need zero refinement; weighted average 1.69 LLM calls and 15.4K tokens
- **MAGIC** ([AAAI 2025](https://ojs.aaai.org/index.php/AAAI/article/view/34511)): automatically
  generates self-correction guidelines tailored to the model's specific failure patterns from
  training data, rather than hand-written rules

**ExeSQL** ([EMNLP 2025](https://aclanthology.org/2025.findings-emnlp.1320.pdf)): execution-based
self-training with DPO — uses execution outcomes to build preference pairs for offline training,
not just prompt-time correction.

### Semantic Validation (Beyond Syntax)

Execution success is necessary but not sufficient. Semantic errors (structurally valid SQL
returning wrong data) are the hardest category.

Common semantic errors ([VLDB 2025 survey](https://dbgroup.cs.tsinghua.edu.cn/ligl/papers/VLDB25-NL2SQL.pdf)):

- Wrong table/column selection
- Missing or incorrect WHERE predicates
- Aggregation mismatches (GROUP BY without corresponding SELECT)
- Invalid JOIN conditions
- Misaligned outer/inner query attribute types in subqueries

**REDSQL** ([VLDB 2025](https://dl.acm.org/doi/10.14778/3734839.3734847)) is the strongest
current approach for semantic debugging. It validates the predicted SQL against actual database
content using defined constraints (type compatibility, NULL checks, column relationship checks),
generates a structured verification report, and feeds it back to the LLM for refinement.
Results: +8–11 percentage points on BIRD.

**Reverse translation**: ask a critic LLM "given this SQL, what question does it answer?",
then compare to the original question. Semantic divergence signals a wrong query. Used in
CHASE-SQL's selection agent.

**Multi-layer validation architecture** (practical pattern):

1. Fast syntax check via `sqlglot` or the target language's parser
2. Schema validation: verify every table/column reference exists
3. Type checking: verify operator/operand type compatibility
4. LLM semantic pass: check JOIN logic, aggregation correctness, filter completeness
5. Optional: reverse-translation consistency check

---

## Technique 6: Fine-Tuning and Reinforcement Learning

### When Fine-Tuning Wins Over Prompting

Fine-tuning a smaller model on the target dialect consistently outperforms zero-shot prompting
of a larger general model on dialect-specific tasks:

- **Snowflake SQL / GoogleSQL** ([arXiv:2312.02251](https://arxiv.org/html/2312.02251v1)):
  GPT-4 generates synthetic NL-SQL pairs for the target dialect → LoRA fine-tune Code-LLaMA /
  Mistral / StarCoder+. Result: Code-LLaMA 13B at 81.58% (Snowflake) / 82.66% (GoogleSQL),
  outperforming zero-shot GPT-4
- **Text2Cypher**: Neo4j generated 44K NL-Cypher training pairs; fine-tuned open-source models
  outperform GPT-4 on schema-grounded Cypher generation
- **SQLCoder** (Defog): 15B model (StarCoder base) fine-tuned on curated SQL; matches or beats
  GPT-4 on target schemas. Used in production under data privacy constraints

For any custom DSL: **synthetic data generation via GPT-4o + LoRA fine-tuning** is the most
cost-effective path to above-GPT-4 performance on the specific dialect.

### Reinforcement Learning (2025 Trend)

RL is replacing supervised fine-tuning as the preferred training strategy:

**Reasoning-SQL** ([arXiv:2503.23157](https://arxiv.org/abs/2503.23157)): applies GRPO
(Group Relative Policy Optimization, the same technique as DeepSeek-R1) with SQL-specific
partial rewards:

- Schema-linking reward
- n-gram similarity to reference SQL
- Syntax check reward
- AI feedback reward

These partial rewards overcome reward sparsity from binary execution accuracy alone. Key
results: the 14B RL-trained model beats o3-mini by 4% and Gemini-1.5-Pro by 3% on BIRD.
RL-only training (+6.77%) outperforms SFT (+4.11%); SFT alone can cause memorization that
hurts generalization.

**CSC-SQL** ([arXiv:2505.13271](https://arxiv.org/pdf/2505.13271)): combines self-consistency
with self-correction using GRPO, addressing the limitation that vanilla self-consistency can
select a wrong majority answer.

---

## DSL-Specific Research: Key Papers

### KQL (Kusto Query Language — Azure)

**NL2KQL** ([arXiv:2404.02933](https://arxiv.org/abs/2404.02933), KDD 2024, Microsoft):
the first formal NL-to-KQL system. KQL is architecturally the closest analog to HogQL:
custom analytics query language over a columnar engine (Azure Data Explorer vs ClickHouse),
pipe-based syntax, rich time-series functions, semi-structured log/telemetry data.

Pipeline:

1. **Semantic Data Catalog**: YAML-annotated schema with per-column type, enumerated values,
   and semantic description
2. **Schema Refiner**: selects relevant tables/columns for the question
3. **Few-shot Selector**: dynamic retrieval using cluster-specific NL-KQL example library
4. **Prompt Builder**: assembles system prompt + schema slice + few-shots
5. **Query Refiner**: post-hoc syntax/semantic correction via another LLM call

Key finding: generic few-shots hurt; examples must match the _specific deployment's schema_.

[github.com/microsoft/NL2KQL](https://github.com/microsoft/NL2KQL)

### Cypher (Neo4j)

**Text2Cypher** ([arXiv:2412.10064](https://arxiv.org/html/2412.10064v1)): 44,387 NL-Cypher
training pairs. Fine-tuned LLaMA and Codestral models outperform GPT-4 on schema-grounded
Cypher generation. Uses RLHF-style preference learning.

Key challenge: graph schema alignment — mapping NL entity mentions to specific node/edge types
without confabulating relationship types.

### SPARQL

**SPARQL-LLM** ([arXiv:2512.14277](https://arxiv.org/html/2512.14277v1)): RAG over schema
metadata + validated examples + post-generation validation/correction. Main failure mode:
hallucinated URIs when knowledge graph schema is not in training distribution.

### PromQL

**PromAssistant** ([arXiv:2503.03114](https://arxiv.org/abs/2503.03114), March 2025): first
text-to-PromQL benchmark (280 questions). Uses a knowledge graph representing service topology
(metrics, labels, relationships) for LLM+KG joint reasoning. Key challenge: NL label
descriptions do not map directly to Prometheus label keys due to naming conventions.

### Elasticsearch Query DSL

**NL2EQ** ([Springer CompCom 2025](https://link.springer.com/chapter/10.1007/978-3-031-92605-1_15)):
generates Elasticsearch Query DSL from NL. AWS also provides a prescriptive guidance pattern
for OpenSearch Query DSL.

---

## Task Decomposition Methods

Task decomposition addresses complex multi-step queries by breaking generation into structured
subtasks.

**DIN-SQL** ([arXiv:2304.11015](https://arxiv.org/abs/2304.11015), NeurIPS 2023):

1. Schema linking with few-shot examples
2. Query classification: EASY / NON-NESTED COMPLEX / NESTED COMPLEX
3. Difficulty-tailored SQL generation
4. Self-correction pass

**MAC-SQL** (2024): multi-agent decomposition where a manager agent decomposes the question,
specialist agents handle subqueries, and a refiner agent applies execution-guided correction.

**OmniSQL** ([arXiv:2503.02240](https://arxiv.org/html/2503.02240), 2025): generates CoT
reasoning traces as training targets using synthetic data. Fine-tunes on schema linking + SQL
revision + SQL selection as auxiliary tasks. Matches or outperforms DeepSeek-V3, Qwen2.5-72B,
GPT-4o at smaller model sizes.

---

## Relevance to PostHog's HogQL Implementation

Mapping research findings to the current HogQL generation architecture
(`ee/hogai/chat_agent/sql/`):

### What PostHog Currently Does

| Technique                 | Current Implementation                                                      |
| ------------------------- | --------------------------------------------------------------------------- |
| Schema injection          | `_serialize_database_schema()` in `HogQLDatabaseMixin`                      |
| DSL-specific instructions | `HOGQL_GENERATOR_SYSTEM_PROMPT` (925+ lines)                                |
| Function documentation    | `SQL_SUPPORTED_FUNCTIONS_DOCS`, `SQL_SUPPORTED_AGGREGATIONS_DOCS`           |
| Execution error feedback  | `_validate_hogql_query_sync()` → `PydanticOutputParserException` → retry    |
| Structured output         | `SQL_SCHEMA = {"query": "..."}` via function calling                        |
| Schema linking            | `QueryPlannerNode` with taxonomy discovery tools (property/event retrieval) |
| Few-shot examples         | `HOGQL_GENERATOR_SYSTEM_PROMPT` includes one static example                 |
| Core memory context       | Injected via `AssistantContextMixin`                                        |

### Techniques Not Yet Present

Based on the research, the following techniques are absent from the current implementation and
represent opportunities:

1. **Dynamic few-shot retrieval**: Only one static example is in the prompt. A library of
   validated NL-HogQL pairs with dual-similarity retrieval (DAIL-SQL method) would be a
   direct improvement. Historical workload retrieval (TailorSQL) would have the highest ROI
   for PostHog's analytics use case

2. **Grammar-constrained decoding**: HogQL has an ANTLR grammar
   (`posthog/hogql/grammar/`). Deriving a GBNF/Outlines grammar would eliminate syntactic
   errors before any validation step. However, the CRANE finding suggests this should be
   applied only to the final output tokens, not the full CoT

3. **Multi-candidate generation with selection**: Currently generates a single query with
   retries on failure. Generating 3 candidates and selecting by execution consistency would
   improve accuracy on ambiguous or complex questions

4. **Semantic validation (REDSQL pattern)**: Current validation is purely syntactic
   (parse + compile). Adding post-execution constraint checking (NULL handling, type
   compatibility, JOIN correctness) with structured feedback to the LLM would address
   queries that compile but return wrong results

5. **RL-based fine-tuning with SQL-specific partial rewards**: The Reasoning-SQL GRPO approach
   with HogQL-specific rewards (schema linking, syntax, AI feedback) would be the highest-lift
   long-term investment, applicable once a labeled NL-HogQL evaluation set exists

6. **Dialect-specific synthetic data + LoRA fine-tuning**: Use GPT-4o to generate NL-HogQL
   pairs at scale, then LoRA fine-tune a smaller model. This consistently outperforms
   zero-shot prompting of a larger model on custom dialects and would reduce inference cost

---

## Consolidated Best-Practice Recommendations for Custom DSL Query Generation

Ordered by estimated ROI for a system like HogQL:

### Tier 1: High ROI, Low Implementation Cost

1. **Value injection in schema**: Include 2–5 sample values per string/enum column in schema
   serialization. Most cost-effective single improvement for cross-schema generalization

2. **Query hint library**: Accumulate validated NL-query pairs from user interactions. At
   inference, retrieve top-1 to top-3 most similar past queries as few-shot examples.
   TailorSQL shows +10–22% accuracy. For HogQL, this leverages PostHog's existing user
   query history

3. **Execution-error refinement loop**: Feed query engine error messages back for up to 3
   rounds. The `_validate_hogql_query_sync()` mechanism already exists; extending it to
   pass the error string back to the LLM in a follow-up message is a small change

4. **Masked Question Similarity for example retrieval**: When building the few-shot library,
   mask entity tokens (event names, property names) before computing similarity to find
   structurally similar examples regardless of schema

### Tier 2: High ROI, Moderate Implementation Cost

5. **Two-pass schema linking**: Fast first pass to select relevant events/properties (already
   done by `QueryPlannerNode`), then a second pass to serialize only the selected slice into
   the SQL generator prompt. Currently the SQL generator receives the full schema

6. **Multi-candidate generation**: Generate 3 candidates with varied prompting strategies,
   execute all, select by consistency. Cap at 3 to control latency

7. **CRANE-style alternating generation**: Apply grammar constraints only to the final query
   output tokens; allow free chain-of-thought reasoning before the query block. Prevents the
   reasoning suppression observed with full GCD

8. **Semantic Data Catalog (NL2KQL pattern)**: Formalize the HogQL schema representation as
   a YAML catalog with per-property annotations (type, example values, semantic description)
   rather than generated-at-runtime schema strings

### Tier 3: High ROI, High Implementation Cost

9. **Grammar-constrained decoding**: Derive an Outlines/GBNF grammar from HogQL's ANTLR
   grammar. Eliminates all syntactic errors at generation time. Requires integrating a
   constrained decoding library into the inference stack

10. **Synthetic data + LoRA fine-tuning**: Use GPT-4o to generate 10K–50K NL-HogQL pairs
    across query patterns. LoRA fine-tune Code-LLaMA 13B or similar. Consistently outperforms
    zero-shot prompting of larger models on dialect-specific tasks once training data exists

11. **RL fine-tuning (GRPO) with HogQL-specific rewards**: Long-term highest-lift investment.
    Requires a labeled evaluation set of NL-HogQL pairs with execution ground truth

---

## Key Papers Reference

| Paper                | Venue     | Year | Key Technique                                            | Link                                                                                                                             |
| -------------------- | --------- | ---- | -------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| DIN-SQL              | NeurIPS   | 2023 | Task decomposition + self-correction                     | [arXiv:2304.11015](https://arxiv.org/abs/2304.11015)                                                                             |
| DAIL-SQL             | VLDB      | 2024 | Dual-similarity few-shot selection                       | [arXiv:2308.15363](https://arxiv.org/abs/2308.15363)                                                                             |
| CHESS                | Stanford  | 2024 | 4-agent RAG + LSH schema linking                         | [arXiv:2405.16755](https://arxiv.org/abs/2405.16755)                                                                             |
| NL2KQL               | KDD       | 2024 | Semantic catalog + dynamic few-shot for custom DSL       | [arXiv:2404.02933](https://arxiv.org/abs/2404.02933)                                                                             |
| RSL-SQL              | —         | 2024 | Robust schema linking + multi-turn correction            | [arXiv:2411.00073](https://arxiv.org/html/2411.00073v1)                                                                          |
| CHASE-SQL            | ICLR      | 2025 | Multi-path + preference-optimized selection              | [arXiv:2410.01943](https://arxiv.org/abs/2410.01943)                                                                             |
| Grammar Prompting    | NeurIPS   | 2023 | BNF grammar augmentation of few-shot prompts             | [arXiv:2305.19234](https://arxiv.org/pdf/2305.19234)                                                                             |
| CRANE                | ICML/ICLR | 2025 | Alternating unconstrained reasoning + constrained output | [arXiv:2502.09061](https://arxiv.org/abs/2502.09061)                                                                             |
| ASAp / GAD           | NeurIPS   | 2024 | Distribution-preserving constrained sampling             | [NeurIPS 2024](https://proceedings.neurips.cc/paper_files/paper/2024/file/2bdc2267c3d7d01523e2e17ac0a754f3-Paper-Conference.pdf) |
| REDSQL               | VLDB      | 2025 | Constraint-based semantic validation (+8–11% BIRD)       | [ACM](https://dl.acm.org/doi/10.14778/3734839.3734847)                                                                           |
| OpenSearch-SQL       | ACM       | 2025 | MQs + self-taught CoT; BIRD SOTA 72.28%                  | [arXiv:2502.14913](https://arxiv.org/abs/2502.14913)                                                                             |
| TailorSQL            | VLDB      | 2025 | Query hint documents from historical workloads           | [arXiv](https://arxiv.org/html/2505.23039v1)                                                                                     |
| Reasoning-SQL        | —         | 2025 | GRPO RL with SQL-specific partial rewards                | [arXiv:2503.23157](https://arxiv.org/abs/2503.23157)                                                                             |
| MAGIC                | AAAI      | 2025 | Auto-generated self-correction guidelines                | [AAAI](https://ojs.aaai.org/index.php/AAAI/article/view/34511)                                                                   |
| Text2Cypher          | —         | 2024 | Large-scale NL-Cypher dataset + fine-tuning              | [arXiv:2412.10064](https://arxiv.org/html/2412.10064v1)                                                                          |
| SPARQL-LLM           | —         | 2024 | RAG + schema metadata + validation for SPARQL            | [arXiv:2512.14277](https://arxiv.org/html/2512.14277v1)                                                                          |
| PromAssistant        | —         | 2025 | First text-to-PromQL benchmark + KG context              | [arXiv:2503.03114](https://arxiv.org/abs/2503.03114)                                                                             |
| PARROT               | —         | 2025 | Cross-dialect SQL translation benchmark (38.5%)          | [arXiv:2509.23338](https://arxiv.org/pdf/2509.23338)                                                                             |
| Spider 2.0           | ICLR      | 2025 | Enterprise SQL benchmark (o3-mini: 23.8%)                | [arXiv:2411.07763](https://arxiv.org/abs/2411.07763)                                                                             |
| Fine-tuning dialects | —         | 2023 | Synthetic GPT-4 data + LoRA for Snowflake/GoogleSQL      | [arXiv:2312.02251](https://arxiv.org/html/2312.02251v1)                                                                          |

## Resources

- [BIRD Benchmark](https://bird-bench.github.io/)
- [Spider 2.0](https://spider2-sql.github.io/)
- [BIRD-CRITIC](https://bird-critic.github.io/)
- [LiveSQLBench](https://livesqlbench.ai/)
- [NL2KQL GitHub](https://github.com/microsoft/NL2KQL)
- [DAIL-SQL GitHub](https://github.com/BeachWang/DAIL-SQL)
- [CHESS GitHub](https://github.com/ShayanTalaei/CHESS)
- [OpenSearch-SQL GitHub](https://github.com/OpenSearch-AI/OpenSearch-SQL)
- [Awesome-LLM-based-Text2SQL](https://github.com/DEEP-PolyU/Awesome-LLM-based-Text2SQL)
- [NL2SQL Handbook](https://github.com/HKUSTDial/NL2SQL_Handbook)
- [Awesome-LLM-Constrained-Decoding](https://github.com/Saibo-creator/Awesome-LLM-Constrained-Decoding)
- [Awesome-Text2SQL (including DSL)](https://github.com/eosphoros-ai/Awesome-Text2SQL)
- [VLDB 2025 NL2SQL Survey](https://dbgroup.cs.tsinghua.edu.cn/ligl/papers/VLDB25-NL2SQL.pdf)
- [TKDE 2025 Survey: Next-Generation Database Interfaces](https://arxiv.org/html/2406.08426v4)

## Related Research

- `thoughts/shared/research/2026-03-15-nl-hogql-agent.md` — prior research documenting
  the current HogQL agent implementation in this codebase
