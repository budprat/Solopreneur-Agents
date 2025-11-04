# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A2A-MCP is a multi-agent system implementing the Agent-to-Agent (A2A) protocol with Model Context Protocol (MCP) integration. The system demonstrates collaborative AI agents working together to solve complex tasks through standardized communication and dynamic tool discovery.

**Key Systems:**
- **Travel Agency System**: Orchestrated booking for flights, hotels, and car rentals
- **Solopreneur Oracle**: Autonomous workflow intelligence with specialized domain agents
- **Market Oracle**: Market analysis and research coordination

## Core Architecture

### Multi-Agent Architecture (Tiered)

The system uses a 3-tier agent architecture:

```
Tier 1 (Master Agents) - High-level orchestration and planning
├── Orchestrator Agent (port 10101) - Workflow coordination
└── Planner Agent (port 10102) - Task decomposition using LangGraph

Tier 2 (Domain Specialists) - Specialized knowledge and execution
├── Air Ticketing Agent (port 10103)
├── Hotel Booking Agent (port 10104)
├── Car Rental Agent (port 10105)
└── Solopreneur Oracle Agents (various ports 10200+)

Tier 3 (Service Agents) - Direct tool integration and atomic operations
└── MCP Server (port 10100) - Agent/tool discovery and registry
```

### Critical Communication Flow

1. **Client → Orchestrator** (10101): User sends request
2. **Orchestrator → Planner** (10102): Request decomposition into structured plan
3. **Orchestrator → MCP Server** (10100): Agent discovery via resource queries
4. **Orchestrator → Task Agents** (10103-10105): Execute tasks via A2A protocol
5. **Task Agents → MCP Server** (10100): Query tools (database, APIs)
6. **Task Agents → Orchestrator**: Return structured results
7. **Orchestrator → Client**: Aggregated response

### Unified Agent Pattern

All travel service agents use the **same TravelAgent class** (`src/a2a_mcp/agents/adk_travel_agent.py`) specialized through:
- **Agent Cards**: JSON configs in `agent_cards/` defining capabilities
- **Prompt Instructions**: Service-specific chain-of-thought workflows in prompts.py
- **Port Assignment**: Each service runs on unique port
- **MCP Tool Integration**: Database access via MCP tools

## Development Commands

### Environment Setup

```bash
# Project uses uv for Python package management
uv venv
source .venv/bin/activate

# Dependencies are auto-installed by uv when running commands
# Configuration in pyproject.toml
```

### Starting Services

**IMPORTANT**: MCP server must start first, then other agents can start in any order.

```bash
# Start MCP Server (REQUIRED FIRST)
uv run a2a-mcp --run mcp-server --transport sse --host localhost --port 10100

# Start individual agents
uv run src/a2a_mcp/agents/ --agent-card agent_cards/orchestrator_agent.json --port 10101
uv run src/a2a_mcp/agents/ --agent-card agent_cards/planner_agent.json --port 10102
uv run src/a2a_mcp/agents/ --agent-card agent_cards/air_ticketing_agent.json --port 10103
uv run src/a2a_mcp/agents/ --agent-card agent_cards/hotel_booking_agent.json --port 10104
uv run src/a2a_mcp/agents/ --agent-card agent_cards/car_rental_agent.json --port 10105

# OR start all services at once
./run_all_agents.sh
```

### Stopping Services

```bash
# Stop all running agents and MCP server
./stop_services.sh

# Or manually kill processes
pkill -f "a2a-mcp"
pkill -f "src/a2a_mcp/agents"
```

### Testing

```bash
# Run full test suite with coverage
cd agentic-framework-boilerplate
./run_tests.sh

# Run specific test file
uv run pytest tests/test_specific.py -v

# Run integration tests
uv run python test_solopreneur_comprehensive.py
uv run python test_full_solopreneur_orchestration.py
```

### Database Operations

```bash
# Initialize travel agency database (SQLite)
cd agentic-framework-boilerplate
python init_database.py

# Database location: travel_agency.db
# Tables: flights, hotels, car_rentals
```

### Client Interaction

```bash
# Interactive chat with orchestrator
python chat.py

# Simple query client
python query.py "Book a flight from NYC to London"

# Comprehensive test client
python comprehensive_client.py
```

## Key Protocols and Standards

### A2A Protocol (Agent-to-Agent)

- JSON-RPC based communication between agents
- Standardized message format with SSE support
- All agents expose HTTP endpoints for A2A messages
- Protocol implementation in `src/a2a_mcp/core/protocol.py`

### MCP (Model Context Protocol)

- Registry for agent cards and tool discovery
- Exposes agents as **resources** (discoverable via `list_resources`)
- Exposes tools for database queries and external services
- Server implementation in `src/a2a_mcp/mcp/server.py`

### Agent Cards

JSON configurations defining agent capabilities:
```json
{
  "name": "Agent Name",
  "description": "What this agent does",
  "port": 10103,
  "version": "1.0.0",
  "skills": ["skill1", "skill2"],
  "specializations": ["domain1", "domain2"]
}
```

Location: `agent_cards/*.json`

## Source Code Structure

```
src/a2a_mcp/
├── core/               # Core protocols and quality standards
│   ├── protocol.py     # A2A protocol implementation
│   ├── agent.py        # Base agent functionality
│   └── quality.py      # Quality validation
├── agents/             # Agent implementations
│   ├── __main__.py     # Agent startup and factory
│   ├── orchestrator_agent.py       # Sequential orchestration
│   ├── parallel_orchestrator_agent.py  # Parallel execution
│   ├── langgraph_planner_agent.py  # LangGraph-based planning
│   ├── adk_travel_agent.py         # Unified travel agent
│   ├── solopreneur_oracle/         # Solopreneur domain agents
│   └── tier3/          # Service tier agents
├── mcp/               # MCP integration
│   ├── server.py      # MCP server implementation
│   └── client.py      # MCP client for testing
└── common/            # Shared utilities
    ├── base_agent.py  # Abstract base for all agents
    ├── prompts.py     # Chain-of-thought instructions
    ├── types.py       # Pydantic models and types
    └── utils.py       # Utility functions
```

## Configuration and Environment

### Required Environment Variables

**CRITICAL**: `GOOGLE_API_KEY` is required for Planner and TravelAgent instances (uses Gemini).

```bash
# Copy template and configure
cp .env.example .env

# Essential variables
GOOGLE_API_KEY=your_key_here          # REQUIRED for Gemini
LOG_LEVEL=INFO
MCP_SERVER_HOST=localhost
MCP_SERVER_PORT=10100
```

See `.env.example` for full list of optional integrations (Notion, Reddit, Snowflake, etc.).

## Adding New Agents or Services

### Step 1: Create Agent Card
Create JSON file in `agent_cards/` with capabilities definition.

### Step 2: Add Chain-of-Thought Prompt
Add prompt instructions to `src/a2a_mcp/agents/prompts.py` following existing pattern.

### Step 3: Update Agent Factory
Modify `src/a2a_mcp/agents/__main__.py` to instantiate your agent.

### Step 4: Create Database Schema (if needed)
Add tables to database initialization script.

### Step 5: Update Launch Scripts
Add agent startup command to `run_all_agents.sh`.

**Reference**: See `DEVELOPER_GUIDE.md` for complete walkthrough with Travel Insurance Agent example.

## Important Development Rules

### Debugging and Problem Solving

**CRITICAL**: When encountering errors:
- **NEVER bypass errors** by creating simpler demos or test files
- **NEVER use mock data** when real data sources fail - fix the integration
- **NEVER create "simple" or "basic" versions** when tests fail - fix actual problems
- **ALWAYS find root cause** before implementing fixes
- If API/service not working, fix connection/auth/timeout - don't bypass with fake responses

### Notion Integration Rules

**WRITE ONLY TO THE "AI" PAGE** (ID: 21e0f849-e0a2-80c3-8b72-ccb0a1b61ab9)
- All other Notion pages are READ-ONLY
- See `notion_mcp_rules.md` for detailed guidelines
- Available pages listed in `notion_pages_complete_list.md`

## Common Workflows

### Starting from Scratch
```bash
# 1. Set up environment
cp .env.example .env
# Edit .env and add GOOGLE_API_KEY

# 2. Initialize database
python init_database.py

# 3. Start all services
./run_all_agents.sh

# 4. Test the system
python chat.py
```

### Debugging Agent Issues
```bash
# 1. Check if MCP server is running
curl http://localhost:10100/health

# 2. Check agent status
./status.sh

# 3. View agent logs
tail -f logs/orchestrator_agent.log
tail -f logs/planner_agent.log

# 4. Test individual agent
curl -X POST http://localhost:10103/ \
  -H "Content-Type: application/json" \
  -d '{"message": {"role": "user", "parts": [{"kind": "text", "text": "test query"}]}}'
```

### Performance Optimization

- Use `parallel_orchestrator_agent.py` for concurrent task execution
- Parallel execution provides 2-3x performance improvement
- Tasks with no dependencies execute concurrently via asyncio

## Framework Reference

This project follows the [A2A-MCP Unified Framework V2.0](A2A_MCP_UNIFIED_FRAMEWORK_V2.md) for standardized patterns.

Key documents:
- `A2A_MCP_FRAMEWORK_REFERENCE.md` - Complete framework specification
- `DEVELOPER_GUIDE.md` - Step-by-step guide for adding services
- `PROJECT.md` - Project-specific guidance and commands
- `TESTING_WORKFLOW.md` - Comprehensive testing guidelines

## Troubleshooting

### Port Already in Use
```bash
# Clean up ports
./cleanup_ports.sh

# Or manually
lsof -ti:10100 | xargs kill -9
```

### Agent Not Discovered by MCP
- Verify agent card JSON syntax
- Check MCP server logs for registration
- Ensure agent skills/description match query terms

### Database Connection Issues
- Verify database exists: `ls -la travel_agency.db`
- Re-initialize if needed: `python init_database.py`
- Check file permissions

### GOOGLE_API_KEY Not Set
```bash
# Set temporarily
export GOOGLE_API_KEY='your-key-here'

# Or add to .env file
echo "GOOGLE_API_KEY=your-key-here" >> .env
```