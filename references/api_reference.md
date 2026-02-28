# ClickUp API v2 Reference

Complete endpoint reference for the ClickUp REST API v2 used in agent workflows.

**Base URL:** `https://api.clickup.com/api/v2`

**Authentication:** Include `Authorization: {token}` header on every request.
Personal tokens look like `pk_4753994_EXP7MPOJ7XQM5UJDV2M45MPF0YHH5YHO`.

**Rate Limits:** 100 requests/minute/token. Returns 429 on exceed.

**Dates:** Unix timestamps in milliseconds throughout.

**API v2 vs v3 Note:** In v2, "Team" = "Workspace". v3 uses "Workspace" consistently.
Most endpoints are v2. Some newer endpoints use v3.

---

## Table of Contents

1. [Workspace & Navigation](#workspace--navigation)
2. [Task CRUD](#task-crud)
3. [Comments](#comments)
4. [Custom Fields](#custom-fields)
5. [Dependencies & Links](#dependencies--links)
6. [Tags](#tags)
7. [Webhooks](#webhooks)
8. [Statuses](#statuses)
9. [Members](#members)
10. [Common Patterns](#common-patterns)

---

## Workspace & Navigation

### Get Workspaces (Teams)
```
GET /team
```
Returns all workspaces the token has access to.

```python
resp = requests.get(f"{BASE}/team", headers=headers)
teams = resp.json()["teams"]
# Each team: {"id": "123", "name": "My Workspace", "members": [...]}
```

### Get Spaces
```
GET /team/{team_id}/space
Query: archived=false
```

### Get Folders
```
GET /space/{space_id}/folder
Query: archived=false
```

### Get Lists (in Folder)
```
GET /folder/{folder_id}/list
Query: archived=false
```

### Get Folderless Lists (directly in Space)
```
GET /space/{space_id}/list
Query: archived=false
```

---

## Task CRUD

### Create Task
```
POST /list/{list_id}/task
```

**Body fields:**

| Field                  | Type      | Required | Notes                                        |
|------------------------|-----------|----------|----------------------------------------------|
| `name`                 | string    | Yes      | Task title                                   |
| `description`          | string    | No       | Plain text description                       |
| `markdown_description` | string    | No       | Markdown description (overrides description) |
| `status`               | string    | No       | Must match list's configured statuses        |
| `priority`             | int       | No       | 1=Urgent, 2=High, 3=Normal, 4=Low           |
| `assignees`            | int[]     | No       | Array of user IDs                            |
| `tags`                 | string[]  | No       | Array of tag names                           |
| `due_date`             | int       | No       | Unix timestamp in milliseconds               |
| `start_date`           | int       | No       | Unix timestamp in milliseconds               |
| `time_estimate`        | int       | No       | Milliseconds                                 |
| `parent`               | string    | No       | Parent task ID (creates subtask)             |
| `links_to`             | string    | No       | Task ID to create dependency link            |
| `custom_fields`        | object[]  | No       | [{"id": "field_id", "value": "..."}]         |
| `notify_all`           | bool      | No       | Send creation notifications                  |
| `points`               | float     | No       | Story points / effort                        |

```python
task_data = {
    "name": "Analyze dataset #42",
    "description": "Run feature engineering pipeline on batch #42",
    "status": "to do",
    "priority": 3,
    "tags": ["data", "research"],
    "custom_fields": [
        {"id": "abc123", "value": "planner-agent-001"}
    ]
}
resp = requests.post(
    f"{BASE}/list/{list_id}/task",
    headers=headers,
    json=task_data
)
new_task = resp.json()
task_id = new_task["id"]
```

### Get Task
```
GET /task/{task_id}
Query: custom_task_ids=false, include_subtasks=true
```

Returns full task details including status, assignees, custom_fields, dependencies,
linked_tasks, tags, comments count, etc.

### Update Task
```
PUT /task/{task_id}
```

Only pass fields you want to change. Supports: `name`, `description`, `status`,
`priority`, `due_date`, `start_date`, `time_estimate`, `assignees`, `archived`.

**Assignee updates use add/remove pattern:**
```python
update_data = {
    "status": "in progress",
    "assignees": {
        "add": [user_id],
        "rem": []
    }
}
requests.put(f"{BASE}/task/{task_id}", headers=headers, json=update_data)
```

**Important:** Custom fields CANNOT be updated via this endpoint. Use Set Custom Field Value.

### Delete Task
```
DELETE /task/{task_id}
```

### Get Tasks (from List)
```
GET /list/{list_id}/task
```

**Query parameters:**

| Param             | Type      | Notes                                     |
|-------------------|-----------|-------------------------------------------|
| `page`            | int       | 0-indexed, 100 tasks per page             |
| `statuses[]`      | string[]  | Filter by statuses                        |
| `assignees[]`     | int[]     | Filter by assignee user IDs               |
| `tags[]`          | string[]  | Filter by tags                            |
| `include_closed`  | bool      | Include completed tasks                   |
| `subtasks`        | bool      | Include subtasks                          |
| `order_by`        | string    | created, updated, due_date                |
| `reverse`         | bool      | Reverse sort order                        |
| `due_date_gt`     | int       | Due date greater than (ms timestamp)      |
| `due_date_lt`     | int       | Due date less than (ms timestamp)         |
| `date_created_gt` | int       | Created after (ms timestamp)              |
| `date_updated_gt` | int       | Updated after (ms timestamp)              |

```python
params = {
    "statuses[]": ["to do"],
    "tags[]": ["research"],
    "page": 0,
    "subtasks": "true"
}
resp = requests.get(f"{BASE}/list/{list_id}/task", headers=headers, params=params)
tasks = resp.json()["tasks"]
```

### Get Filtered Team Tasks (Cross-List Search)
```
GET /team/{team_id}/task
```

Search tasks across the entire workspace. Same query params as Get Tasks plus
additional filters. Useful for agents finding their tasks across all lists.

---

## Comments

### Create Task Comment
```
POST /task/{task_id}/comment
```

Body:
```json
{
    "comment_text": "Agent status update: processing complete, output at /results/42.json",
    "notify_all": false
}
```

### Get Task Comments
```
GET /task/{task_id}/comment
Query: start (timestamp), start_id (comment ID for pagination)
```

Returns newest 25 comments by default. Use `start` + `start_id` from the last
comment to paginate backward.

### Update Comment
```
PUT /comment/{comment_id}
Body: {"comment_text": "updated text", "resolved": true}
```

### Delete Comment
```
DELETE /comment/{comment_id}
```

---

## Custom Fields

### Get List Custom Fields
```
GET /list/{list_id}/field
```

Returns all custom field definitions for a list, including `id`, `name`, `type`,
and `type_config` (options for dropdowns, etc.).

### Set Custom Field Value
```
POST /task/{task_id}/field/{field_id}
Body: {"value": "the value"}
```

Value format depends on field type:
- **Text/URL/Email/Phone:** string
- **Number/Currency/Percentage:** number
- **Dropdown:** option index (int) or option id (string)
- **Checkbox:** true/false
- **Date:** Unix timestamp in ms
- **Labels:** array of label IDs
- **Users:** array of user IDs

### Remove Custom Field Value
```
DELETE /task/{task_id}/field/{field_id}
```

---

## Dependencies & Links

### Add Dependency
```
POST /task/{task_id}/dependency
Body: {"depends_on": "other_task_id"}    # task_id waits for other_task_id
  OR: {"dependency_of": "other_task_id"} # task_id blocks other_task_id
```

### Delete Dependency
```
DELETE /task/{task_id}/dependency
Query: depends_on={other_task_id} OR dependency_of={other_task_id}
```

### Add Task Link
```
POST /task/{task_id}/link/{links_to_task_id}
```

### Delete Task Link
```
DELETE /task/{task_id}/link/{links_to_task_id}
```

---

## Tags

### Get Space Tags
```
GET /space/{space_id}/tag
```

### Add Tag to Task
```
POST /task/{task_id}/tag/{tag_name}
```

### Remove Tag from Task
```
DELETE /task/{task_id}/tag/{tag_name}
```

### Create Space Tag
```
POST /space/{space_id}/tag
Body: {"tag": {"name": "tag_name"}}
```

---

## Webhooks

### Create Webhook
```
POST /team/{team_id}/webhook
```

```python
webhook_data = {
    "endpoint": "https://your-endpoint.com/webhook",
    "events": [
        "taskCreated",
        "taskUpdated",
        "taskStatusUpdated",
        "taskCommentPosted",
        "taskDeleted"
    ]
}
```

**Available events:** taskCreated, taskUpdated, taskDeleted, taskPriorityUpdated,
taskStatusUpdated, taskAssigneeUpdated, taskDueDateUpdated, taskTagUpdated,
taskMoved, taskCommentPosted, taskCommentUpdated, taskTimeEstimateUpdated,
taskTimeTrackedUpdated, listCreated, listUpdated, listDeleted, folderCreated,
folderUpdated, folderDeleted, spaceCreated, spaceUpdated, spaceDeleted,
goalCreated, goalUpdated, goalDeleted, keyResultCreated, keyResultUpdated,
keyResultDeleted

### Get Webhooks
```
GET /team/{team_id}/webhook
```

### Update Webhook
```
PUT /webhook/{webhook_id}
Body: {"endpoint": "...", "events": [...], "status": "active"}
```

### Delete Webhook
```
DELETE /webhook/{webhook_id}
```

---

## Statuses

Task statuses are configured per List. To see available statuses, get the list details:

```
GET /list/{list_id}
```

The response includes a `statuses` array:
```json
{
    "statuses": [
        {"status": "to do", "type": "open", "orderindex": 0},
        {"status": "in progress", "type": "custom", "orderindex": 1},
        {"status": "review", "type": "custom", "orderindex": 2},
        {"status": "complete", "type": "closed", "orderindex": 3}
    ]
}
```

When setting status on a task, the string must exactly match one of these values.

---

## Members

### Get Task Members
```
GET /task/{task_id}/member
```

### Get List Members
```
GET /list/{list_id}/member
```

---

## Common Patterns

### Pagination
Most list endpoints return 100 items max. Use `page` parameter (0-indexed):

```python
all_tasks = []
page = 0
while True:
    batch = get_tasks(list_id=lid, page=page)
    if not batch:
        break
    all_tasks.extend(batch)
    page += 1
```

### Timestamp Conversion
ClickUp uses millisecond Unix timestamps:

```python
import time
from datetime import datetime

# Python datetime -> ClickUp timestamp
due = int(datetime(2026, 3, 15, 17, 0).timestamp() * 1000)

# ClickUp timestamp -> Python datetime
dt = datetime.fromtimestamp(int("1592452913384") / 1000)
```

### Task ID Formatting
The ClickUp GUI shows task IDs with a `#` prefix (e.g., `#8ckjp5`).
The API expects IDs without the hash: `8ckjp5`.

```python
def clean_task_id(task_id: str) -> str:
    return task_id.lstrip("#")
```
