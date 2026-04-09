---
name: api-doc
description: "Extract API documentation from git changes and post to Jira. Use when: summarizing new or changed API endpoints for FE and QA teams; documenting API changes from a branch; posting endpoint specs (method, path, request params, response schema) as a Jira ticket comment. Triggers: 'document APIs', 'API doc from git', 'post API spec to Jira', 'what APIs changed', 'generate API docs for ticket'."
argument-hint: 'Optional: branch name or ticket ID override (e.g. MC-9056)'
---

# API Doc from Git Changes

Extracts new and modified API endpoints from git changes and posts structured documentation as a Jira comment for FE and QA consumption.

## When to Use

- A feature branch adds or modifies API endpoints
- FE or QA needs endpoint specs before integration or test writing
- You want to document APIs tied to a specific Jira ticket

---

## Procedure

### Step 1 — Extract Ticket ID from Branch Name

Run:

```bash
git branch --show-current
```

Parse the ticket ID using this pattern:

- Match the first occurrence of `[A-Z]+-[0-9]+` in the branch name
- Example: `EPIC-MC-9056-PDAM-CREDIT` → ticket ID = `MC-9056`
- Example: `feat/MC-1234-some-feature` → ticket ID = `MC-1234`

If the user provided an override argument (e.g. `MC-9056`), use that instead.

If no ticket ID can be found, ask the user to provide it.

---

### Step 1.5 — Fetch Jira Ticket Context

Use `mcp_atlassian_mcp_getJiraIssue` to pull the ticket context:

```
tool: mcp_atlassian_mcp_getJiraIssue
params:
  issue_key: "MC-9056"   # extracted ticket ID
```

From the response, extract:

- **Summary** — ticket title (use as section heading in the comment)
- **Description** — read to understand the feature scope (helps filter unrelated API changes)
- **Assignee / Reporter** — optional context for the comment header

If the ticket is not found or MCP returns an error, continue with git-only context and note the issue in the comment header.

---

### Step 2 — Get Git Diff Against Master

Run:

```bash
git diff master...HEAD --name-only
```

Then get the full diff for relevant files only:

```bash
git diff master...HEAD -- config/routes.rb app/controllers/ spec/swagger/
```

Focus on:

- `config/routes.rb` — new route definitions
- `app/controllers/**/*.rb` — new actions, changed params, render calls
- `spec/swagger/**/*_spec.rb` — rswag specs (most authoritative source)

---

### Step 3 — Identify Changed APIs

**Priority order** (use the first available source):

1. **Swagger specs** (`spec/swagger/**`) — most complete: has path, method, parameters, responses
2. **Routes diff** — reveals new `path`, `resources`, `get/post/put/patch/delete` entries
3. **Controller diff** — reveals new action methods and `render json:` responses

For each identified endpoint, collect:
| Field | Source |
|-------|--------|
| HTTP Method | swagger `get/post/put/patch/delete`, or routes verb |
| Path | swagger `path '/api/...'`, or routes `namespace`/`resources` |
| Summary | swagger `summary` or controller action name |
| Tags / Group | swagger `tags` |
| Auth required | swagger `security` block |
| Request headers | swagger `parameter name: ..., in: :header` |
| Request body/query params | swagger `parameter name: ..., in: :body/:query` |
| Response codes | swagger `response '200'`, `response '422'`, etc. |
| Response schema | swagger `schema` reference or controller `render json:` |

If swagger specs are not present, derive from routes + controllers as best effort.

---

### Step 4 — Format the Documentation

Produce one section per endpoint. Use this format:

```
## [METHOD] /api/path/to/endpoint

**Summary:** Brief description of what it does
**Tags:** Comma-separated tag names
**Auth:** Required headers (e.g. X-IAG-API-KEY, X-Authenticated-Userid)

### Request

#### Headers
| Name | Type | Required | Description |
|------|------|----------|-------------|
| X-IAG-API-KEY | string | ✅ | IAG API key |

#### Query / Body Parameters
| Name | Type | Required | Description |
|------|------|----------|-------------|
| amount | integer | ✅ | Transaction amount in IDR |
| bank_code | string | ✅ | Destination bank code |

**Example Request Body:**
\`\`\`json
{
  "amount": 500000,
  "bank_code": "BCA"
}
\`\`\`

### Response

#### 200 — Success
| Field | Type | Description |
|-------|------|-------------|
| data.id | string | Transaction ID |
| data.type | string | Resource type |
| data.attributes.status | string | Transaction status |

**Example Response:**
\`\`\`json
{
  "data": {
    "id": "123",
    "type": "transaction",
    "attributes": {
      "status": "created",
      "amount": 500000
    }
  }
}
\`\`\`

#### 401 — Unauthorized
Standard error response when API key is missing or invalid.

#### 422 — Unprocessable Entity
Validation error response.
```

If multiple endpoints are found, list all of them, each with their own `##` section.

Start the full comment with:

```
## 📋 API Documentation — [Branch Name]

> Auto-generated from git diff against `master`.
> Ticket: [MC-XXXX]
> Generated: [today's date]

---
```

---

### Step 5 — Post to Jira

Use `mcp_atlassian_mcp_addCommentToJiraIssue` to post the documentation:

```
tool: mcp_atlassian_mcp_addCommentToJiraIssue
params:
  issue_key: "MC-9056"        # extracted ticket ID
  comment_body: "<full markdown from Step 4>"
```

Before posting:

1. Show the user a preview of the full comment
2. Ask for confirmation: "Post this API doc to [MC-9056]?"
3. Only call the tool after confirmation

After posting, share the Jira ticket URL in the response.

---

## Output

A Jira comment on the ticket containing:

- All new/changed API endpoints
- Method, path, auth requirements
- Request parameter table + JSON example
- Response schema table + JSON example per status code

---

## Notes

- **Swagger specs are the source of truth.** If a swagger spec exists for the changed controller, always prefer it over inferring from routes/controller code.
- **Scope tightly**: only document endpoints whose files appear in `git diff master...HEAD --name-only`. Do not document unchanged endpoints.
- **FE/QA audience**: use plain language in summaries and avoid Rails-internal terminology.
- **Handle missing info gracefully**: if response schema isn't inferrable, note `Schema: not documented — infer from implementation`.
