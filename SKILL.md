---
name: clickup
description: >
  ClickUp API integration skill for agent-to-agent task management. Use this skill whenever an agent
  needs to create, read, update, or manage tasks in ClickUp as part of a multi-agent workflow. This
  includes: creating tasks for other agents to pick up, updating task status/progress, adding comments
  for inter-agent communication, querying task lists for pending work, managing dependencies between
  tasks, and navigating the ClickUp hierarchy (Workspace, Space, Folder, List, Task). Also use
  when the user mentions ClickUp, task boards, agent task queues, or multi-agent orchestration via
  project management tools. Covers both direct REST API usage and MCP server integration patterns.
---

# ClickUp Integration Skill

A Python-based toolkit for integrating agents with ClickUp's REST API v2 for task management,
inter-agent communication, and workflow orchestration.

## When to Use This Skill

- A planner agent needs to create tasks for sub-agents
- A sub-agent needs to pick up, update status, or mark tasks complete
- Agents need to communicate via task comments
- You need to query pending/in-progress/completed tasks
- You need to set up the ClickUp workspace hierarchy for an agent system
- You need to manage task dependencies between agent steps

## Quick Start

Read `references/api_reference.md` for the full endpoint catalog and Python examples.
Read `references/agent_patterns.md` for multi-agent orchestration patterns and conventions.

## Architecture Overview

### ClickUp Hierarchy

```
Workspace (team_id in API v2)
  └── Space (space_id)
       └── Folder (folder_id)  [optional]
            └── List (list_id)
                 └── Task (task_id)
                      └── Subtask (task with parent)
                      └── Comments
                      └── Custom Fields
```

### Authentication

All requests require an `Authorization` header with a personal API token or OAuth access token.

```python
HEADERS = {
    "Authorization": "YOUR_API_TOKEN",
    "Content-Type": "application/json"
}
```

Store the token in an environment variable:

```python
import os
CLICKUP_API_TOKEN = os.environ.get("CLICKUP_API_TOKEN")
CLICKUP_TEAM_ID = os.environ.get("CLICKUP_TEAM_ID")  # workspace ID
CLICKUP_LIST_ID = os.environ.get("CLICKUP_LIST_ID")   # default task list
```

### Rate Limits

- 100 requests per minute per token (varies by plan)
- Handle 429 responses with exponential backoff

## Core Client Module

Create a reusable `clickup_client.py` that all agents import:

```python
"""
clickup_client.py - Shared ClickUp API client for multi-agent task management.

Environment variables required:
    CLICKUP_API_TOKEN  - Personal API token or OAuth token
    CLICKUP_TEAM_ID    - Workspace ID (called team_id in API v2)
    CLICKUP_LIST_ID    - Default list ID for task creation
"""

import os
import time
import requests
from typing import Optional

BASE_URL = "https://api.clickup.com/api/v2"


def _headers() -> dict:
    token = os.environ["CLICKUP_API_TOKEN"]
    return {
        "Authorization": token,
        "Content-Type": "application/json"
    }


def _request(method: str, path: str, **kwargs) -> dict:
    """Make an API request with retry on rate limit."""
    url = f"{BASE_URL}{path}"
    for attempt in range(3):
        resp = getattr(requests, method)(url, headers=_headers(), **kwargs)
        if resp.status_code == 429:
            wait = 2 ** attempt
            time.sleep(wait)
            continue
        resp.raise_for_status()
        return resp.json() if resp.text else {}
    raise Exception(f"Rate limited after 3 retries: {method.upper()} {path}")


# ──────────────────────────────────────────────
# Workspace Navigation
# ──────────────────────────────────────────────

def get_workspaces() -> list:
    """List all workspaces (teams) accessible to the authenticated user."""
    return _request("get", "/team")["teams"]


def get_spaces(team_id: Optional[str] = None) -> list:
    """List spaces in a workspace."""
    tid = team_id or os.environ["CLICKUP_TEAM_ID"]
    return _request("get", f"/team/{tid}/space")["spaces"]


def get_folders(space_id: str) -> list:
    """List folders in a space."""
    return _request("get", f"/space/{space_id}/folder")["folders"]


def get_lists(folder_id: str) -> list:
    """List lists in a folder."""
    return _request("get", f"/folder/{folder_id}/list")["lists"]


def get_folderless_lists(space_id: str) -> list:
    """List lists that are directly under a space (no folder)."""
    return _request("get", f"/space/{space_id}/list")["lists"]


# ──────────────────────────────────────────────
# Task CRUD
# ──────────────────────────────────────────────

def create_task(
    name: str,
    list_id: Optional[str] = None,
    description: str = "",
    status: str = "to do",
    priority: Optional[int] = None,
    assignees: Optional[list] = None,
    tags: Optional[list] = None,
    due_date: Optional[int] = None,
    parent: Optional[str] = None,
    custom_fields: Optional[list] = None,
    markdown_description: Optional[str] = None,
) -> dict:
    """
    Create a task in a list.

    Args:
        name: Task name (required).
        list_id: Target list ID. Falls back to CLICKUP_LIST_ID env var.
        description: Plain text description.
        status: Task status string (must match list's configured statuses).
        priority: 1=Urgent, 2=High, 3=Normal, 4=Low.
        assignees: List of user IDs.
        tags: List of tag strings.
        due_date: Unix timestamp in milliseconds.
        parent: Parent task ID to create a subtask.
        custom_fields: List of {"id": field_id, "value": value} dicts.
        markdown_description: Markdown-formatted description (overrides description).

    Returns:
        Created task dict with id, name, status, url, etc.
    """
    lid = list_id or os.environ["CLICKUP_LIST_ID"]
    body = {"name": name, "status": status}
    if markdown_description:
        body["markdown_description"] = markdown_description
    elif description:
        body["description"] = description
    if priority is not None:
        body["priority"] = priority
    if assignees:
        body["assignees"] = assignees
    if tags:
        body["tags"] = tags
    if due_date:
        body["due_date"] = due_date
    if parent:
        body["parent"] = parent
    if custom_fields:
        body["custom_fields"] = custom_fields
    return _request("post", f"/list/{lid}/task", json=body)


def get_task(task_id: str) -> dict:
    """Get full details of a single task."""
    return _request("get", f"/task/{task_id}")


def update_task(task_id: str, **fields) -> dict:
    """
    Update a task. Only pass fields you want to change.

    Common fields: name, description, status, priority, due_date,
                   assignees (dict with "add" and "rem" lists).

    Note: Custom fields cannot be updated here. Use set_custom_field().
    """
    return _request("put", f"/task/{task_id}", json=fields)


def delete_task(task_id: str) -> dict:
    """Delete a task permanently."""
    return _request("delete", f"/task/{task_id}")


def get_tasks(
    list_id: Optional[str] = None,
    statuses: Optional[list] = None,
    assignees: Optional[list] = None,
    page: int = 0,
    include_closed: bool = False,
    subtasks: bool = True,
) -> list:
    """
    Get tasks from a list (paginated, 100 per page).

    Args:
        list_id: List to query. Falls back to CLICKUP_LIST_ID.
        statuses: Filter by status strings, e.g. ["to do", "in progress"].
        assignees: Filter by user IDs.
        page: Page number (0-indexed).
        include_closed: Include closed/completed tasks.
        subtasks: Include subtasks in results.

    Returns:
        List of task dicts.
    """
    lid = list_id or os.environ["CLICKUP_LIST_ID"]
    params = {"page": page, "subtasks": str(subtasks).lower(),
              "include_closed": str(include_closed).lower()}
    if statuses:
        params["statuses[]"] = statuses
    if assignees:
        params["assignees[]"] = assignees
    return _request("get", f"/list/{lid}/task", params=params)["tasks"]


def get_filtered_tasks(
    team_id: Optional[str] = None,
    statuses: Optional[list] = None,
    assignees: Optional[list] = None,
    tags: Optional[list] = None,
    page: int = 0,
) -> list:
    """
    Search tasks across the entire workspace (not limited to one list).

    Useful for agents finding tasks assigned to them across all lists.
    """
    tid = team_id or os.environ["CLICKUP_TEAM_ID"]
    params = {"page": page}
    if statuses:
        params["statuses[]"] = statuses
    if assignees:
        params["assignees[]"] = assignees
    if tags:
        params["tags[]"] = tags
    return _request("get", f"/team/{tid}/task", params=params)["tasks"]


# ──────────────────────────────────────────────
# Comments (Inter-Agent Communication)
# ──────────────────────────────────────────────

def add_comment(task_id: str, comment_text: str, notify_all: bool = False) -> dict:
    """
    Add a comment to a task. Used for agent-to-agent messaging.

    Args:
        task_id: Target task.
        comment_text: Plain text comment body.
        notify_all: Send notifications to all assignees.

    Returns:
        Comment response dict.
    """
    body = {"comment_text": comment_text, "notify_all": notify_all}
    return _request("post", f"/task/{task_id}/comment", json=body)


def get_comments(task_id: str) -> list:
    """Get comments on a task (newest first, 25 per page)."""
    return _request("get", f"/task/{task_id}/comment")["comments"]


# ──────────────────────────────────────────────
# Custom Fields
# ──────────────────────────────────────────────

def get_list_custom_fields(list_id: Optional[str] = None) -> list:
    """Get available custom fields for a list."""
    lid = list_id or os.environ["CLICKUP_LIST_ID"]
    return _request("get", f"/list/{lid}/field")["fields"]


def set_custom_field(task_id: str, field_id: str, value) -> dict:
    """Set a custom field value on a task."""
    return _request("post", f"/task/{task_id}/field/{field_id}", json={"value": value})


# ──────────────────────────────────────────────
# Dependencies
# ──────────────────────────────────────────────

def add_dependency(task_id: str, depends_on: str) -> dict:
    """Make task_id depend on (wait for) depends_on task."""
    body = {"depends_on": depends_on}
    return _request("post", f"/task/{task_id}/dependency", json=body)


def remove_dependency(task_id: str, depends_on: str) -> dict:
    """Remove a dependency relationship."""
    params = {"depends_on": depends_on}
    return _request("delete", f"/task/{task_id}/dependency", params=params)


# ──────────────────────────────────────────────
# Tags
# ──────────────────────────────────────────────

def add_tag_to_task(task_id: str, tag_name: str) -> dict:
    """Add a tag to a task."""
    return _request("post", f"/task/{task_id}/tag/{tag_name}")


def remove_tag_from_task(task_id: str, tag_name: str) -> dict:
    """Remove a tag from a task."""
    return _request("delete", f"/task/{task_id}/tag/{tag_name}")
```

## Suggested ClickUp Workspace Setup for Agents

Set up a dedicated Space with a single List (or one List per agent type):

```
Workspace: "OpenClaw Agents"
  └── Space: "Agent Tasks"
       └── List: "Task Queue"
            Statuses: to do | picked up | in progress | blocked | review | done | failed
```

### Recommended Custom Fields

| Field Name       | Type     | Purpose                                       |
|------------------|----------|-----------------------------------------------|
| `agent_id`       | Text     | Which agent created or owns this task          |
| `agent_type`     | Dropdown | planner, researcher, coder, reviewer, etc.     |
| `output_ref`     | Text     | Path/URL to agent output artifact              |
| `retry_count`    | Number   | How many times this task has been retried       |
| `error_log`      | Text     | Last error message if task failed               |
| `parent_goal`    | Text     | High-level objective this task serves           |

### Recommended Tags for Agent Routing

Use tags to let sub-agents filter for tasks they should handle:
`research`, `code`, `review`, `deploy`, `data`, `report`, `urgent`

## MCP Server Integration (Alternative)

ClickUp provides an official MCP server at `https://mcp.clickup.com/mcp` that supports OAuth.
For agent frameworks that support MCP (Claude Code, Cursor, OpenAI Agents SDK), this can be
an alternative to direct REST API calls.

Third-party MCP servers also exist (e.g., `taazkareem/clickup-mcp-server`) that support
API key auth and provide fuzzy search across workspace objects.

See `references/agent_patterns.md` for when to use MCP vs REST.

## Error Handling Patterns

```python
from requests.exceptions import HTTPError

try:
    task = create_task("Process data batch #42", tags=["data"])
except HTTPError as e:
    if e.response.status_code == 429:
        # Rate limited - back off and retry
        pass
    elif e.response.status_code == 401:
        # Auth failed - check token
        pass
    elif e.response.status_code == 404:
        # List/task not found - check IDs
        pass
    else:
        raise
```
