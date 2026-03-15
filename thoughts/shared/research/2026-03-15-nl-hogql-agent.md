---
date: 2026-03-15T17:09:10+00:00
researcher: claude
git_commit: 3b6c944bcc2d9b726032a53249e9a2266bc5bac4
branch: claude/research-nl-hogql-agent-8xsVo
repository: jstammers/posthog
topic: 'Natural Language to HogQL Agent Implementation'
tags: [research, codebase, hogql, ai-agent, hogai, langraph, insights]
status: complete
last_updated: 2026-03-15
last_updated_by: claude
---

# Research: Natural Language to HogQL Agent Implementation

**Date**: 2026-03-15T17:09:10+00:00
**Researcher**: claude
**Git Commit**: 3b6c944bcc2d9b726032a53249e9a2266bc5bac4
**Branch**: claude/research-nl-hogql-agent-8xsVo
**Repository**: jstammers/posthog

## Research Question

How is the natural language to HogQL agent capability currently implemented in the PostHog codebase?

## Summary

PostHog has two distinct implementations for natural language to HogQL conversion:

1. **Legacy `posthog/hogql/ai.py`** — a standalone function (`write_sql_from_prompt`) that directly calls OpenAI (GPT-4.1-mini) with a schema-aware prompt, validates the output by parsing the HogQL AST, and retries up to 3 times on errors.

2. **Modern HogAI chat agent system** (`ee/hogai/`) — a multi-node LangGraph-based architecture that orchestrates taxonomy discovery, query planning, insight-type-specific generation (Trends, Funnel, Retention, SQL), and query execution. The SQL/HogQL path within this system goes through `SQLGeneratorNode` with comprehensive prompts, database schema serialization, and iterative validation.

The modern system is the primary path for the AI assistant ("Max") exposed to users. The legacy `posthog/hogql/ai.py` remains for direct HogQL-from-prompt use cases (e.g., data warehouse integrations).

---

## Detailed Findings

### Component 1: Legacy HogQL AI (`posthog/hogql/ai.py`)

**Location**: `posthog/hogql/ai.py:139-234`

**`write_sql_from_prompt(prompt, current_query, team, user)`**:

- Accepts a natural language prompt and optional current HogQL query to modify
- Builds a 3-message chain: identity + schema context → (optional current query) → user request
- Calls `hit_openai()` (`posthog/hogql/ai.py:237-255`) using model `gpt-4.1-mini` at temperature 0
- Validates the output by parsing and compiling the HogQL AST against the real database schema
- On validation failure: re-prompts the LLM with the error message (up to 3 retries, `posthog/hogql/ai.py:189-210`)
- Reports user actions via `report_user_action()` with token metrics

**System Prompts**:

- `IDENTITY_MESSAGE` (`posthog/hogql/ai.py:31-44`): HogQL expert role, key differences from standard SQL
- `HOGQL_EXAMPLE_MESSAGE` (`posthog/hogql/ai.py:45-62`): Weekly-active-users example query
- `SCHEMA_MESSAGE` (`posthog/hogql/ai.py:64-122`): Dynamic schema generated from team's actual database, includes critical person_id JOIN limitation warning
- `REQUEST_MESSAGE` (`posthog/hogql/ai.py:128-132`): User request template; if LLM responds with `UNCLEAR_PREFIX`, raises `PromptUnclear`

---

### Component 2: HogAI Chat Agent — Insights Graph (`ee/hogai/chat_agent/insights_graph/graph.py`)

This is the primary pipeline invoked by the AI assistant ("Max") when a user asks a data question.

**`InsightsGraph`** class with `graph_name = AssistantGraphName.INSIGHTS`:

**Full pipeline**:

```text
START
  ↓
INSIGHT_RAG_CONTEXT  (retrieves available actions for context)
  ↓
QUERY_PLANNER ↔ QUERY_PLANNER_TOOLS  (decides insight type + builds filter plan)
  ↓ routes to one of:
  ├── TRENDS_GENERATOR ↔ TRENDS_GENERATOR_TOOLS
  ├── FUNNEL_GENERATOR ↔ FUNNEL_GENERATOR_TOOLS
  ├── RETENTION_GENERATOR ↔ RETENTION_GENERATOR_TOOLS
  └── SQL_GENERATOR ↔ SQL_GENERATOR_TOOLS
  ↓
QUERY_EXECUTOR
  ↓
END
```

**Conditional routing** from `QueryPlannerToolsNode.router()` (`ee/hogai/chat_agent/query_planner/nodes.py:297-304`):

- Returns `"trends"`, `"funnel"`, `"retention"`, or `"sql"` based on the plan
- Returns `"continue"` to loop back for more planning
- Returns `"end"` when human-in-the-loop requested

---

### Component 3: Query Planner (`ee/hogai/chat_agent/query_planner/nodes.py`)

**`QueryPlannerNode`** (lines 58-239):

- **Model**: `o4-mini` with reasoning enabled (`auto` summary)
- **Tool choice**: `required`, `parallel_tool_calls=False`
- **8 tools bound**: `retrieve_event_properties`, `retrieve_action_properties`, entity-property tool (dynamic per team), event/action property values tools, `ask_user_for_help`, `final_answer`
- Constructs conversation prompt with schema and prior message history
- On `final_answer` tool call: returns plan, insight type; transitions to generator
- On `ask_user_for_help`: returns reset state (human-in-the-loop)
- **Max iterations**: 16 (`QueryPlannerToolsNode`, line 243)

**Key prompt**: `QUERY_PLANNER_STATIC_SYSTEM_PROMPT` (`ee/hogai/chat_agent/query_planner/prompts.py:1-97`)

- Explains when to choose Trends vs. Funnel vs. Retention vs. SQL
- Defines plan output format: Logic, Sources, Query kind, Tradeoffs
- Provides schema placeholders for each insight type's JSON schema

---

### Component 4: Taxonomy Discovery (`ee/hogai/chat_agent/taxonomy/`)

Used by the query planner to discover events, properties, and their values.

**`TaxonomyAgent`** (`ee/hogai/chat_agent/taxonomy/agent.py:15-71`):

- Builds a `START → LOOP_NODE ↔ TOOLS_NODE → END` LangGraph
- Iterates until `final_answer` or `ask_user_for_help` tool called, or max 10 iterations

**`TaxonomyAgentNode`** (`ee/hogai/chat_agent/taxonomy/nodes.py:42-158`):

- **Model**: `gpt-4.1` at temperature 0.3 with `parallel_tool_calls=True`
- Constructs prompt from system prompts + human input + tool progress messages
- Key system prompts:
  - `PROPERTY_TYPES_PROMPT` — distinguishes person/session/group vs. event properties, maps to correct tools
  - `TAXONOMY_TOOL_USAGE_PROMPT` — rules: never mix entity/event tools, never repeat same call, use parallel calls
  - `HUMAN_IN_THE_LOOP_PROMPT` — when to ask user for clarification

**`TaxonomyAgentToolkit`** (`ee/hogai/chat_agent/taxonomy/toolkit.py:140-871`):

- **MAX_ENTITIES_PER_BATCH**: 6 (line 146)
- **MAX_PROPERTIES**: 500 (line 147)
- Provides batched, parallel execution of taxonomy queries:
  - `retrieve_event_or_action_properties_parallel()` — multi-event batch
  - `retrieve_entity_properties_parallel()` — multi-entity batch
  - `retrieve_event_or_action_property_values()` — multi-property batch
  - `retrieve_entity_property_values()` — multi-entity multi-property batch
- Backed by `EventTaxonomyQueryRunner` and `ActorsPropertyTaxonomyQueryRunner`
- Uses `RECENT_CACHE_CALCULATE_ASYNC_IF_STALE_AND_BLOCKING_ON_MISS` execution strategy

---

### Component 5: SQL/HogQL Generator (`ee/hogai/chat_agent/sql/`)

The leaf node that generates the actual HogQL query when the query planner selects the `"sql"` path.

**`SQLGeneratorNode`** (`ee/hogai/chat_agent/sql/nodes.py:14-23`):

- `INSIGHT_NAME = "SQL"`
- `OUTPUT_MODEL = SQLSchemaGeneratorOutput`
- `OUTPUT_SCHEMA = SQL_SCHEMA` — JSON schema constraining LLM output to `{"query": "<string>"}`
- Calls `_run_with_prompt()` from `SchemaGeneratorNode`

**`HogQLGeneratorMixin._construct_system_prompt()`** (`ee/hogai/chat_agent/sql/mixins.py:128-147`):

- Concurrently gathers:
  1. Database schema via `_serialize_database_schema()` (uses `HogQLDatabaseMixin`)
  2. Core memory via `AssistantContextMixin`
- Returns `ChatPromptTemplate` with Mustache formatting injecting:
  - `{{{sql_expressions_docs}}}`, `{{{sql_supported_functions_docs}}}`, `{{{sql_supported_aggregations_docs}}}`
  - `{{{schema_description}}}` — team's actual database schema
  - `{{{core_memory}}}` — persistent agent memory

**`HOGQL_GENERATOR_SYSTEM_PROMPT`** (`ee/hogai/chat_agent/sql/prompts.py:1-149`):

- camelCase function name requirements
- JSON property access syntax (`properties.foo.bar`)
- Window function alternatives (`lagInFrame`/`leadInFrame`)
- Array null-guarding
- JOIN constraint restrictions (no relational operators in `ON` clause)
- `person_id` join limitation + required workarounds (subqueries, `WHERE IN`)
- Variable/placeholder syntax (`{variables.name}` with `coalesce()`/`IS NULL`)
- Example query template (lines 121-139)

**Validation** (`HogQLOutputParserMixin._validate_hogql_query_sync()`, `ee/hogai/chat_agent/sql/mixins.py:73-112`):

- Strips whitespace and trailing semicolons
- Parses with `parse_select()` from HogQL parser
- Handles filter/non-filter placeholders
- Compiles with ClickHouse dialect
- Catches ANTLR errors, converts "no viable alternative" to user-friendly messages
- Raises `PydanticOutputParserException` on failure (triggers `SchemaGeneratorNode` retry logic)

---

### Component 6: Supporting Documentation (`ee/hogai/chat_agent/sql/prompts.py`)

Beyond the main prompt, the file contains extensive inline documentation fed to the LLM:

- **`SQL_EXPRESSIONS_DOCS`** (lines 152-266): Property access syntax, data types, operators, common patterns
- **`SQL_SUPPORTED_FUNCTIONS_DOCS`** (lines 269-811): ~80+ enabled ClickHouse functions organized by category (type conversion, arithmetic, arrays, strings, dates, JSON, geo, maps, tuples, etc.)
- **`SQL_SUPPORTED_AGGREGATIONS_DOCS`** (lines 814-925): Standard and ClickHouse-specific aggregate functions

---

### Component 7: Main Assistant Graph (`ee/hogai/chat_agent/graph.py`)

**`AssistantGraph`** orchestrates the full conversational loop:

```text
START
  ↓
TITLE_GENERATOR
  ↓
SLASH_COMMAND_HANDLER
  ↓
MEMORY_ONBOARDING (multi-step flow with interrupts)
  ↓
MEMORY_COLLECTOR ↔ MEMORY_COLLECTOR_TOOLS
  ↓
ROOT (main agent loop + tools)
  ↓
END
```

The `ROOT` node invokes the `InsightsGraph` (and other subgraphs) when the user asks for analytics queries.

---

### Component 8: Query Executor (`ee/hogai/chat_agent/query_executor/nodes.py`)

**`QueryExecutorNode.arun()`** (lines 16-54):

- Extracts the artifact (generated query) from state
- Creates `InsightContext` and executes the query
- Handles `MaxToolRetryableError` gracefully
- Returns `AssistantToolCallMessage` with formatted results
- Clears planning state: `root_tool_call_id`, `root_tool_insight_plan`, `root_tool_insight_type`, `rag_context`

---

### Component 9: Agent Modes (`ee/hogai/chat_agent/mode_manager.py`)

**`ChatAgentModeManager`** manages which capabilities are active:

| Mode                | Description                            |
| ------------------- | -------------------------------------- |
| `PRODUCT_ANALYTICS` | Standard product analytics queries     |
| `SQL`               | Direct HogQL/SQL generation            |
| `SESSION_REPLAY`    | Session replay analysis                |
| `ERROR_TRACKING`    | Error tracking investigation           |
| `FLAGS`             | Feature flags management               |
| `SURVEY`            | Survey analysis                        |
| `LLM_ANALYTICS`     | LLM usage analytics                    |
| `PLAN`              | Plan mode for complex multi-step tasks |
| `EXECUTION`         | Execution after plan approval          |
| `ONBOARDING`        | Initial onboarding mode                |

Mode determines which `toolkit_class`, `prompt_builder_class`, and available tools are active.

---

### Component 10: API Layer (`ee/api/conversation.py`, `ee/hogai/api/serializers.py`)

**`ConversationViewSet.create()`** (`ee/api/conversation.py:304-452`):

- Primary endpoint for initiating AI conversations
- Rate-limited (different limits for `DEEP_RESEARCH` vs standard)
- Quota-checked via `is_team_limited()`
- Creates `ChatAgentWorkflow` or `ResearchAgentWorkflow`
- Returns `StreamingHttpResponse` with Server-Sent Events (SSE)

**`ConversationSerializer`** (`ee/hogai/api/serializers.py:53-206`):

- Serializes conversation state, messages, agent mode, pending approvals
- Retrieves enriched messages from LangGraph checkpoint state

---

## Code References

- `posthog/hogql/ai.py:139` — `write_sql_from_prompt()` legacy NL→HogQL function
- `posthog/hogql/ai.py:31-44` — `IDENTITY_MESSAGE` system prompt for legacy path
- `posthog/hogql/ai.py:64-122` — `SCHEMA_MESSAGE` with dynamic team schema
- `ee/hogai/chat_agent/insights_graph/graph.py:154` — `InsightsGraph.compile_full_graph()`
- `ee/hogai/chat_agent/query_planner/nodes.py:58` — `QueryPlannerNode` class definition
- `ee/hogai/chat_agent/query_planner/nodes.py:141-173` — `_get_model()` with o4-mini + reasoning
- `ee/hogai/chat_agent/query_planner/nodes.py:297-304` — `router()` deciding insight type
- `ee/hogai/chat_agent/query_planner/prompts.py:1-97` — `QUERY_PLANNER_STATIC_SYSTEM_PROMPT`
- `ee/hogai/chat_agent/taxonomy/toolkit.py:140` — `TaxonomyAgentToolkit` class
- `ee/hogai/chat_agent/taxonomy/toolkit.py:146-147` — batch size constants
- `ee/hogai/chat_agent/taxonomy/nodes.py:42` — `TaxonomyAgentNode` class
- `ee/hogai/chat_agent/taxonomy/nodes.py:70` — `_get_model()` using gpt-4.1 at temp 0.3
- `ee/hogai/chat_agent/taxonomy/prompts.py:1` — `PROPERTY_TYPES_PROMPT`
- `ee/hogai/chat_agent/taxonomy/prompts.py:53` — `TAXONOMY_TOOL_USAGE_PROMPT`
- `ee/hogai/chat_agent/sql/nodes.py:14` — `SQLGeneratorNode` class
- `ee/hogai/chat_agent/sql/mixins.py:41` — `HogQLDatabaseMixin`
- `ee/hogai/chat_agent/sql/mixins.py:65` — `HogQLOutputParserMixin`
- `ee/hogai/chat_agent/sql/mixins.py:73-112` — `_validate_hogql_query_sync()`
- `ee/hogai/chat_agent/sql/mixins.py:127` — `HogQLGeneratorMixin`
- `ee/hogai/chat_agent/sql/prompts.py:1` — `HOGQL_GENERATOR_SYSTEM_PROMPT`
- `ee/hogai/chat_agent/sql/prompts.py:50-95` — person_id join limitation + workarounds
- `ee/hogai/chat_agent/sql/prompts.py:152` — `SQL_EXPRESSIONS_DOCS`
- `ee/hogai/chat_agent/sql/prompts.py:269` — `SQL_SUPPORTED_FUNCTIONS_DOCS`
- `ee/hogai/chat_agent/sql/toolkit.py:1` — `generate_sql_schema()` + `SQL_SCHEMA`
- `ee/hogai/chat_agent/query_executor/nodes.py:15` — `QueryExecutorNode`
- `ee/hogai/chat_agent/graph.py:28` — `AssistantGraph`
- `ee/hogai/chat_agent/mode_manager.py` — `ChatAgentModeManager`
- `ee/hogai/api/serializers.py:53` — `ConversationSerializer`
- `ee/api/conversation.py:180` — `ConversationViewSet`

---

## Architecture Documentation

### Two-Path Architecture

```text
User natural language input
          │
          ├─── Legacy path (direct HogQL generation)
          │         posthog/hogql/ai.py:write_sql_from_prompt()
          │         → GPT-4.1-mini + schema context → validate → retry
          │
          └─── Modern path (AI assistant "Max")
                    ee/api/conversation.py:ConversationViewSet.create()
                    → SSE stream
                    → AssistantGraph (LangGraph)
                          → InsightsGraph
                                → QueryPlannerNode (o4-mini, ReAct)
                                      → TaxonomyAgentToolkit (discover events/props)
                                      → route to insight type
                                → [Trends|Funnel|Retention|SQL]GeneratorNode
                                → QueryExecutorNode
```

### LangGraph State Flow

The `AssistantState` carries:

- `messages`: Full conversation history
- `intermediate_steps`: `(AgentAction, result)` tuples for ReAct loops
- `root_tool_call_id`: Current active tool invocation ID
- `root_tool_insight_plan`: Generated textual plan from planner
- `root_tool_insight_type`: Chosen insight type
- `rag_context`: Retrieved actions/schema context
- `query_planner_intermediate_messages`: Planning loop message history
- `plan`: Final validated query plan

### Generator Node Pattern

Every insight-type generator (`Trends`, `Funnel`, `Retention`, `SQL`) follows the same pattern:

1. Inherits from `SchemaGeneratorNode[T]` (base in `ee/hogai/chat_agent/schema_generator/nodes.py`)
2. Pairs with a `*GeneratorToolsNode` for the tools loop
3. Has `OUTPUT_MODEL`, `OUTPUT_SCHEMA`, `INSIGHT_NAME` class attributes
4. Constructs system prompt asynchronously (including schema/memory)
5. LLM generates structured output constrained by JSON schema
6. Output parsed by `_parse_output()`, quality-checked by `_quality_check_output()`
7. On failure: `PydanticOutputParserException` triggers retry via `FAILOVER_PROMPT`

### Taxonomy Query Backends

The taxonomy toolkit queries two dedicated query runners:

- `EventTaxonomyQueryRunner` (`posthog/hogql_queries/ai/event_taxonomy_query_runner.py`): Event/action properties from last 30 days
- `ActorsPropertyTaxonomyQueryRunner` (`posthog/hogql_queries/ai/actors_property_taxonomy_query_runner.py`): Person/session/group properties with sample values

Both use `RECENT_CACHE_CALCULATE_ASYNC_IF_STALE_AND_BLOCKING_ON_MISS` for caching.

### Model Selection per Component

| Component                      | Model                                | Temperature | Notes                            |
| ------------------------------ | ------------------------------------ | ----------- | -------------------------------- |
| Legacy `write_sql_from_prompt` | `gpt-4.1-mini`                       | 0           | Deterministic                    |
| `QueryPlannerNode`             | `o4-mini`                            | —           | Reasoning enabled (auto summary) |
| `TaxonomyAgentNode`            | `gpt-4.1`                            | 0.3         | Parallel tool calls              |
| `SQLGeneratorNode`             | inherited from `SchemaGeneratorNode` | —           | Structured output                |

---

## Open Questions

- What model does `SchemaGeneratorNode` use for the SQL/Trends/Funnel/Retention generators? The base class configuration was not fully inspected.
- How does the RAG context node (`INSIGHT_RAG_CONTEXT`) retrieve and format actions? The full content of `ee/hogai/chat_agent/rag/nodes.py` was not read in detail.
- What is the full content of the `CoreMemory` injected into the SQL generator system prompt?
- How does the `HogQLDatabaseMixin._serialize_database_schema()` format the schema for the LLM?
