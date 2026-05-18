---
name: "confluence-reader"
description: "Reads and parses content from Confluence pages using VS Code MCP plugin. Invoke when user needs to fetch feature documents, requirements, or testing specifications from Confluence."
---

# Confluence Reader Skill

This skill reads content from Confluence pages using the **VS Code Confluence MCP plugin** and prepares it for further analysis.

## Prerequisites

Ensure the VS Code Confluence MCP plugin is configured in `.vscode/mcp.json`:

```json
{
  "servers": {
    "confluence": {
      "type": "stdio",
      "command": "npx",
      "args": [
        "-y",
        "@mcp-confluence/server"
      ],
      "env": {
        "CONFLUENCE_URL": "https://your-domain.atlassian.net/wiki",
        "CONFLUENCE_API_TOKEN": "${CONFLUENCE_API_TOKEN}",
        "CONFLUENCE_USERNAME": "${CONFLUENCE_EMAIL}"
      }
    }
  }
}
```

## Available MCP Tools

The Confluence MCP plugin provides these tools for reading content:

| Tool Name | Description | Use Case |
|-----------|-------------|----------|
| `confluence_get_page` | Get page by ID | Read a specific feature doc |
| `confluence_get_page_by_title` | Get page by title + space | Search by document title |
| `confluence_get_child_pages` | Get child pages | Read EPIC -> Stories hierarchy |
| `confluence_get_space_content` | Get space content | List all pages in a space |
| `confluence_search` | Search by CQL | Advanced search like `label=testing` |

## Reading Flow

### Step 1: Identify the Target Page

Determine which Confluence page(s) to read. Options:

```
a) By Page ID:     "Read Confluence page ID 123456"
b) By Title:       "Find the 'Login Feature' page in TEST space"
c) By Label:       "Find all pages with label 'sprint-24' in TEST space"
d) By Parent:      "Get all child pages of parent page 123456"
```

### Step 2: Read Content via MCP

Use the MCP plugin tools to fetch content:

**Example 1: Read a single page by ID**
```
Use confluence_get_page with pageId: "123456"
```

**Example 2: Read all feature pages from a space by label**
```
Use confluence_search with cql: "space=TEST and label=feature"
```

**Example 3: Read a page hierarchy (EPIC with stories)**
```
1. Use confluence_get_page_by_title → Get EPIC page
2. Use confluence_get_child_pages → Get all story pages
```

### Step 3: Extract and Normalize Content

The raw Confluence content will be transformed:

```markdown
## Feature: [Feature Name]
**Page ID**: 123456
**Last Updated**: 2024-01-15

### Description
[Feature description from Confluence]

### Acceptance Criteria
- AC1: ...
- AC2: ...

### Business Rules
- Rule 1: ...
- Rule 2: ...

### Technical Notes
- Note 1: ...
```

## Normalization Rules

### Table Extraction
Confluence tables are converted to structured markdown:

```markdown
| Field | Value | Type |
|-------|-------|------|
| Input | username | Text |
| Validation | Required, 3-50 chars | Rule |
```

### Link Resolution
- Internal Confluence links → Markdown links
- Jira issue links → Preserved as references
- Attachments → Noted with filename and URL

### Code Blocks
- Confluence code macros → Fenced code blocks with language tag
- Inline code → Backtick formatting

## Output Format

After reading, the content is structured as follows for downstream skills:

```yaml
source:
  pageId: "123456"
  title: "User Login Feature"
  space: "TEST"
  url: "https://.../123456"
  lastModified: "2024-01-15T10:30:00Z"
  
content:
  raw_markdown: "[Full markdown content]"
  
metadata:
  labels: ["feature", "sprint-24"]
  parentPageId: "12345"
  childPageIds: ["123457", "123458"]
  
extracted:
  tables: [...]
  code_blocks: [...]
  links: [...]
```

## Error Handling

| Error | Handling Strategy |
|-------|------------------|
| Page not found | Log error, suggest search by title/label |
| Authentication failure | Check CONFLUENCE_API_TOKEN validity |
| Network timeout | Retry with exponential backoff (max 3 attempts) |
| MCP plugin not configured | Prompt user to configure `.vscode/mcp.json` |

## Best Practices

1. **Always verify** the page title matches the expected feature
2. **Prefer page IDs** over titles to avoid ambiguity
3. **Include metadata** (labels, parent/child relationships) for context
4. **Handle large documents** by splitting into sections
5. **Cache results** when processing multiple related pages
6. **Log the page URL** for traceability

## Integration with Downstream Skills

The output of this skill feeds directly into **feature-analyzer**:

```
confluence-reader (raw content)
    │
    ▼
feature-analyzer (structured analysis)
    │
    ▼
test-scenario-writer (generated test scenarios)
```
