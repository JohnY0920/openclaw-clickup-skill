# OpenClaw ClickUp Skill

A Python-based toolkit for integrating AI agents with [ClickUp](https://clickup.com/)'s REST API v2 for task management, inter-agent communication, and multi-agent workflow orchestration.

## Overview

This skill enables AI agents to:
- Create, read, update, and manage tasks in ClickUp
- Communicate via task comments for agent-to-agent coordination
- Query task lists for pending work
- Manage dependencies between tasks
- Navigate ClickUp's hierarchy (Workspace → Space → Folder → List → Task)

Perfect for building multi-agent systems where planner agents delegate work to sub-agents through a shared project management interface.

## Quick Start

### 1. Installation

```bash
# Clone to your skills directory
git clone https://github.com/JohnY0920/openclaw-clickup-skill.git ~/.claw/skills/clickup-skill

# Or copy to your project's skills folder
cp -r openclaw-clickup-skill ./skills/
```

### 2. Configuration

Set these environment variables:

```bash
export CLICKUP_API_TOKEN="your_personal_api_token"
export CLICKUP_TEAM_ID="your_workspace_id"
export CLICKUP_LIST_ID="your_default_list_id"
```

**Getting your credentials:**
- **API Token**: ClickUp Settings → Apps → Generate API Token
- **Team ID**: Use the [Get Teams](https://clickup.com/api/clickupreference/operation/GetAuthorizedTeams/) endpoint or check URL when viewing workspace
- **List ID**: Check URL when viewing a list (`/li/{list_id}`) or use API to list lists

### 3. Usage

```python
from clickup_client import create_task, get_tasks, update_task, add_comment

# Create a task for another agent
task = create_task(
    name="Research competitor pricing",
    description="Find pricing for top 3 competitors",
    status="to do",
    tags=["research", "urgent"],
    assignees=[12345678]  # User ID of the agent
)
print(f"Created task: {task['url']}")

# Pick up pending tasks
pending = get_tasks(statuses=["to do"], tags=["research"])
for t in pending:
    print(f"- {t['name']} (ID: {t['id']})")

# Update progress
update_task(task_id="123", status="in progress")

# Communicate via comments
add_comment(task_id="123", comment_text="Found 2/3 competitors, waiting on API access for third")
```

## ClickUp Hierarchy

```
Workspace (team_id)
  └── Space (space_id)
       └── Folder (folder_id) [optional]
            └── List (list_id)
                 └── Task (task_id)
                      ├── Subtask (task with parent)
                      ├── Comments
                      └── Custom Fields
```

## Recommended Workspace Setup for Agents

Create a dedicated structure for agent workflows:

```
Workspace: "OpenClaw Agents"
  └── Space: "Agent Tasks"
       └── List: "Task Queue"
```

### Suggested Status Flow
```
to do → picked up → in progress → review → done
         ↓
      blocked → failed
```

### Suggested Custom Fields

| Field | Type | Purpose |
|-------|------|---------|
| `agent_id` | Text | Which agent owns the task |
| `agent_type` | Dropdown | planner, researcher, coder, reviewer |
| `output_ref` | Text | Path to agent output artifact |
| `retry_count` | Number | Retry attempts |
| `error_log` | Text | Last error if failed |
| `parent_goal` | Text | High-level objective |

### Suggested Tags

Route tasks by capability:
- `research` - Data gathering tasks
- `code` - Programming tasks
- `review` - Code/content review
- `deploy` - Deployment tasks
- `data` - Data processing
- `report` - Report generation
- `urgent` - High priority

## Core Functions

### Task Management
- `create_task()` - Create new tasks with metadata
- `get_task()` - Get task details
- `update_task()` - Update status, assignees, etc.
- `delete_task()` - Remove tasks
- `get_tasks()` - List tasks from a list
- `get_filtered_tasks()` - Search across workspace

### Comments (Agent Communication)
- `add_comment()` - Add comment to task
- `get_comments()` - Read task comments

### Navigation
- `get_workspaces()` - List accessible workspaces
- `get_spaces()` - List spaces in workspace
- `get_folders()` - List folders in space
- `get_lists()` - Lists in a folder
- `get_folderless_lists()` - Lists directly under space

### Advanced
- `add_dependency()` - Set task dependencies
- `get_list_custom_fields()` - Get available custom fields
- `set_custom_field()` - Update custom field values
- `add_tag_to_task()` / `remove_tag_from_task()` - Tag management

## Rate Limits

- 100 requests per minute per token (varies by plan)
- Built-in retry with exponential backoff for 429 responses

## Error Handling

```python
from requests.exceptions import HTTPError

try:
    task = create_task("Process batch #42")
except HTTPError as e:
    if e.response.status_code == 429:
        # Rate limited - will auto-retry
        pass
    elif e.response.status_code == 401:
        # Check your CLICKUP_API_TOKEN
        pass
    elif e.response.status_code == 404:
        # Verify team/list/task IDs
        pass
```

## MCP Server Alternative

For frameworks that support [Model Context Protocol](https://modelcontextprotocol.io/) (Claude Code, Cursor, etc.), ClickUp provides an official MCP server:
- **URL**: `https://mcp.clickup.com/mcp`
- **Auth**: OAuth

Third-party MCP servers with API key support:
- `taazkareem/clickup-mcp-server` - Fuzzy search, API key auth

Use MCP when your agent framework supports it; use this REST skill for direct API control.

## Documentation

- `SKILL.md` - Full technical documentation
- `references/api_reference.md` - Complete endpoint catalog
- `references/agent_patterns.md` - Multi-agent orchestration patterns

## Contributing

This skill was created for [OpenClaw](https://github.com/openclaw/openclaw) agent orchestration. Feel free to fork and adapt for your agent framework.

## License

MIT License - See repository for details.

---

**Maintained by**: [John Yin](https://github.com/JohnY0920)  
**Part of**: [Synervex](https://github.com/JohnY0920/synervex_website) AI consulting
