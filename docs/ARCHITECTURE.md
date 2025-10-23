# GitHub MCP Server - Architecture Overview

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         AI Assistant                             │
│                      (Claude, ChatGPT, etc.)                     │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ MCP Protocol (stdio/HTTP/SSE)
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                     GitHub MCP Server                            │
│                      (github_mcp.py)                            │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Tool Layer (8 Tools)                         │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │  • github_get_repo_info        • github_get_file_content │  │
│  │  • github_list_issues          • github_list_pull_requests│ │
│  │  • github_create_issue         • github_get_user_info     │ │
│  │  • github_search_repositories  • github_list_repo_contents│ │
│  └──────────────────────────────────────────────────────────┘  │
│                             │                                    │
│  ┌──────────────────────────▼────────────────────────────────┐ │
│  │           Input Validation Layer                          │ │
│  │           (Pydantic Models)                               │ │
│  ├───────────────────────────────────────────────────────────┤ │
│  │  • Type checking          • Field validation              │ │
│  │  • Constraint enforcement • Automatic coercion            │ │
│  └──────────────────────────┬────────────────────────────────┘ │
│                             │                                    │
│  ┌──────────────────────────▼────────────────────────────────┐ │
│  │           Utility Layer                                    │ │
│  ├───────────────────────────────────────────────────────────┤ │
│  │  • _make_github_request   • _truncate_response            │ │
│  │  • _handle_api_error      • _format_timestamp             │ │
│  └──────────────────────────┬────────────────────────────────┘ │
│                                                                  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ HTTPS REST API
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       GitHub API                                 │
│                  (api.github.com)                               │
│                                                                  │
│  • Repositories    • Pull Requests    • Users                   │
│  • Issues          • Search           • Contents                │
└─────────────────────────────────────────────────────────────────┘
```

## Data Flow

### Example: Getting Repository Information

```
1. User Query (Natural Language)
   "Tell me about the tensorflow/tensorflow repository"
           │
           ▼
2. AI Assistant (Claude)
   Decides to use: github_get_repo_info
           │
           ▼
3. MCP Protocol Message
   {
     "tool": "github_get_repo_info",
     "params": {
       "owner": "tensorflow",
       "repo": "tensorflow",
       "response_format": "markdown"
     }
   }
           │
           ▼
4. Input Validation (Pydantic)
   ✓ owner: str (valid)
   ✓ repo: str (valid)
   ✓ response_format: "markdown" (enum valid)
           │
           ▼
5. GitHub API Request
   GET https://api.github.com/repos/tensorflow/tensorflow
   Headers: Accept, Authorization, API-Version
           │
           ▼
6. Response Processing
   • Parse JSON response
   • Format as Markdown
   • Check character limit
   • Apply truncation if needed
           │
           ▼
7. Return to AI Assistant
   Formatted Markdown with:
   - Repository stats
   - Description
   - License info
   - URLs
           │
           ▼
8. AI Assistant Response
   Natural language summary presented to user
```

## Component Details

### Tool Layer
**Responsibility**: Expose GitHub functionality as MCP tools
- Each tool focuses on a specific workflow
- Comprehensive docstrings for LLM understanding
- Tool annotations for client hints
- Async implementation for performance

### Input Validation Layer
**Responsibility**: Ensure data integrity and type safety
- Pydantic v2 models for all inputs
- Field-level constraints (min/max length, ranges)
- Type coercion and validation
- Custom validators where needed
- Helpful error messages

### Utility Layer
**Responsibility**: Shared functionality across tools
- DRY principle - no code duplication
- Consistent error handling
- Response formatting (JSON/Markdown)
- API request management
- Timestamp formatting
- Truncation logic

### GitHub API Layer
**Responsibility**: Official GitHub REST API v3
- Rate limiting: 60/hour (unauth) or 5000/hour (auth)
- Search API: 10 requests/minute
- Comprehensive REST endpoints
- Well-documented response schemas

## Error Handling Flow

```
Tool Execution
     │
     ▼
Try-Catch Block
     │
     ├─── Success ──────────────────────────┐
     │                                       │
     └─── Exception                          │
           │                                 │
           ▼                                 │
     Check Exception Type                   │
           │                                 │
           ├── HTTPStatusError               │
           │   ├─── 404 → "Not found"        │
           │   ├─── 403 → "Permission denied"│
           │   ├─── 401 → "Auth required"    │
           │   ├─── 422 → "Invalid params"   │
           │   └─── 429 → "Rate limited"     │
           │                                 │
           ├── TimeoutException              │
           │   └─── "Request timeout"        │
           │                                 │
           ├── NetworkError                  │
           │   └─── "Network error"          │
           │                                 │
           └── Generic Exception             │
               └─── "Unexpected error"       │
                                             │
           ┌─────────────────────────────────┘
           │
           ▼
     Format Response
           │
           ▼
     Apply Truncation (if needed)
           │
           ▼
     Return to AI Assistant
```

## Response Format Options

### Markdown (Default)
```
# tensorflow/tensorflow

**Description:** An Open Source Machine Learning Framework

## Statistics
- ⭐ Stars: 185,000
- 🍴 Forks: 74,000
...
```

**Use Case**: Human-readable, presentable to users

### JSON
```json
{
  "full_name": "tensorflow/tensorflow",
  "description": "An Open Source...",
  "stargazers_count": 185000,
  ...
}
```

**Use Case**: Programmatic processing, data extraction

## Security Architecture

```
┌─────────────────────────────────────────────┐
│         User Provides Token                  │
│         (Optional)                           │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│    Token Validation Layer                   │
│    • Never logged                            │
│    • Only sent in Authorization header       │
│    • Scoped permissions checked              │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│    GitHub API                                │
│    • Validates token                         │
│    • Enforces rate limits                    │
│    • Checks repository permissions           │
└─────────────────────────────────────────────┘
```

## Scalability Considerations

### Rate Limiting
- **Without Token**: 60 requests/hour
- **With Token**: 5,000 requests/hour
- **Search API**: 10 requests/minute

### Pagination
- Default: 20 items per page
- Maximum: 100 items per page
- Prevents memory issues with large datasets

### Character Limits
- Maximum response: 25,000 characters
- Automatic truncation with clear notices
- Guidance on filtering to reduce size

### Caching Strategy (Future)
```
Request
   │
   ▼
Check Cache
   │
   ├─── Hit ────────► Return Cached Response
   │
   └─── Miss
         │
         ▼
   GitHub API
         │
         ▼
   Store in Cache (TTL: 5 minutes)
         │
         ▼
   Return Response
```

## Extension Points

### Adding New Tools
```python
@mcp.tool(
    name="github_new_feature",
    annotations={...}
)
async def github_new_feature(params: NewFeatureInput) -> str:
    """
    1. Define Pydantic input model
    2. Implement async function
    3. Use shared utilities
    4. Handle errors consistently
    5. Support both response formats
    """
```

### Adding New Utilities
```python
async def _new_utility(param: str) -> dict:
    """
    1. Keep async for I/O operations
    2. Add type hints
    3. Document clearly
    4. Reuse in multiple tools
    """
```

## Technology Stack

```
┌─────────────────────────────────────────┐
│   Python 3.10+                          │
│   ├── mcp (MCP SDK)                     │
│   ├── httpx (Async HTTP client)        │
│   ├── pydantic (Data validation)       │
│   └── typing (Type hints)              │
└─────────────────────────────────────────┘
```

## Deployment Options

### Local Development
```bash
python github_mcp.py
# Runs stdio transport for local testing
```

### Claude Desktop Integration
```json
{
  "mcpServers": {
    "github": {
      "command": "python",
      "args": ["/path/to/github_mcp.py"]
    }
  }
}
```

### Future: HTTP Server
```python
# Modify for HTTP transport
mcp.run(transport="http", port=8080)
```

### Future: SSE Server
```python
# Modify for SSE transport
mcp.run(transport="sse", port=8080)
```

## Performance Characteristics

| Operation | Latency | Rate Limit Impact |
|-----------|---------|-------------------|
| Get Repo Info | ~200ms | 1 request |
| List Issues | ~300ms | 1 request |
| Search Repos | ~400ms | 1 request (search limit) |
| Create Issue | ~300ms | 1 request |
| Get File Content | ~250ms | 1 request |

**Note**: Latencies are approximate and depend on network conditions and GitHub API performance.

## Quality Metrics

- ✅ **Test Coverage**: 10 evaluation scenarios
- ✅ **Type Coverage**: 100% with type hints
- ✅ **Documentation**: Comprehensive docstrings
- ✅ **Error Handling**: All exceptions caught
- ✅ **Input Validation**: Pydantic models for all inputs
- ✅ **Code Quality**: Follows PEP 8 and MCP best practices
- ✅ **Security**: No hardcoded credentials, optional auth
