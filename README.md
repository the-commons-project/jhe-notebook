# JHE Notebook: JupyterHealth Exchange MCP Client with LLM Integration

This project demonstrates how to use the Model Context Protocol (MCP) to connect a Jupyter notebook to the JupyterHealth Exchange server, enabling LLM-powered queries over health data using the NRP-hosted Qwen3 model.

## Overview

The JHE Notebook connects:
- **MCP Client** → **JupyterHealth Exchange MCP Server** (local, port 8001)
- **OpenAI Client** → **NRP Qwen3 LLM** (hosted at NRP)
- **Integration**: LLM uses MCP tools to query health data and respond in natural language

## Architecture

```
┌─────────────────┐
│ Jupyter Notebook│
│  (This Project) │
└────┬────────┬───┘
     │        │
     │        └──────────────┐
     │                       │
     ▼                       ▼
┌─────────────────┐    ┌──────────────┐
│  MCP Client     │    │ OpenAI Client│
│  (Python SDK)   │    │  (NRP API)   │
└────┬────────────┘    └──────┬───────┘
     │                        │
     │                        │
     ▼                        ▼
┌─────────────────┐    ┌──────────────┐
│ JHE MCP Server  │    │ NRP Qwen3    │
│ localhost:8001  │    │   Model      │
└────┬────────────┘    └──────────────┘
     │
     ▼
┌─────────────────┐
│ JupyterHealth   │
│ Exchange (JHE)  │
│ localhost:8000  │
└─────────────────┘
```

## Prerequisites

### 0. JupyterHealth Exchange with MCP Support

**Important**: You need the MCP-enabled version of JupyterHealth Exchange.

Clone the JupyterHealth Exchange repository and checkout the MCP feature branch:
```bash
git clone https://github.com/jupyterhealth/jupyterhealth-exchange.git
cd jupyterhealth-exchange

# TODO: Update this when PR is merged - for now use the feature branch
git fetch origin pull/XXX/head:mcp-support  # Replace XXX with actual PR number
git checkout mcp-support
```

Follow the JupyterHealth Exchange setup instructions to:
- Install dependencies
- Configure the database
- Seed test data

### 1. Python 3.10+

**Important**: The MCP Python SDK requires Python 3.10 or later.

Check your Python version:
```bash
python3 --version  # Should be 3.10 or higher
```

If needed, use a specific Python version:
```bash
# macOS with Homebrew
brew install python@3.11

# Use specific Python version for venv
python3.11 -m venv .venv
```

### 2. Running Infrastructure

**JupyterHealth Exchange Server** must be running:
```bash
cd /path/to/jupyterhealth-exchange
.venv/bin/python manage.py runserver
```

**MCP Server** must be running:
```bash
cd /path/to/jupyterhealth-exchange
./start_mcp_server.sh
```

Verify servers are healthy:
```bash
curl http://localhost:8000/admin/  # Should return 200
curl http://localhost:8001/health  # Should return {"status": "healthy"}
```

### 3. NRP API Access

You need an API key from the National Research Platform:
1. Visit https://portal.nrp.ai/
2. Create an account or log in
3. Generate an API token
4. Note the model name (e.g., `qwen3`)

### 4. Async Event Loop Compatibility

**Important**: The MCP Python SDK requires `nest-asyncio` to work in Jupyter notebooks:
- Jupyter runs an active asyncio event loop
- MCP SDK uses async/await extensively
- nest-asyncio patches the loop to allow nested async operations

This is automatically installed via requirements.txt and applied in the notebook's first cell.

**Security Note**: All queries go through the MCP server which enforces OAuth authentication and permission filtering. Never query the database directly - the MCP server is the gatekeeper that ensures:
- OAuth authentication with JWT verification
- Permission filtering by study_ids from ID token claims
- SQL injection protection
- Read-only operations only

### 5. OAuth Authentication

The MCP server uses OAuth to authenticate with JHE. You should have already completed the OAuth flow (tokens cached at `~/.jhe_mcp/token_cache.json`). If not, the notebook will guide you through authentication.

## Setup

### 1. Clone This Repository

```bash
git clone https://github.com/the-commons-project/jhe-notebook.git
cd jhe-notebook
```

### 2. Create Virtual Environment

**Important**: Use Python 3.10+ (e.g., python3.11)

```bash
# Use Python 3.11 (or 3.10+)
python3.11 -m venv .venv

# Activate the environment
source .venv/bin/activate  # On Windows: .venv\Scripts\activate
```

### 3. Install Dependencies

```bash
pip install -r requirements.txt
```

### 4. Configure Environment Variables

```bash
cp .env.example .env
# Edit .env and add your NRP API key
```

Example `.env`:
```
JHE_MCP_URL=http://localhost:8001/sse
NRP_BASE_URL=https://ellm.nrp-nautilus.io/v1
NRP_API_KEY=your_actual_api_key_here
NRP_MODEL=qwen3
```

### 5. Start Jupyter

```bash
jupyter notebook
```

Navigate to `notebooks/jhe_mcp_llm_demo.ipynb` and open it.

## Usage

The demonstration notebook shows:

1. **MCP Connection**: Connect to the local JHE MCP server via SSE
2. **Tool Discovery**: List available MCP tools (get_study_count, list_studies, execute_filtered_query, etc.)
3. **LLM Integration**: Use Qwen3 to interpret natural language queries
4. **Function Calling**: LLM decides which MCP tools to call
5. **Data Retrieval**: Execute SQL queries filtered by user permissions
6. **Natural Language Response**: LLM formats results for human consumption

### Example Queries

```python
# Simple query
"How many studies are in the system?"

# Complex health data query
"Show me all blood pressure readings for patient 40001 on January 1, 2025,
and calculate the average systolic pressure"

# Time-series aggregation
"Give me hourly averages and standard deviations for patient 40001's
blood pressure data over the first week of January"
```

## Available MCP Tools

The JHE MCP Server provides these tools:

- **get_study_count**: Count studies accessible to authenticated user
- **list_studies**: List all studies with IDs, names, and organizations
- **get_patient_demographics**: Get patient demographics for a specific study
- **get_study_metadata**: Get detailed metadata about a study
- **get_patient_observations**: Get FHIR observations for a specific patient
- **get_database_schema**: Get comprehensive database schema with examples
- **execute_filtered_query**: Execute arbitrary SQL SELECT queries (auto-filtered by permissions)

## Security & Permissions

- All MCP tool calls are authenticated via OAuth
- SQL queries are automatically filtered based on user permissions
- Users can only access studies/patients they have permission to view
- Only SELECT queries allowed (no INSERT/UPDATE/DELETE)
- Dangerous SQL operations are blocked

## Troubleshooting

### CancelledError in Cell 2

```
asyncio.CancelledError: Cancelled via cancel scope...
```

**Solution**: Make sure Cell 1 includes the nest_asyncio setup:
```python
import nest_asyncio
nest_asyncio.apply()
```

If you see this error, check that Cell 1 has been run and includes the nest_asyncio code. See [NOTEBOOK_UPDATES.md](NOTEBOOK_UPDATES.md) for detailed instructions.

### MCP Connection Fails

```
Error: Connection refused to localhost:8001
```

**Solution**: Ensure the MCP server is running:
```bash
curl http://localhost:8001/health
```

### Authentication Errors

```
Error: Authentication failed
```

**Solution**: Re-authenticate by deleting the token cache:
```bash
rm ~/.jhe_mcp/token_cache.json
```
Then restart the notebook - it will trigger OAuth flow.

### NRP API Errors

```
Error: 401 Unauthorized
```

**Solution**: Check your `.env` file has the correct `NRP_API_KEY`.

### No Data Returned

```
"row_count": 0
```

**Solution**:
1. Verify you're logged in as a user with study access (e.g., mary@example.com)
2. Check that test data exists in the database
3. Ensure patient IDs are correct (e.g., 40001)

## Development

### Project Structure

```
jhe-notebook/
├── README.md              # This file
├── requirements.txt       # Python dependencies
├── .env.example          # Environment variable template
├── .env                  # Your actual config (gitignored)
├── .gitignore            # Git ignore rules
└── notebooks/
    └── jhe_mcp_llm_demo.ipynb  # Main demonstration notebook
```

### Adding New Notebooks

Create new `.ipynb` files in the `notebooks/` directory. They will automatically have access to the same dependencies and configuration.

## Contributing

This is a demonstration project showing MCP integration patterns. Feel free to:
- Add new example notebooks
- Create helper utilities
- Improve error handling
- Add data visualizations

## Repository

This project will be published to The Commons Project GitHub organization at:
`https://github.com/the-commons-project/jhe-notebook`

## License

Apache 2.0 (same as other Commons Project repositories)

## Resources

- [Model Context Protocol Docs](https://modelcontextprotocol.io/)
- [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- [NRP Documentation](https://portal.nrp.ai/documentation/)
- [JupyterHealth Exchange](https://github.com/the-commons-project/jupyterhealth-exchange)
