# sample-cost-analyzer-agent Improvement Plan

Improvements identified by comparing with `sample-finops-agent`. Each item includes what to adopt, where it lives in the finops agent, where to apply it in the cost-analyzer, and estimated effort.

---

## 1. DDL/DML Blocking for Athena Queries

**Priority:** P0 — Security  
**Effort:** ~30 minutes  
**Source:** `sample-finops-agent/src/lambda/mcp_servers/athena/lambda_function.py` lines 56-60

### Problem

The cost-analyzer's `execute_cur_athena_query` and `execute_vpc_flowlog_query` tools rely solely on IAM permissions to prevent destructive SQL operations. If IAM is misconfigured or overly permissive, there's no application-level defense.

### What to adopt

The finops-agent blocks DDL/DML at the application layer before queries reach Athena:

```python
BLOCKED_SQL_PATTERNS = re.compile(
    r"\b(DROP|DELETE|INSERT|UPDATE|CREATE|ALTER|TRUNCATE|MERGE|GRANT|REVOKE|MSCK)\b",
    re.IGNORECASE,
)

def validate_query(query_string):
    if BLOCKED_SQL_PATTERNS.search(query_string):
        return False, "Only SELECT, SHOW, and DESCRIBE queries are allowed."
    return True, None
```

### Where to apply

- `agent/tools/athena_tools.py` — Add validation at the top of `execute_cur_athena_query` and `execute_vpc_flowlog_query` before calling `athena_service.execute_query()`
- Optionally also add to `agent/services/athena_service.py` in the `execute_query` method as a second layer

### Acceptance criteria

- [ ] Queries containing DDL/DML keywords are rejected before reaching Athena
- [ ] Error message clearly states only SELECT/SHOW/DESCRIBE are allowed
- [ ] Existing SELECT queries continue to work unchanged
- [ ] Unit tests cover blocked patterns (DROP TABLE, DELETE FROM, INSERT INTO, etc.)

---

## 2. RAGAS Evaluation Framework

**Priority:** P0 — Quality  
**Effort:** 1-2 days  
**Source:** `sample-finops-agent/tests/` (entire directory)

### Problem

The cost-analyzer uses Hypothesis for property-based testing of individual tool functions, but has no evaluation framework for agent-level behavior: Does the agent pick the right tool? Does it use correct parameters? Does it handle edge cases gracefully?

### What to adopt

The finops-agent has a 3-tier evaluation framework:

**Tier 1 — Deterministic (no LLM needed)**
- `test_tool_call_accuracy.py`: Verifies the correct tool is called with correct args
- `test_tool_call_f1.py`: Precision/recall for multi-tool scenarios (flexible ordering)

**Tier 2 — LLM-as-judge**
- `test_agent_goal.py`: Uses RAGAS `AgentGoalAccuracy` for semantic correctness
- Custom `AspectCritic` metrics in `tests/helpers/finops_metrics.py`:
  - Date Range Correctness — validates YYYY-MM-01 start dates, exclusive end dates
  - Tool Selection Efficiency — right tool for the job (e.g., `get_cost_and_usage` vs `get_cost_and_usage_comparisons`)
  - Parameter Completeness — group_by when asked for breakdown, correct granularity
  - Cost Analysis Quality — rubric-scored overall quality

**Tier 3 — Ground truth differential**
- `test_ground_truth.py`: Compares MCP tool results against direct boto3 API calls
- `tests/helpers/comparators.py`: Normalizes response formats, float tolerance matching ($0.01)

**Scenario definitions:**
- `tests/scenarios/cost_explorer.py`: 12 scenarios (CE-1 through CE-12)
- `tests/scenarios/athena.py`: 8 scenarios (ATH-1 through ATH-8)
- Each scenario is an `EvalScenario` dataclass with `id`, `user_input`, `target`, `reference_tool_calls`, `metric_type`, and optional `ground_truth_fn`

### Where to apply

Create a new `tests/evals/` directory in the cost-analyzer:

```
tests/
├── evals/
│   ├── conftest.py                  # Agent client fixture
│   ├── scenarios/
│   │   ├── billing_tools.py         # Scenarios for 43 billing tools
│   │   ├── athena_tools.py          # Scenarios for CUR/VPC queries
│   │   └── analysis_tools.py        # Scenarios for analysis helpers
│   ├── helpers/
│   │   ├── agent_client.py          # Client that invokes the Strands agent
│   │   ├── comparators.py           # Adapt from finops-agent
│   │   └── finops_metrics.py        # Adapt AspectCritic metrics
│   ├── test_tool_selection.py       # Tier 1: deterministic
│   ├── test_agent_quality.py        # Tier 2: LLM-as-judge
│   └── test_ground_truth.py         # Tier 3: differential
├── unit/                            # Existing Hypothesis tests
```

### Key adaptation notes

- The cost-analyzer invokes tools directly via Strands SDK, not through an MCP Gateway. The `agent_client.py` wrapper should call the agent's `__call__` method and capture tool invocations from the response.
- The cost-analyzer has 43 billing API tools + 8 specialized tools vs 17 in the finops-agent. Start with scenarios for the most-used tools: `get_cost_and_usage`, `get_cost_forecast`, `get_savings_plans_purchase_recommendation`, `execute_cur_athena_query`.
- Add a `make test-evals` target to the project.

### Acceptance criteria

- [ ] At least 20 eval scenarios covering billing, Athena, and analysis tools
- [ ] Deterministic tests pass without LLM calls
- [ ] LLM-as-judge tests run with configurable evaluator model
- [ ] Ground truth tests compare agent output against direct boto3 calls
- [ ] `make test-evals` runs the full suite

---

## 3. Batch CUR Analysis Tool

**Priority:** P1 — Performance  
**Effort:** 1 day  
**Source:** `sample-finops-agent/src/lambda/mcp_servers/cur_analyst/lambda_function.py`

### Problem

When a user asks "give me a comprehensive cost report," the cost-analyzer's LLM must orchestrate 10-20 sequential tool calls (Cost Explorer queries, Athena queries, savings plan checks). This consumes tokens, adds latency, and risks the LLM losing context mid-analysis.

### What to adopt

The finops-agent's `analyze_cur` tool executes a complete analysis in a single tool call:

1. **Phase 1a** — 5 Cost Explorer API queries (monthly trends by account, by service, current month breakdowns)
2. **Phase 1b** — 5 Savings/RI queries (SP coverage, SP utilization, RI coverage, RI utilization, forecast)
3. **Phase 2** — 10 CUR Athena queries submitted in parallel (monthly totals, service by account, top services, region breakdown, charge types, daily trend, usage types, purchase options, RI/SP savings, instance types)
4. **Post-processing** — Aggregates and truncates results to fit response size limits

### Where to apply

- Create `agent/tools/report_tools.py` with a `generate_comprehensive_report` tool
- Adapt the query templates from the finops-agent's `HISTORICAL_QUERIES` and `DETAILED_QUERIES` dicts
- Use the cost-analyzer's existing `ConcurrentToolExecutor` for parallel Athena query submission
- Add `postprocess_results()`, `aggregate_cost_explorer()`, `aggregate_savings()`, and `limit_cur_rows()` from the finops-agent

### Key adaptation notes

- The cost-analyzer already has `multi_account_executor.py` for parallel execution — reuse it
- The finops-agent hardcodes CUR column names (`unblended_cost`, `amortized_cost`, etc.) which are CUR 2.0 specific. Verify these match the cost-analyzer's CUR schema expectations.
- The finops-agent's `postprocess_results` limits CUR rows to 30 per query. The cost-analyzer streams responses, so the limit can be higher, but keeping results concise helps the LLM produce better analysis.

### Acceptance criteria

- [ ] Single tool call produces a comprehensive cost report
- [ ] Cost Explorer, Savings/RI, and CUR Athena data are all included
- [ ] Athena queries run in parallel (not sequential)
- [ ] Results are post-processed to a reasonable size for LLM consumption
- [ ] Tool works with multi-account configurations
- [ ] Month parameters default to current/previous month when not specified

---

## 4. Config Content Validation at Load Time

**Priority:** P2 — Reliability  
**Effort:** ~1 hour  
**Source:** `sample-finops-agent/src/lambda/mcp_servers/cur_analyst/lambda_function.py` lines 57-63

### Problem

The cost-analyzer validates config file integrity (SHA-256 hash) but doesn't validate the content format. A config with `database: "my;database"` or `output_location: "not-an-s3-path"` would pass integrity checks but fail at runtime with confusing errors.

### What to adopt

The finops-agent validates config values at module load:

```python
_IDENTIFIER_PATTERN = re.compile(r"^[a-zA-Z0-9_]+$")
for _key in ("database", "table"):
    if not _IDENTIFIER_PATTERN.match(CUR_CONFIG[_key]):
        raise ValueError(f"CUR_CONFIG[{_key}] contains invalid characters: {CUR_CONFIG[_key]}")
if not CUR_CONFIG["output_location"].startswith("s3://"):
    raise ValueError(f"CUR_CONFIG[output_location] must start with s3://")
```

### Where to apply

- `agent/services/config_service.py` — Add a `_validate_content()` method called after `_load_config()`
- Validate: AWS account IDs (12-digit numeric), region format, database/table identifiers, S3 paths, role ARN format

### Acceptance criteria

- [ ] Invalid account IDs, regions, database names, or S3 paths raise clear errors at startup
- [ ] Validation runs before any AWS API calls
- [ ] Error messages identify the specific invalid field and expected format

---

## 5. Structured Test Scenarios with EvalScenario Dataclass

**Priority:** P2 — Maintainability  
**Effort:** 2-3 hours  
**Source:** `sample-finops-agent/tests/scenarios/cost_explorer.py`

### Problem

Test scenarios are currently embedded in test functions. Adding new scenarios requires writing new test methods. There's no central registry of "what questions should the agent handle well."

### What to adopt

The finops-agent's `EvalScenario` dataclass pattern:

```python
@dataclass
class EvalScenario:
    id: str                          # e.g., "CE-1"
    user_input: str                  # Natural language question
    target: str                      # Tool category
    reference_tool_calls: list[dict] # Expected tool name + args
    metric_type: str                 # deterministic | llm_judge | ground_truth
    ground_truth_fn: str | None      # Direct API function for differential testing
    ground_truth_args: dict          # Args for ground truth function
    notes: str                       # Implementation notes
```

Scenarios are defined as module-level lists and consumed by parametrized pytest tests. Adding a new scenario is a single dataclass entry — no new test code needed.

### Where to apply

- Create `tests/evals/scenarios/` with scenario files per tool category
- Use `@pytest.mark.parametrize` to iterate over scenarios
- This is a prerequisite for Improvement #2 (RAGAS framework)

### Acceptance criteria

- [ ] All eval scenarios defined as `EvalScenario` instances
- [ ] Scenarios are parametrized in pytest (one test method, many scenarios)
- [ ] Adding a new scenario requires only a new dataclass entry

---

## 6. Makefile with Eval Targets

**Priority:** P2 — Developer experience  
**Effort:** 1-2 hours  
**Source:** `sample-finops-agent/Makefile`

### Problem

The cost-analyzer uses `deploy.sh` and `cli.sh` scripts. There's no unified command interface for common development tasks like running evals, linting, or deploying.

### What to adopt

Key targets from the finops-agent's Makefile:

```makefile
test-evals:        ## Run RAGAS agentic evals
test-ground-truth: ## Run differential ground truth tests
test-all-evals:    ## Run all evals
ruff-check:        ## Check Python code with ruff
ruff-format:       ## Format Python code with ruff
check:             ## Run all checks (lint + format + validate)
deploy:            ## Full deploy (apply + update schemas)
```

### Where to apply

- Create `Makefile` in the cost-analyzer project root
- Include targets for: `deploy`, `test`, `test-evals`, `test-ground-truth`, `lint`, `format`, `check`, `clean`
- Keep existing `deploy.sh` and `cli.sh` as the underlying implementations

### Acceptance criteria

- [ ] `make deploy` runs the full deployment
- [ ] `make test` runs unit tests
- [ ] `make test-evals` runs evaluation suite
- [ ] `make check` runs all linting and validation
- [ ] `make help` shows available targets

---

## 7. Response Size Post-Processing

**Priority:** P3 — Token efficiency  
**Effort:** 3-4 hours  
**Source:** `sample-finops-agent/src/lambda/mcp_servers/cur_analyst/lambda_function.py` (postprocess_results, aggregate_cost_explorer, aggregate_savings, limit_cur_rows)

### Problem

When the cost-analyzer's tools return large Cost Explorer or Athena results, the full response goes into the LLM context. This wastes tokens and can degrade analysis quality when the LLM is overwhelmed with data.

### What to adopt

The finops-agent's post-processing pipeline:

- `aggregate_cost_explorer()` — Simplifies CE response structure, keeps top 20 items per month for trends, top 30 for current month
- `aggregate_savings()` — Extracts monthly coverage/utilization percentages from nested structures
- `limit_cur_rows()` — Caps Athena results at 30 rows with a truncation note

### Where to apply

- `agent/tools/billing_tools.py` — Add result truncation to tools that return large datasets (e.g., `get_cost_and_usage` with DAILY granularity)
- `agent/tools/athena_tools.py` — Add row limits to CUR and VPC Flow Log query results
- Create `agent/tools/result_utils.py` for shared post-processing functions

### Acceptance criteria

- [ ] Cost Explorer results are capped at configurable row limits
- [ ] Athena results include a truncation note when rows are dropped
- [ ] Trend data retains time-series structure but limits items per period
- [ ] Existing tool behavior unchanged for small result sets

---

## Summary

| # | Improvement | Priority | Effort | Impact |
|---|-------------|----------|--------|--------|
| 1 | DDL/DML Blocking | P0 | 30 min | Security defense-in-depth |
| 2 | RAGAS Eval Framework | P0 | 1-2 days | Agent quality assurance |
| 3 | Batch CUR Analysis Tool | P1 | 1 day | Latency and token cost reduction |
| 4 | Config Content Validation | P2 | 1 hour | Fail-fast on misconfig |
| 5 | EvalScenario Dataclass | P2 | 2-3 hours | Test maintainability (prereq for #2) |
| 6 | Makefile with Eval Targets | P2 | 1-2 hours | Developer experience |
| 7 | Response Size Post-Processing | P3 | 3-4 hours | Token efficiency |

**Recommended execution order:** 1 → 4 → 5 → 2 → 6 → 3 → 7

Items 1 and 4 are quick wins. Item 5 is a prerequisite for item 2. Item 6 ties the eval workflow together. Items 3 and 7 are performance optimizations that can be done independently.

---

## 8. Athena Tools Hardening

**Priority:** P1 — Reliability & Cost Protection  
**Effort:** 1–2 days  
**Location:** `agent/tools/athena_tools.py`, `agent/services/athena_service.py`

### Issues Identified

| # | Issue | Impact | Fix |
|---|-------|--------|-----|
| 1 | **100-row hard limit with no indication** | Users miss data beyond 100 rows, no pagination | Add row limit note to output, or support `NextToken` pagination |
| 2 | **Hardcoded date in CUR schema example** | LLM may copy `2026-01-01` instead of calculating correct date | Dynamically generate example with current month, or add "replace with actual date" comment |
| 3 | **Hardcoded Unix timestamps in VPC examples** | LLM must calculate epoch times — error-prone | Add `to_unixtime()` conversion pattern in examples |
| 4 | **No scan size guard** | Poorly filtered VPC query can scan TBs ($5+ per query) | Add `LIMIT` enforcement check, or configurable `max_scan_bytes` threshold |
| 5 | **Docstring says "default (payer)" but code uses first member** | Minor doc inaccuracy in `execute_vpc_flowlog_query` | Fix docstring to say "first configured member account" |
| 6 | **60-second query timeout too short for VPC** | Complex VPC queries on large datasets timeout silently | Increase default for VPC, or suggest partition filters on timeout |
| 7 | **No inter-AZ correlation workflow** | Agent acknowledges limitation but doesn't guide CUR+VPC correlation | Add guidance: "Query CUR for inter-AZ costs by resource, then VPC for traffic patterns" |
| 8 | **No EXPLAIN support for cost estimation** | Can't preview scan size before running expensive queries | Encourage `EXPLAIN` before large queries, or auto-run it |
| 9 | **Multi-account query doesn't aggregate results** | Per-account top-N, not global top-N across accounts | Add post-processing to merge and re-sort, or document limitation |

### Recommended Implementation Order

1. **#5** — Docstring fix (5 minutes)
2. **#1** — Add "Results limited to 100 rows" note to output (15 minutes)
3. **#4** — Scan size guard / LIMIT enforcement (1–2 hours)
4. **#2 + #3** — Dynamic date examples (1 hour)
5. **#6** — Configurable timeout per tool type (30 minutes)
6. **#7** — Inter-AZ correlation guidance in schema info (30 minutes)
7. **#8** — EXPLAIN pre-check (1–2 hours)
8. **#9** — Multi-account result aggregation (2–3 hours)

---

## Feature Improvements — AgentCore Platform Capabilities

This section covers feature enhancements that leverage Amazon Bedrock AgentCore's managed platform capabilities. These go beyond code fixes — they add new functionality to the agent.

---

### Short Term (1–5 days each)

#### F1. Short-Term Memory (Session Persistence)

**Priority:** P0 | **Effort:** 1 day

**What it offers:** Conversation continuity within a session. Users can ask follow-up questions like "now break that down by region" or "compare that to last month" without re-stating the full context. Currently, each invocation is stateless — the agent doesn't remember what was discussed earlier in the session.

**How:** Remove `--disable-memory` from `deploy.sh`. Install `bedrock-agentcore[strands-agents]` and configure `AgentCoreMemorySessionManager` with short-term memory (STM) enabled. STM stores conversation history per session ID and automatically injects relevant context into each request.

```python
from bedrock_agentcore.memory.integrations.strands import AgentCoreMemorySessionManager

session_manager = AgentCoreMemorySessionManager(
    memory_id="cost-analyzer-memory",
    session_id=session_id,
    short_term_memory=True
)
agent = Agent(model=model, tools=tools, session_manager=session_manager)
```

**STM benefit for Glue Catalog & Partition Detection:**

Currently, `get_cur_schema_info()` and `get_vpc_flowlog_schema_info()` are called once per conversation to discover schema, but results only live in the tool's return text. With STM:
- Schema stays in working memory — the agent doesn't need to re-parse column lists for each query
- Partition values discovered early (e.g., `partition_0=region, partition_1=year`) are remembered for all subsequent queries in the session
- Failed query patterns (e.g., "column X doesn't exist") are avoided on retry — the agent won't attempt the same invalid column again

---

#### F2. Observability & Distributed Tracing

**Priority:** P0 | **Effort:** 1 day

**What it offers:** Visibility into what the agent is doing — which tools it calls, how long each takes, where latency comes from, and what errors occur. Currently, the agent uses basic Python `logging` with no structured tracing.

**How:** AgentCore natively supports OpenTelemetry and CloudWatch. Add trace spans around tool calls, Athena queries, and STS role assumptions. Create a CloudWatch dashboard showing: queries/minute, average response time, tool call distribution, Athena scan costs, and error rates.

**Value:** Debug slow queries ("why did this take 45 seconds?"), identify cost hotspots ("80% of Athena scan costs come from VPC queries"), and monitor agent health in production.

---

#### F3. Guardrails Integration

**Priority:** P1 | **Effort:** 1 day

**What it offers:** Platform-level enforcement of the agent's boundaries. Currently, the system prompt says "don't create resources" and regex patterns detect prompt injection — but these are advisory. Guardrails enforce it at the model layer.

**How:** Create a Bedrock Guardrail with:
- **Topic denial:** Block requests to create/modify/delete AWS resources
- **Content filtering:** Block harmful content generation
- **PII detection:** Redact account IDs or sensitive data in responses
- **Word filters:** Block specific terms that indicate out-of-scope requests

Attach the guardrail to the model invocation. No agent code changes needed — it's a model-level configuration.

---

#### F4. Code Interpreter for Data Visualization

**Priority:** P1 | **Effort:** 2 days

**What it offers:** The agent can generate charts, calculate trends, and produce formatted reports — not just raw tables. Currently, Athena returns tabular data and the agent formats it as markdown. With Code Interpreter, it can create bar charts, pie charts, trend lines, and executive summaries.

**How:** Enable AgentCore's managed Code Interpreter tool. The agent writes Python code (matplotlib, pandas) that executes in a secure sandbox and returns images or formatted output.

**Example interactions:**
- "Show me a chart of my monthly costs for the last 6 months"
- "Create a pie chart of cost distribution by service"
- "Calculate the month-over-month growth rate"

---

#### F5. Result Caching

**Priority:** P1 | **Effort:** 2 days

**What it offers:** Faster responses and lower costs for repeated queries. Currently, every query hits Athena or Cost Explorer fresh — even if the same question was asked 5 minutes ago. CUR data doesn't change intra-day, and Cost Explorer data updates at most once daily.

**How:** Implement a cache layer (DynamoDB or in-memory) keyed by:
- SQL query hash + time window (for Athena)
- API parameters hash + date (for Cost Explorer)

Cache TTL: 1 hour for Cost Explorer, 6 hours for CUR queries (data is daily). Invalidate on new billing period.

**Value:** 2nd query for "top EC2 costs last month" returns in <1 second instead of 5–15 seconds. Athena scan costs drop to zero for cached queries.

---

### Long Term (1–3 weeks each)

#### F6. Long-Term Memory (Cross-Session Learning)

**Priority:** P2 | **Effort:** 3 days

**What it offers:** The agent remembers user preferences, past analyses, and organizational context across sessions. It gets smarter over time per user — like a FinOps analyst who knows your environment.

**How:** Enable AgentCore Memory with long-term memory (LTM) strategies:
- **Semantic memory:** Remembers facts ("Production workloads are in us-east-1 and eu-west-1")
- **Summarization:** Condenses past sessions ("Last week, user investigated S3 egress costs and found CloudFront was misconfigured")
- **User preferences:** Learns patterns ("User always wants AmortizedCost metric, prefers monthly granularity")

```python
session_manager = AgentCoreMemorySessionManager(
    memory_id="cost-analyzer-memory",
    session_id=session_id,
    short_term_memory=True,
    long_term_memory=True,
    long_term_memory_strategies=["SEMANTIC", "SUMMARIZATION", "USER_PREFERENCE"]
)
```

**Example:** After a few sessions, the agent proactively says: "Based on your previous analysis, your S3 egress costs have increased 20% since last month. Want me to investigate?"

**LTM benefit for Glue Catalog & Partition Detection:**

Long-term memory eliminates schema discovery entirely for returning users:

| Memory Strategy | What to Store | Example |
|----------------|---------------|---------|
| **Semantic** | Table schemas, column names, partition structures | "CUR table `cur_db.cur_report` has `line_item_net_unblended_cost` column" |
| **Semantic** | Account-to-table mappings | "Account 222 has VPC Flow Logs in `vpc_db.flow_logs`" |
| **Semantic** | Partition format per table | "VPC Flow Logs use `partition_0=region, partition_1=year, partition_2=month, partition_3=day`" |
| **Summarization** | Query patterns that worked/failed | "JOINs on VPC Flow Logs cause duplication — use simple aggregations" |
| **Summarization** | Data availability windows | "VPC Flow Logs in account 333 only have data from March 2026 onwards" |
| **User Preference** | Preferred cost metric, default time ranges | "User prefers AmortizedCost, usually asks about last month" |

**Performance impact:**

Without memory (current):
```
Session 1: "top EC2 instances by cost"
  → get_cur_schema_info() via Glue API (5s)
  → get_current_date_context() (1s)
  → Write + execute query (10s)
  Total: ~16s

Session 2 (next day): "top S3 buckets by cost"
  → get_cur_schema_info() AGAIN via Glue API (5s)
  → get_current_date_context() (1s)
  → Write + execute query (10s)
  Total: ~16s
```

With LTM:
```
Session 2: "top S3 buckets by cost"
  → Agent recalls from LTM: schema, partitions, working patterns
  → get_current_date_context() (1s)
  → Write optimized query immediately, execute (10s)
  Total: ~11s (schema discovery eliminated)
```

The agent learns your infrastructure over time — table structures, partition formats, which columns exist, which query patterns work — and applies that knowledge immediately in future sessions.

---

#### F7. Scheduled Reports (Proactive Agent)

**Priority:** P2 | **Effort:** 2–3 days

**What it offers:** Automated cost reports delivered on a schedule — daily, weekly, or monthly — without human intervention. The agent becomes proactive instead of purely reactive.

**How:** Use Amazon EventBridge scheduled rules to invoke the agent with predefined queries from `prompts.yaml`. Results are delivered via SNS (email), Slack webhook, or written to S3.

**Example workflows:**
- Daily: "What were yesterday's costs? Any anomalies?"
- Weekly: "Compare this week to last week. Top 5 cost changes."
- Monthly: "Full month cost report with optimization recommendations."

---

#### F8. Evaluation Framework

**Priority:** P2 | **Effort:** 3–5 days

**What it offers:** Automated testing of agent behavior — does it call the right tools, produce accurate results, and stay within its boundaries? AgentCore provides built-in evaluation capabilities.

**How:** Build a test suite with scenarios:
- "What are my top 5 services?" → Should call `get_cost_and_usage` (not Athena)
- "Which EC2 instances cost the most?" → Should call `execute_cur_athena_query` (not Cost Explorer)
- "Create an EC2 instance" → Should refuse politely

Track metrics: tool selection accuracy, response relevance, boundary adherence, latency.

---

#### F9. AgentCore Identity for Multi-Tenant Access

**Priority:** P3 | **Effort:** 1 week

**What it offers:** Safe multi-user deployment where each user only sees costs for their authorized accounts. Currently, whoever invokes the agent can query all configured accounts.

**How:** Use AgentCore Identity to map end users to specific account scopes:
- User A (team lead) → sees all accounts
- User B (developer) → sees only their team's account
- User C (finance) → sees payer-level aggregates only

The agent checks the caller's identity and filters tool access accordingly.

---

#### F10. Multi-Agent Architecture

**Priority:** P3 | **Effort:** 2 weeks

**What it offers:** Specialized sub-agents that are faster, simpler, and independently improvable. Instead of one agent with 51 tools, split into focused specialists.

**How:** Use Strands' multi-agent patterns (Agent-as-Tool or Swarm):
- **Billing Agent:** Cost Explorer, Budgets, Savings Plans, Pricing (43 APIs)
- **Resource Agent:** CUR Athena queries for resource-level analysis
- **Network Agent:** VPC Flow Logs analysis
- **Advisor Agent:** Optimization recommendations + AWS Knowledge search
- **Orchestrator:** Routes user queries to the right specialist

**Value:** Each agent has a smaller tool set → faster tool selection, lower token usage, easier to test and improve independently.

---

#### F11. AgentCore Gateway for Tool Management

**Priority:** P3 | **Effort:** 1 week

**What it offers:** Decouple tools from the agent deployment. Add, update, or remove tools without redeploying the agent. Tools become managed API endpoints accessible by multiple agents.

**How:** Register billing tools and Athena tools as Gateway targets. The agent calls tools through the Gateway instead of directly. New tools (e.g., a CloudWatch metrics tool) can be added to the Gateway without touching agent code.

---

### Priority Matrix

| # | Feature | Effort | Impact | Priority |
|---|---------|--------|--------|----------|
| F1 | Short-term memory | 1 day | High | P0 |
| F2 | Observability/tracing | 1 day | High | P0 |
| F3 | Guardrails | 1 day | Medium | P1 |
| F4 | Code Interpreter | 2 days | High | P1 |
| F5 | Result caching | 2 days | Medium | P1 |
| F6 | Long-term memory | 3 days | High | P2 |
| F7 | Scheduled reports | 2–3 days | Medium | P2 |
| F8 | Evaluation framework | 3–5 days | High | P2 |
| F9 | Multi-tenant identity | 1 week | Medium | P3 |
| F10 | Multi-agent architecture | 2 weeks | Medium | P3 |
| F11 | Gateway for tools | 1 week | Medium | P3 |

**Recommended execution order:** F1 → F2 → F3 → F4 → F5 → F6 → F8 → F7 → F9 → F10 → F11
