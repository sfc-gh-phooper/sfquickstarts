author: Josh Reini
id: self-improving-agents-with-cortex-code
categories: snowflake-site:taxonomy/solution-center/certification/quickstart, snowflake-site:taxonomy/product/ai, snowflake-site:taxonomy/product/platform, snowflake-site:taxonomy/snowflake-feature/cortex-llm-functions
language: en
summary: Build a Cortex Agent, evaluate it with Agent GPA, analyze failures, and optimize its instructions — all from Cortex Code.
environments: web
status: Published
feedback link: https://github.com/Snowflake-Labs/sfguides/issues
tags: Cortex Agents, Evaluations, AI, LLM, Snowflake Cortex, Cortex Code, Agent GPA

# Self-Improving Agents with Cortex Code

## Overview

Building AI agents is just the beginning — understanding how well they perform and systematically improving them is what separates prototypes from production systems. In this guide, you'll build a marketing analytics agent, evaluate it with Agent GPA, analyze its failures, and optimize its instructions through iterative improvement — all driven by Cortex Code.

By the end, you'll have a versioned agent that gets measurably better with each round of evaluation and optimization.

| Round | Focus |
|-------|-------|
| Baseline | Test the agent, curate traces into an eval dataset, run first evaluation |
| Round 1 | Add multi-tool routing queries, optimize instructions, measure improvement |
| Round 2 | Add edge cases + ambiguity, optimize again, build score matrix |

### Architecture

```
┌──────────────────────────────────────────────────────┐
│              MARKETING CAMPAIGNS AGENT               │
│                                                      │
│  Tool 1: query_performance_metrics (Cortex Analyst)  │
│  Tool 2: search_campaign_content   (Cortex Search)   │
│  Tool 3: generate_campaign_report  (Stored Proc)     │
│  Tool 4: web_search                (Web Search)      │
│  Tool 5: data_to_chart             (Visualization)   │
└──────────────────────────────────────────────────────┘
        │                                     ▲
        ▼                                     │
┌───────────────┐    ┌──────────────┐   ┌─────────────┐
│  Evaluate     │───▶│  Analyze     │──▶│  Optimize   │
│  (Agent GPA)  │    │  (failures)  │   │  (AI-driven)│
└───────────────┘    └──────────────┘   └─────────────┘
```

### What You'll Learn

- How to build a Cortex Agent with multiple tool types (Cortex Analyst, Cortex Search, stored procedures, web search, data-to-chart)
- How to use Snowflake Intelligence to interact with your agent and generate observability traces
- How to curate evaluation datasets from real agent interactions using Cortex Code
- How to run Agent GPA evaluations with built-in metrics
- How to analyze failure patterns and generate improved orchestration instructions
- How to build a score matrix comparing agent versions across datasets

### What You'll Build

A complete agent optimization workflow:

- A marketing campaigns agent with 5 tools and versioned configurations
- Evaluation datasets curated from real agent interactions
- Multiple agent versions with progressively better orchestration instructions
- A score matrix demonstrating measurable improvement across versions

### What You'll Need

- A [Snowflake account](https://signup.snowflake.com/?utm_source=snowflake-devrel&utm_medium=developer-guides&utm_cta=developer-guides) with ACCOUNTADMIN access
- [Cross-region inference](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-suite-cross-region) enabled (required for evaluation LLM judge models)
- ~5 minutes for setup script to complete

### Prerequisites

- Basic familiarity with Snowflake SQL and Cortex Agents
- Python 3.8+ (for Cortex Code CLI installation)

<!-- ------------------------ -->

## Install Cortex Code

Cortex Code is an AI-powered CLI that you'll use throughout this guide to analyze evaluation results, identify failure patterns, and generate improved agent instructions.

Install it via pip:

```bash
pip install snowflake-cli
```

Verify the installation:

```bash
cortex --version
```

For detailed setup instructions, see the [Cortex Code docs](https://docs.snowflake.com/en/developer-guide/cortex-code/cortex-code).

<!-- ------------------------ -->

## Run Setup

Download the [`setup.sql`](https://github.com/Snowflake-Labs/sfguide-self-improving-agents-with-cortex-code/blob/main/assets/setup.sql) file from the repository.

Open a Snowflake worksheet in Snowsight and run the entire `setup.sql` file. This creates:

- Database `SELF_IMPROVING_AGENT_DB` with schema `AGENTS`
- 4 data tables (25 campaigns, ~1578 performance records, content, feedback)
- Semantic view, Cortex Search service, report generation procedure
- The agent `MARKETING_CAMPAIGNS_AGENT` (VERSION$1 — no orchestration instructions)
- Eval data tables (8 baseline queries + 10 harder queries for later rounds)

Verify setup succeeded — the final statement should print a success banner.

<!-- ------------------------ -->

## Test the Agent in Snowflake Intelligence

Open the agent in Snowflake Intelligence:

1. Go to [ai.snowflake.com](https://ai.snowflake.com) or in Snowsight select **AI & ML > Agents**
2. Select **MARKETING_CAMPAIGNS_AGENT**

The agent has VERSION$1 with no orchestration instructions. Try a mix of easy and hard queries to generate traces. Copy-paste these one at a time:

### Good single-tool queries

The agent should handle these well:

- `What is the total spend across all campaigns?`
- `What content was used in the Summer Sale campaign?`
- `Which campaign had the highest ROI?`

### Multi-step chained queries

These require 3+ tools in sequence — the agent will miss steps:

- `Which campaign had the highest ROI and what did customers say about it? Generate a report for that campaign too.`
- `Find our worst performing campaigns, look up what customers complained about, compare to industry benchmarks, and recommend fixes`

### Vague / underspecified queries

The agent won't know where to start:

- `For each of our top 5 campaigns by revenue, show me the customer feedback and whether the A/B test results support scaling them up`
- `Build me a quarterly business review — top campaigns, underperformers, customer sentiment trends, and how we stack up against competitors`

Notice the patterns: single-tool queries work okay, but the agent fails on multi-tool coordination and ambiguity. These traces will be the raw material for building eval datasets.

<!-- ------------------------ -->

## Curate an Eval Dataset

Now open Cortex Code and use it to curate the traces from the previous step into an evaluation dataset:

```bash
cortex --bypass
```

Then enter the following prompt:

```
Use the dataset-curation skill to pull observability traces for
SELF_IMPROVING_AGENT_DB.AGENTS.MARKETING_CAMPAIGNS_AGENT and curate
an evaluation dataset. Include a mix of:
- Simple single-tool queries the agent handles well
- Multi-tool queries where it struggles
- Edge cases like ambiguous or out-of-scope questions

For each query, include ground truth with expected tool invocations.
Store it in SELF_IMPROVING_AGENT_DB.AGENTS and register it as an
evaluation dataset called DS_BASELINE.
```

Cortex Code will:

1. Query the observability traces
2. Help you select and annotate queries with ground truth
3. Create an eval table and register it via `SYSTEM$CREATE_EVALUATION_DATASET`

<!-- ------------------------ -->

## Baseline Evaluation

Run Agent GPA on your curated dataset. Enter this prompt in Cortex Code:

```
Run an evaluation of SELF_IMPROVING_AGENT_DB.AGENTS.MARKETING_CAMPAIGNS_AGENT
against the DS_BASELINE dataset. Use the default Agent GPA metrics:
answer_correctness, logical_consistency, groundedness, execution_efficiency,
and tool_selection. Upload the config to a stage in
SELF_IMPROVING_AGENT_DB.AGENTS and kick off the eval.
```

Once the eval completes, analyze the results:

```
Show me the baseline evaluation results. Break down scores by metric and
identify which queries scored lowest. What are the common failure patterns?
```

### GPA Metrics

| Metric | Description |
|--------|-------------|
| `answer_correctness` | Is the answer factually right? |
| `tool_selection` | Did the agent pick the right tool(s)? |
| `groundedness` | Are claims backed by evidence from tool outputs? |
| `execution_efficiency` | Was the tool usage minimal and efficient? |
| `logical_consistency` | Is the reasoning coherent? |

<!-- ------------------------ -->

## Round 1 — Optimize Multi-Tool Routing

### Analyze failures

Enter this prompt in Cortex Code:

```
Dig into the lowest-scoring queries from the baseline eval of
SELF_IMPROVING_AGENT_DB.AGENTS.MARKETING_CAMPAIGNS_AGENT. Show me the
actual tool calls the agent made vs what ground truth expected. What
patterns do you see?
```

**Common failure patterns you'll see:**

- Agent uses only one tool when the query needs both quantitative AND qualitative data
- Report generation fails because the agent doesn't look up `campaign_id` first
- Vague answers that don't combine data from multiple sources

### Optimize with Cortex Code

```
Based on the failure analysis, generate improved orchestration instructions
for SELF_IMPROVING_AGENT_DB.AGENTS.MARKETING_CAMPAIGNS_AGENT that fix
multi-tool routing. The instructions should tell the agent when to use
multiple tools and in what order. Apply the changes and commit as VERSION$2.
```

Cortex Code will:

1. Draft improved orchestration instructions with explicit tool routing rules
2. Apply via `ALTER AGENT ... MODIFY LIVE VERSION SET SPECIFICATION = ...`
3. Commit as VERSION$2

**What changes:** Only the `instructions.orchestration` field. Tools, tool_resources, and models stay identical. Better instructions are the only lever.

### Add harder queries and re-evaluate

```
There are multi-tool routing queries staged in
SELF_IMPROVING_AGENT_DB.AGENTS.HARD_QUERIES_STAGING (DAY_NUMBER = 1).
Create an expanded eval dataset that includes the baseline queries plus
these 5 new ones. Register it as DS_ROUND1 and run the eval of
SELF_IMPROVING_AGENT_DB.AGENTS.MARKETING_CAMPAIGNS_AGENT against it
for both VERSION$1 and VERSION$2 so we can compare.
```

<!-- ------------------------ -->

## Round 2 — Edge Cases & Ambiguity

### Preview Day 2 queries

```
Show me the Day 2 queries from SELF_IMPROVING_AGENT_DB.AGENTS.HARD_QUERIES_STAGING.
These test edge cases and ambiguity — things like "Tell me about our best
campaign" where "best" is undefined, or "What should we do next quarter?"
which requires synthesis.
```

### Evaluate and optimize

```
Create the full 18-query dataset (baseline + day 1 + day 2 queries from
SELF_IMPROVING_AGENT_DB.AGENTS.HARD_QUERIES_STAGING), register it as
DS_ROUND2, evaluate SELF_IMPROVING_AGENT_DB.AGENTS.MARKETING_CAMPAIGNS_AGENT
VERSION$2 against it, analyze the new failures, and generate VERSION$3
instructions that handle ambiguous queries and graceful degradation
without regressing on the Day 1 fixes.
```

### Build the score matrix

```
Run all versions (V1, V2, V3) of
SELF_IMPROVING_AGENT_DB.AGENTS.MARKETING_CAMPAIGNS_AGENT against all
datasets (DS_BASELINE, DS_ROUND1, DS_ROUND2) and build a score matrix
showing improvement across versions and datasets. Visualize the results.
```

**What to look for in the matrix:**

```
              |  Baseline |  +Multi   |  +Edge
  V1 (base)  |  0.xxx    |  0.xxx    |  0.xxx
  V2 (day 1) |  0.xxx    |  0.xxx    |  0.xxx
  V3 (day 2) |  0.xxx    |  0.xxx    |  0.xxx
```

- **Diagonal improvement**: V2 should beat V1 on multi-tool queries; V3 should beat V2 on edge cases
- **No regressions**: V3 should not score worse than V1 on baseline queries
- **Generalization**: Later versions handle harder queries without losing ground on easy ones

<!-- ------------------------ -->

## Review the Journey

Ask Cortex Code to summarize the full optimization journey:

```
Show me a summary of everything we did — all agent versions, what changed
in each version's instructions, the eval runs, and the score progression.
```

You can also inspect agent versions directly:

```sql
SHOW VERSIONS IN AGENT SELF_IMPROVING_AGENT_DB.AGENTS.MARKETING_CAMPAIGNS_AGENT;
```

### Explore Observability Traces

Ask Cortex Code to dig into what the agent actually did during evaluations:

```
Show me the observability traces for
SELF_IMPROVING_AGENT_DB.AGENTS.MARKETING_CAMPAIGNS_AGENT. Find an example
where V1 failed and V3 succeeded on the same query — compare the tool
calls and reasoning.
```

<!-- ------------------------ -->

## Conclusion and Resources

Congratulations! You've built a self-improving AI agent workflow that demonstrates measurable improvement through iterative evaluation and optimization.

### What You Learned

- How to build a multi-tool Cortex Agent with Cortex Analyst, Cortex Search, stored procedures, web search, and data-to-chart capabilities
- How to generate observability traces by interacting with your agent in Snowflake Intelligence
- How to use Cortex Code to curate evaluation datasets from real agent interactions
- How to run Agent GPA evaluations with 5 built-in metrics
- How to analyze failure patterns and generate improved orchestration instructions
- How to build a score matrix comparing agent versions across progressively harder datasets
- That better instructions — not more tools — are the key lever for agent improvement

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Agent GPA** | 5-metric evaluation framework measuring correctness, tool selection, groundedness, efficiency, and consistency |
| **Orchestration Instructions** | The only thing that changes between versions — natural language instructions telling the agent how to route queries and coordinate tools |
| **Eval Dataset** | Frozen snapshot of queries + ground truth used to score agent versions |
| **Score Matrix** | Versions x Datasets grid showing measurable improvement |
| **Cortex Code** | AI-powered CLI that analyzes eval results, identifies failures, and generates improved agent instructions |

### Related Resources

- [Agent GPA Paper](https://bit.ly/agent-gpa)
- [Cortex Agent Evals Guide](https://bit.ly/cortex-agent-evals)
- [DeepLearning.AI Course](https://bit.ly/deeplearning-agent-gpa)
- [Cortex Code Docs](https://bit.ly/cortex-code-docs)

### Cleanup

```sql
USE ROLE ACCOUNTADMIN;
DROP DATABASE IF EXISTS SELF_IMPROVING_AGENT_DB;
DROP ROLE IF EXISTS SELF_IMPROVING_AGENT_ROLE;
```
