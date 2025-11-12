# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Model Context Protocol (MCP) server that wraps the Apache Airflow REST API, enabling MCP clients (like Claude Desktop) to interact with Airflow in a standardized way. It uses the official `apache-airflow-client` library and the `fastmcp` framework to expose Airflow functionality as MCP tools.

## Development Commands

### Testing
```bash
# Run all tests
make test

# Run tests directly with pytest
uv run python -m pytest test/ -v
```

### Code Quality
```bash
# Run linting (with auto-fix)
make lint

# Format code
make format
```

### Running the Server
```bash
# Run with default settings (stdio transport, all APIs)
make run

# Run with SSE transport
make run-sse

# Run with custom parameters
make run PARAMS="--transport http --port 8080 --apis dag --apis variable --read-only"

# Run directly via uv
uv run src --transport stdio --apis dag --apis dagrun
```

### Building and Publishing
```bash
# Build distribution packages
make build

# Publish to PyPI (auto-deployed on version bump in pyproject.toml)
make publish
```

## Architecture

### Core Components

1. **src/main.py**: CLI entry point that registers tools and starts the MCP server
   - Uses Click for CLI argument parsing
   - Dynamically registers tools based on `--apis` and `--read-only` flags
   - Maps API types to their function providers via `APITYPE_TO_FUNCTIONS`

2. **src/server.py**: FastMCP server instance (`app = FastMCP("mcp-apache-airflow")`)
   - Single global server instance that tools are registered to

3. **src/envs.py**: Environment variable configuration
   - Supports both basic auth (username/password) and JWT token auth
   - JWT token takes precedence if both are provided
   - Defaults to `http://localhost:8080` for development

4. **src/airflow/airflow_client.py**: Centralized Airflow API client configuration
   - Creates a shared `api_client` instance used by all API modules
   - Handles authentication setup

5. **src/airflow/[module].py**: API-specific tool implementations (one per API group)
   - Each module implements tools for a specific Airflow API area (DAGs, variables, connections, etc.)
   - Each module exports `get_all_functions()` returning list of `(func, name, description, is_read_only)` tuples

### Tool Registration Pattern

All API modules follow this pattern:

```python
from typing import Callable

def get_all_functions() -> list[tuple[Callable, str, str, bool]]:
    """Return list of (function, name, description, is_read_only) tuples."""
    return [
        (my_function, "tool_name", "Tool description", True),  # True = read-only
        (another_function, "other_tool", "Another tool", False),  # False = write operation
    ]
```

Tools are async functions that return `List[Union[types.TextContent, types.ImageContent, types.EmbeddedResource]]`.

### Read-Only Mode

The server supports read-only mode (via `--read-only` flag or `READ_ONLY=true` env var) which:
- Filters tools to only include those with `is_read_only=True`
- Implemented in `filter_functions_for_read_only()` in src/main.py:43
- Excludes create/update/delete operations

### Authentication

Authentication is configured in src/airflow/airflow_client.py:18-24:
- JWT token (if `AIRFLOW_JWT_TOKEN` is set): Uses Bearer token authentication
- Basic auth (if `AIRFLOW_USERNAME` and `AIRFLOW_PASSWORD` are set): Uses HTTP basic auth
- JWT takes precedence over basic auth

### API Groups

Available API groups (see src/enums.py):
- `config`: Airflow configuration
- `connection`: Connection management
- `dag`: DAG management and operations
- `dagrun`: DAG run management
- `dagstats`: DAG statistics
- `dataset`: Dataset management
- `eventlog`: Event logs
- `importerror`: Import errors
- `monitoring`: Health and monitoring
- `plugin`: Installed plugins
- `pool`: Pool management
- `provider`: Provider packages
- `taskinstance`: Task instance management
- `variable`: Variable management
- `xcom`: XCom entries

## Key Implementation Details

### Response Format
All tools return responses as dictionaries converted to strings. Many tools add helpful metadata:
- DAG tools add `ui_url` field pointing to the Airflow web UI
- Responses are converted from Airflow client objects using `.to_dict()`

### Test Structure
- Tests are in `test/` directory (flat structure, no subdirectories)
- Test file naming: `test_[module].py` (e.g., `src/main.py` → `test/test_main.py`)
- Uses pytest with fixtures in `test/conftest.py`
- Mocks external dependencies using `unittest.mock`

### Environment Variables
No environment variables are required for tests. In development/production:
- `AIRFLOW_HOST`: Airflow server URL (default: `http://localhost:8080`)
- `AIRFLOW_USERNAME` / `AIRFLOW_PASSWORD`: Basic auth credentials
- `AIRFLOW_JWT_TOKEN`: JWT token for authentication (takes precedence)
- `AIRFLOW_API_VERSION`: API version (default: `v1`)
- `READ_ONLY`: Enable read-only mode (default: `false`)

## Contributing

- Follow semantic versioning (MAJOR.MINOR.PATCH)
- Include version bump in `pyproject.toml` for core logic changes
- Package is auto-deployed to PyPI when version is updated
- Python 3.10+ required (tested on 3.10, 3.11, 3.12)
- CI runs tests and linting on every push/PR to main

## Code Style (from .cursor/rules/)

### Python (.cursor/rules/python.mdc)
- Line length: 120 characters
- Use type hints for all functions
- Use f-strings for formatting
- Follow PEP 8
- Keep try-except blocks minimal
- Put constants at top of file after imports
- Caller should come before callee
- Use lowercase_with_underscores for variables/functions

### MCP Guidelines (.cursor/rules/modelcontextprotocol.mdc)
- Keep servers stateless
- Use async/await for I/O operations
- Handle errors gracefully with meaningful messages
- Document all tool functions (docstrings are used for tool descriptions)
- Use type hints for automatic tool definition generation
- Follow the tool registration pattern with `get_all_functions()`
