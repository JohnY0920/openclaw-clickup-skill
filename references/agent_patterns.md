# Agent Orchestration Patterns with ClickUp

Patterns and conventions for using ClickUp as a task backbone in multi-agent systems.

---

## Table of Contents

1. [Planner-Worker Pattern](#planner-worker-pattern)
2. [Task Lifecycle & Status Convention](#task-lifecycle--status-convention)
3. [Agent Communication via Comments](#agent-communication-via-comments)
4. [Task Claiming (Picking Up Work)](#task-claiming-picking-up-work)
5. [Error Handling & Retries](#error-handling--retries)
6. [Dependency Chains](#dependency-chains)
7. [Polling vs Webhooks](#polling-vs-webhooks)
8. [REST API vs MCP Server](#rest-api-vs-mcp-server)
9. [Complete Workflow Example](#complete-workflow-example)

---

## Planner-Worker Pattern

The core pattern for OpenClaw-style multi-agent orchestration:

```
┌──────────────┐     creates tasks     ┌─────────────────┐
│ Planner Agent│ ──────────────────────>│  ClickUp List   │
│              │                        │  (Task Queue)   │
└──────────────┘                        └────────┬────────┘
                                                 │
                           ┌─────────────────────┼─────────────────────┐
                           │                     │                     │
                    ┌──────▼──────┐       ┌──────▼──────┐       ┌─────▼───────┐
                    │ Research    │       │ Code        │       │ Review      │
                    │ Agent       │       │ Agent       │       │ Agent       │
                    └─────────────┘       └─────────────┘       └─────────────┘
                    polls for tasks       polls for tasks       polls for tasks
                    tagged "research"     tagged "code"         tagged "review"
```

### Planner Agent Responsibilities

1. Decompose a goal into discrete tasks
2. Create tasks with appropriate tags, priorities, and descriptions
3. Set dependencies between tasks that must run sequentially
4. Monitor overall progress by querying task statuses
5. Handle escalation when tasks fail or stall

### Worker Agent Responsibilities

1. Poll for tasks matching their tag/type with status "to do"
2. Claim a task by updating status to "picked up" and adding themselves
3. Process the task and update status to "in progress"
4. Post progress updates as comments
5. On completion, update status to "done" and attach output reference
6. On failure, update status to "failed" and post error details as comment

---

## Task Lifecycle & Status Convention

Recommended statuses for the agent task list:

```
to do  ──>  picked up  ──>  in progress  ──>  review  ──>  done
  │              │                │                           │
  │              │                ▼                           │
  │              │            blocked ──> (unblocked) ──>     │
  │              │                                            │
  │              ▼                                            │
  └──────>  failed  ──────────> (retry) ──────────────────────┘
```

| Status       | Meaning                                          | Who Sets It      |
|--------------|--------------------------------------------------|------------------|
| `to do`      | Created by planner, waiting for pickup            | Planner agent    |
| `picked up`  | An agent has claimed this task                    | Worker agent     |
| `in progress`| Agent is actively working on it                   | Worker agent     |
| `blocked`    | Waiting on dependency or external input            | Worker agent     |
| `review`     | Work complete, needs verification                 | Worker agent     |
| `done`       | Task successfully completed                       | Worker/planner   |
| `failed`     | Task failed after max retries                     | Worker agent     |

---

## Agent Communication via Comments

Use task comments as a structured message log between agents.

### Comment Convention

Use a prefix-based format for machine-parseable comments:

```
[AGENT:{agent_id}] [TYPE:{message_type}] {message body}
```

Message types:
- `STATUS` - Progress update
- `OUTPUT` - Result reference
- `ERROR` - Error details
- `REQUEST` - Asking planner for input
- `INFO` - General information

### Examples

```python
# Worker reports progress
add_comment(
    task_id,
    "[AGENT:research-agent-01] [TYPE:STATUS] Processed 500/1000 records. "
    "ETA: 5 minutes."
)

# Worker posts output reference
add_comment(
    task_id,
    "[AGENT:code-agent-01] [TYPE:OUTPUT] Generated script saved to "
    "/outputs/transform_v2.py. 142 lines, all tests passing."
)

# Worker reports error
add_comment(
    task_id,
    "[AGENT:data-agent-01] [TYPE:ERROR] API rate limit hit on external "
    "data source. Retry #2 scheduled in 60s."
)

# Worker requests planner input
add_comment(
    task_id,
    "[AGENT:research-agent-01] [TYPE:REQUEST] Found 3 candidate approaches. "
    "Need planner decision: (A) Random Forest, (B) XGBoost, (C) LightGBM. "
    "See comparison in description."
)
```

### Parsing Comments

```python
import re

def parse_agent_comment(comment_text: str) -> dict:
    """Parse structured agent comments."""
    pattern = r'\[AGENT:(?P<agent_id>[^\]]+)\]\s*\[TYPE:(?P<type>[^\]]+)\]\s*(?P<body>.*)'
    match = re.match(pattern, comment_text, re.DOTALL)
    if match:
        return match.groupdict()
    return {"agent_id": "unknown", "type": "UNSTRUCTURED", "body": comment_text}
```

---

## Task Claiming (Picking Up Work)

To prevent multiple agents from claiming the same task, use an optimistic locking approach:

```python
def claim_task(task_id: str, agent_id: str) -> bool:
    """
    Attempt to claim a task. Returns True if successful.

    Strategy: Check current status, update if still 'to do'.
    Not perfectly atomic, but sufficient for most agent workloads.
    """
    task = get_task(task_id)
    if task["status"]["status"] != "to do":
        return False  # Already claimed by another agent

    update_task(task_id, status="picked up")
    add_comment(task_id, f"[AGENT:{agent_id}] [TYPE:STATUS] Task claimed.")

    # Set agent_id custom field if configured
    # set_custom_field(task_id, AGENT_ID_FIELD, agent_id)

    return True
```

### Agent Polling Loop

```python
import time

def agent_poll_loop(
    agent_id: str,
    agent_tag: str,
    process_fn,
    poll_interval: int = 30,
    list_id: str = None,
):
    """
    Main loop for a worker agent.

    Args:
        agent_id: Unique identifier for this agent instance.
        agent_tag: Tag to filter tasks (e.g., "research", "code").
        process_fn: Callable that takes a task dict and returns a result dict.
        poll_interval: Seconds between polls.
        list_id: ClickUp list to poll.
    """
    while True:
        try:
            tasks = get_tasks(
                list_id=list_id,
                statuses=["to do"],
                # Note: API doesn't directly filter by tag in get_tasks.
                # Filter client-side or use get_filtered_tasks.
            )

            # Client-side tag filter
            matching = [
                t for t in tasks
                if any(tag["name"] == agent_tag for tag in t.get("tags", []))
            ]

            for task in matching:
                if not claim_task(task["id"], agent_id):
                    continue

                # Process the task
                update_task(task["id"], status="in progress")
                try:
                    result = process_fn(task)
                    add_comment(
                        task["id"],
                        f"[AGENT:{agent_id}] [TYPE:OUTPUT] {result.get('summary', 'Complete')}"
                    )
                    update_task(task["id"], status="done")

                except Exception as e:
                    add_comment(
                        task["id"],
                        f"[AGENT:{agent_id}] [TYPE:ERROR] {str(e)}"
                    )
                    update_task(task["id"], status="failed")

        except Exception as e:
            print(f"Poll error: {e}")

        time.sleep(poll_interval)
```

---

## Error Handling & Retries

### Retry Pattern with Custom Fields

Use a `retry_count` custom field to track attempts:

```python
MAX_RETRIES = 3

def handle_task_failure(task_id: str, agent_id: str, error: str, retry_field_id: str):
    """Handle a failed task with retry logic."""
    task = get_task(task_id)

    # Get current retry count from custom fields
    retry_count = 0
    for cf in task.get("custom_fields", []):
        if cf["id"] == retry_field_id:
            retry_count = int(cf.get("value", 0) or 0)
            break

    retry_count += 1
    set_custom_field(task_id, retry_field_id, retry_count)

    if retry_count >= MAX_RETRIES:
        update_task(task_id, status="failed")
        add_comment(
            task_id,
            f"[AGENT:{agent_id}] [TYPE:ERROR] Max retries ({MAX_RETRIES}) exceeded. "
            f"Last error: {error}"
        )
    else:
        update_task(task_id, status="to do")  # Re-queue
        add_comment(
            task_id,
            f"[AGENT:{agent_id}] [TYPE:ERROR] Attempt {retry_count}/{MAX_RETRIES} "
            f"failed: {error}. Re-queued for retry."
        )
```

---

## Dependency Chains

For sequential agent workflows (e.g., research -> code -> review):

```python
def create_pipeline(
    goal: str,
    steps: list,
    list_id: str = None,
):
    """
    Create a chain of dependent tasks.

    Args:
        goal: High-level objective.
        steps: List of dicts with keys: name, tag, description, priority.
    """
    task_ids = []

    for i, step in enumerate(steps):
        task = create_task(
            name=f"[{i+1}/{len(steps)}] {step['name']}",
            list_id=list_id,
            description=f"Goal: {goal}\n\n{step.get('description', '')}",
            tags=[step["tag"]],
            priority=step.get("priority", 3),
            status="to do" if i == 0 else "blocked",
        )
        task_ids.append(task["id"])

        # Add dependency to previous task
        if i > 0:
            add_dependency(task["id"], depends_on=task_ids[i - 1])

    return task_ids


# Usage
pipeline = create_pipeline(
    goal="Build customer churn prediction model",
    steps=[
        {"name": "Research churn patterns", "tag": "research",
         "description": "Analyze historical churn data..."},
        {"name": "Feature engineering", "tag": "data",
         "description": "Build feature set from transaction data..."},
        {"name": "Train model", "tag": "code",
         "description": "Train XGBoost model with HPO..."},
        {"name": "Review results", "tag": "review",
         "description": "Validate model metrics and business impact..."},
    ]
)
```

### Unblocking Downstream Tasks

When a task completes, the planner (or the completing agent) should unblock the next task:

```python
def complete_and_unblock(task_id: str, agent_id: str, output_summary: str):
    """Mark task done and unblock any dependent tasks."""
    update_task(task_id, status="done")
    add_comment(task_id, f"[AGENT:{agent_id}] [TYPE:OUTPUT] {output_summary}")

    # Find tasks that depend on this one and unblock them
    task = get_task(task_id)
    for dep in task.get("dependencies", []):
        if dep.get("depends_on") == task_id:
            dependent_task_id = dep["task_id"]
            dependent = get_task(dependent_task_id)
            if dependent["status"]["status"] == "blocked":
                update_task(dependent_task_id, status="to do")
                add_comment(
                    dependent_task_id,
                    f"[AGENT:{agent_id}] [TYPE:STATUS] Dependency {task_id} "
                    f"completed. Task unblocked."
                )
```

---

## Polling vs Webhooks

### Polling (Simpler)

Best for: prototypes, low-frequency tasks, agents running as scripts/cron jobs.

```python
# Poll every 30 seconds
while True:
    pending = get_tasks(statuses=["to do"], tags=["research"])
    for task in pending:
        process(task)
    time.sleep(30)
```

### Webhooks (Event-Driven)

Best for: production systems, real-time responsiveness, reducing API calls.

Set up a webhook to listen for `taskCreated` and `taskStatusUpdated` events,
then route to the appropriate agent based on task tags.

```python
# Flask webhook receiver example
from flask import Flask, request

app = Flask(__name__)

@app.route("/webhook", methods=["POST"])
def handle_webhook():
    payload = request.json
    event = payload.get("event")
    task_id = payload.get("task_id")

    if event == "taskCreated":
        task = get_task(task_id)
        tags = [t["name"] for t in task.get("tags", [])]
        route_to_agent(task, tags)

    elif event == "taskStatusUpdated":
        new_status = payload.get("history_items", [{}])[0].get("after", {}).get("status")
        if new_status == "done":
            handle_completion(task_id)

    return "", 200
```

---

## REST API vs MCP Server

### Use REST API When:
- Agents are Python scripts or services you control
- You need full control over request/response handling
- You want to minimize external dependencies
- You need fine-grained error handling and retry logic
- Running in environments without MCP client support

### Use MCP Server When:
- Agents are built on frameworks with native MCP support (Claude Code, Cursor, OpenAI Agents SDK)
- You want natural language task management without writing API calls
- You need fuzzy search across workspace objects
- Rapid prototyping where speed of integration matters more than control

### MCP Server Options

**Official ClickUp MCP:**
- URL: `https://mcp.clickup.com/mcp`
- Auth: OAuth only (no API key)
- Supports: task CRUD, comments, time tracking, docs

**Community (taazkareem):**
- GitHub: `taazkareem/clickup-mcp-server`
- Auth: API key + Team ID (simpler for automation)
- Supports: fuzzy search, bulk operations, custom fields

---

## Complete Workflow Example

End-to-end example: Planner creates a research task, research agent picks it up,
completes it, and the planner creates a follow-up coding task.

```python
# ── planner_agent.py ──

from clickup_client import (
    create_task, get_tasks, get_comments, update_task,
    add_comment, parse_agent_comment
)

def plan_and_monitor():
    # Step 1: Create research task
    task = create_task(
        name="Research: Best embedding models for RAG in 2026",
        description=(
            "Compare top 5 embedding models for retrieval-augmented generation.\n"
            "Consider: accuracy, latency, cost, context window.\n"
            "Output: comparison table + recommendation."
        ),
        tags=["research"],
        priority=2,
    )
    research_task_id = task["id"]
    print(f"Created research task: {research_task_id}")

    # Step 2: Monitor until done
    import time
    while True:
        t = get_task(research_task_id)
        status = t["status"]["status"]

        if status == "done":
            # Get the output from comments
            comments = get_comments(research_task_id)
            for c in comments:
                parsed = parse_agent_comment(c["comment_text"])
                if parsed["type"] == "OUTPUT":
                    print(f"Research complete: {parsed['body']}")
                    break

            # Step 3: Create follow-up coding task
            code_task = create_task(
                name="Implement: RAG pipeline with recommended model",
                description=f"Based on research task #{research_task_id}, implement...",
                tags=["code"],
                priority=2,
            )
            print(f"Created coding task: {code_task['id']}")
            break

        elif status == "failed":
            print("Research task failed. Check error comments.")
            break

        time.sleep(30)


# ── research_agent.py ──

from clickup_client import (
    get_tasks, get_task, update_task, add_comment, claim_task
)

def research_process(task: dict) -> dict:
    """Actual research logic."""
    # ... do the research work ...
    return {
        "summary": "Recommended: text-embedding-3-large. "
                   "Best accuracy/cost ratio. Full comparison attached."
    }

def run():
    agent_id = "research-agent-01"
    while True:
        tasks = get_tasks(statuses=["to do"])
        research_tasks = [
            t for t in tasks
            if any(tag["name"] == "research" for tag in t.get("tags", []))
        ]

        for task in research_tasks:
            if not claim_task(task["id"], agent_id):
                continue

            update_task(task["id"], status="in progress")
            try:
                result = research_process(task)
                add_comment(
                    task["id"],
                    f"[AGENT:{agent_id}] [TYPE:OUTPUT] {result['summary']}"
                )
                update_task(task["id"], status="done")
            except Exception as e:
                add_comment(
                    task["id"],
                    f"[AGENT:{agent_id}] [TYPE:ERROR] {str(e)}"
                )
                update_task(task["id"], status="failed")

        import time
        time.sleep(30)
```
