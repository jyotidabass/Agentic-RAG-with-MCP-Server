# 🤖 Agentic RAG with MCP Server

A production-ready implementation of **Agentic Retrieval-Augmented Generation (RAG)** using the **Model Context Protocol (MCP)**. This project demonstrates how to build an AI agent that can intelligently query a knowledge base (Sri Lanka news articles) by using MCP tools for query refinement, entity extraction, and relevance scoring — all powered by OpenAI.

---

## 📖 Medium Article

This project is accompanied by a detailed walkthrough on Medium:

**[Agentic RAG with MCP Server — Every Line of Code Explained Simply](https://medium.com/tech-ai-made-easy/agentic-rag-with-mcp-server-every-line-of-code-explained-simply-9ddff277d699?sk=2a286eb22d47f7aaf7a8c726461d9cec)**

> *Stop hallucinating. Start retrieving. Here's how to build an AI that actually finds answers instead of making them up.*

Published in **[Tech & AI Made Easy](https://medium.com/tech-ai-made-easy)** by **Jyoti Dabass, Ph.D.**

The article covers:
- A plain-English breakdown of RAG, Agentic RAG, and MCP concepts
- Step-by-step explanation of every line of code across all project files
- How the agentic loop works (the 3-round GPT ↔ tool cycle)
- SSE vs STDIO transport comparison
- How to extend the project with new tools and a full MongoDB + Pinecone pipeline
- Common errors and how to fix them

---

## 📌 Table of Contents

- [What Is This Project?](#what-is-this-project)
- [How It Works — The Big Picture](#how-it-works--the-big-picture)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Running the Project](#running-the-project)
- [Code Walkthrough](#code-walkthrough)
  - [server.py — The MCP Tool Server](#serverpy--the-mcp-tool-server)
  - [mcp-client.py — The AI Agent Client](#mcp-clientpy--the-ai-agent-client)
  - [config.py — Configuration Loader](#configpy--configuration-loader)
- [MCP Tools Explained](#mcp-tools-explained)
- [Transport Modes: SSE vs STDIO](#transport-modes-sse-vs-stdio)
- [Extending This Project](#extending-this-project)
- [Common Errors & Fixes](#common-errors--fixes)

---

## What Is This Project?

### Key Concepts (Plain English)

| Term | What it means |
|------|--------------|
| **RAG** | Instead of relying on an LLM's training data, you fetch relevant documents first and then let the LLM answer based on those documents. |
| **Agentic RAG** | The LLM *decides* which tools to call and when — it is not just a static retrieval pipeline. The agent thinks, retrieves, re-ranks, and answers. |
| **MCP (Model Context Protocol)** | A standard protocol (like a plug-in system) that lets AI models communicate with external tools/servers. Think of it as a USB standard for AI tools. |
| **MCP Server** | A Python process exposing "tools" that any MCP-compatible AI client can call. |
| **MCP Client** | The AI agent side — it connects to the server, discovers available tools, and decides when to invoke them. |

### What does this specific project do?

This project builds an intelligent news-search assistant focused on **Sri Lanka news** that:

1. Takes a natural language question from the user
2. Uses OpenAI to decide which MCP tools to call (automatically)
3. Extracts entities from the query (people, locations, organizations, dates)
4. Refines/optimizes the query for better vector search results
5. Scores the relevance of retrieved documents against the query
6. Returns a final, grounded answer

The system is designed to plug into a **MongoDB + Pinecone** vector database setup (where news articles are already stored and embedded), but the MCP tools themselves work independently.

---

## How It Works — The Big Picture

```
User Query
    │
    ▼
┌─────────────────────────────────┐
│       mcp-client.py             │
│  (OpenAI Agent / LLM Brain)     │
│                                 │
│  1. Sends query to OpenAI       │
│  2. OpenAI decides tool calls   │
│  3. Executes tool calls via MCP │
│  4. Sends results back to OpenAI│
│  5. Gets final answer           │
└────────────┬────────────────────┘
             │  SSE (HTTP) or STDIO
             ▼
┌─────────────────────────────────┐
│         server.py               │
│   (FastMCP Tool Server)         │
│                                 │
│  Tools available:               │
│  ├─ get_time_with_prefix()      │
│  ├─ extract_entities_tool()     │
│  ├─ refine_query_tool()         │
│  └─ check_relevance()           │
│                                 │
│  Each tool calls OpenAI         │
│  internally for intelligence    │
└─────────────────────────────────┘
```

**Important**: Both the client and the server call OpenAI, but for different purposes:
- **Client** → uses OpenAI to orchestrate *which* tools to call
- **Server tools** → use OpenAI to *execute* intelligent tasks (NER, query refinement, relevance scoring)

---

## Project Structure

```
Agentic-RAG-with-MCP-Server/
│
├── server.py           # MCP Server — exposes AI-powered tools
├── mcp-client.py       # MCP Client — the AI agent that uses those tools
├── config.py           # Loads all environment variables / secrets
├── requirements.txt    # Python dependencies
├── .env.sample         # Template for your .env file
├── .gitignore
└── README.md
```

---

## Prerequisites

- Python **3.10+**
- An **OpenAI API key** (with access to `gpt-4o-mini` or your chosen model)
- *(Optional for full RAG pipeline)* A **MongoDB Atlas** account with news articles stored
- *(Optional for full RAG pipeline)* A **Pinecone** account with a vector index created

---

## Installation

### Step 1 — Clone the repository

```bash
git clone https://github.com/YOUR_USERNAME/Agentic-RAG-with-MCP-Server.git
cd Agentic-RAG-with-MCP-Server
```

### Step 2 — Create and activate a virtual environment

```bash
# Create
python -m venv venv

# Activate (Mac/Linux)
source venv/bin/activate

# Activate (Windows)
venv\Scripts\activate
```

### Step 3 — Install dependencies

```bash
pip install -r requirements.txt
```

**What gets installed:**

| Package | Purpose |
|---------|---------|
| `mcp[cli]==1.6.0` | The MCP framework — both server (FastMCP) and client (ClientSession, sse_client) |
| `openai==1.75.0` | OpenAI Python SDK for GPT model calls |
| `python-dotenv` | Loads `.env` file variables into the environment |
| `ipykernel` | Jupyter notebook support (optional, for experimentation) |
| `httpx` | HTTP client (used internally by MCP SSE transport) |

---

## Configuration

### Step 1 — Create your `.env` file

Copy the sample and fill in your values:

```bash
cp .env.sample .env
```

Open `.env` and set:

```env
# Required
OPENAI_API_KEY=sk-...your-openai-key...
OPENAI_MODEL_NAME=gpt-4o-mini

# MCP Server URL (used by the client to connect to the server)
MCP_SERVER_URL=http://0.0.0.0:8050

# Optional — only needed if you integrate MongoDB
MONGODB_USERNAME=your_mongo_username
MONGODB_PASSWORD=your_mongo_password
MONGODB_URI=mongodb+srv://...  # Auto-built from username/password if not provided

# Optional — only needed if you integrate Pinecone
PINECONE_API_KEY=your_pinecone_key
PINECONE_INDEX_NAME=news-articles-index
PINECONE_CLOUD=aws
PINECONE_REGION=us-east-1
```

> **Note**: You only need `OPENAI_API_KEY` and `OPENAI_MODEL_NAME` to run the demo. MongoDB and Pinecone are for a full RAG pipeline.

---

## Running the Project

You need **two terminal windows** — one for the server, one for the client.

### Terminal 1 — Start the MCP Server

```bash
python server.py
```

You should see output like:
```
INFO:     Started server process
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8050
```

The server is now running and waiting for connections on port `8050`.

### Terminal 2 — Run the MCP Client

```bash
python mcp-client.py
```

Expected output:
```
Connected to server with tools:
  - get_time_with_prefix: Get the current date and time.
  - extract_entities_tool: Extract entities from a given text query using OpenAI.
  - refine_query_tool: Refine a given text query using OpenAI.
  - check_relevance: Check the relevance of a text chunk to a given question.

Query: What is the current time?
Response: The current time is 2026-03-22 16:45:32.123456.

Entity Query: What events happened in Colombo, Sri Lanka on January 1st, 2023?
Entity Response: {"LOCATION": ["Colombo", "Sri Lanka"], "DATE": ["January 1st, 2023"]}

...
```

---

## Code Walkthrough

### `server.py` — The MCP Tool Server

This is the heart of the system. It creates an MCP-compatible server with 4 tools.

#### Initialization

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP(
    name="Knowledge Base",
    host="0.0.0.0",   # Accept connections from any IP
    port=8050,        # Port for SSE transport
)
```

`FastMCP` is a high-level wrapper that makes it very easy to turn Python functions into MCP tools. You just decorate them with `@mcp.tool()`.

#### Starting the server

```python
if __name__ == "__main__":
    mcp.run(transport="sse")
```

`transport="sse"` means the server communicates over **Server-Sent Events** (HTTP-based). This is what allows the client to connect via `http://localhost:8050/sse`. The alternative is `stdio` (standard input/output), which is used in subprocess-based setups.

---

### `mcp-client.py` — The AI Agent Client

This is the "brain" — it connects to the server, discovers tools, and runs an agentic loop.

#### Connecting to the server

```python
async with sse_client("http://localhost:8050/sse") as (read_stream, write_stream):
    async with ClientSession(read_stream, write_stream) as session:
        await session.initialize()
```

`sse_client` opens a persistent HTTP connection to the server. `ClientSession` wraps it with the MCP protocol. `session.initialize()` performs a handshake — after this, you can list tools and call them.

#### Discovering tools and converting to OpenAI format

```python
async def get_mcp_tools():
    tools_result = await session.list_tools()
    return [
        {
            "type": "function",
            "function": {
                "name": tool.name,
                "description": tool.description,
                "parameters": tool.inputSchema,
            },
        }
        for tool in tools_result.tools
    ]
```

This fetches all tools from the MCP server and converts them into the format OpenAI's function-calling API expects. OpenAI uses these descriptions to decide which tool to call.

#### The Agentic Loop — `process_query()`

```python
async def process_query(query: str) -> str:
    tools = await get_mcp_tools()

    # Step 1: Ask OpenAI — it may decide to call a tool
    response = await openai_client.chat.completions.create(
        model=os.getenv("OPENAI_MODEL_NAME"),
        messages=[{"role": "user", "content": query}],
        tools=tools,
        tool_choice="auto",  # OpenAI decides whether to use a tool
    )

    assistant_message = response.choices[0].message
    messages = [{"role": "user", "content": query}, assistant_message]

    # Step 2: If OpenAI requested tool calls, execute them
    if assistant_message.tool_calls:
        for tool_call in assistant_message.tool_calls:
            result = await session.call_tool(
                tool_call.function.name,
                arguments=json.loads(tool_call.function.arguments),
            )
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": result.content[0].text,
            })

        # Step 3: Send tool results back to OpenAI for the final answer
        final_response = await openai_client.chat.completions.create(
            model=os.getenv("OPENAI_MODEL_NAME"),
            messages=messages,
            tools=tools,
            tool_choice="none",  # No more tool calls — just answer
        )
        return final_response.choices[0].message.content

    return assistant_message.content
```

**This is the agentic loop in plain English:**
1. User asks a question
2. OpenAI reads the question AND the list of available tools
3. OpenAI decides: "I should call `extract_entities_tool` with this query"
4. The client executes that tool call on the MCP server
5. The tool result is sent back to OpenAI
6. OpenAI now writes a final answer using the tool results

---

### `config.py` — Configuration Loader

```python
def load_config():
    load_dotenv()
    
    # Auto-builds MongoDB URI from username + password if full URI not given
    mongodb_uri = os.getenv('MONGODB_URI') or \
        f"mongodb+srv://{username}:{password}@ircw.371s5.mongodb.net/..."
    
    return {
        "MONGODB_URI": mongodb_uri,
        "DEFAULT_MONGODB_COLLECTIONS": [
            {"db": "newsfirst_data", "collection": "articles"},
            {"db": "newswire_data", "collection": "articles"},
            {"db": "adaderana_data", "collection": "articles"},
        ],
        "PINECONE_API_KEY": ...,
        "PINECONE_INDEX_NAME": ...,
        # etc.
    }
```

This centralizes all credentials and settings. The three pre-configured MongoDB collections (`newsfirst_data`, `newswire_data`, `adaderana_data`) represent Sri Lankan news sources — this tells you the full RAG pipeline has an ingestion stage that stores news from these outlets.

> **Note**: `config.py` is defined but not imported in the current `server.py` or `mcp-client.py` — it is ready for when you wire up the actual vector search. Call `config.load_config()` anywhere you need these values.

---

## MCP Tools Explained

### 1. `get_time_with_prefix()`

**Purpose**: Returns the current date and time.  
**Input**: None  
**Output**: String like `"2026-03-22 16:45:32.123456"`

```python
@mcp.tool()
def get_time_with_prefix():
    """Get the current date and time."""
    return str(datetime.datetime.now())
```

Simple utility tool. Useful for time-aware queries like "What happened today in Sri Lanka?"

---

### 2. `extract_entities_tool(query: str)`

**Purpose**: Uses OpenAI to identify named entities in a query — people, places, organizations, events, dates, etc.  
**Input**: A text query string  
**Output**: JSON object where keys are entity types and values are lists

**Example:**
```
Input:  "What did President Ranil Wickremesinghe say in Colombo last Tuesday?"
Output: {
  "PERSON": ["Ranil Wickremesinghe"],
  "LOCATION": ["Colombo"],
  "DATE": ["last Tuesday"]
}
```

**Why this matters for RAG**: Before searching a vector database, knowing entities lets you apply metadata filters (e.g., filter articles that mention "Colombo") which dramatically improves retrieval precision.

---

### 3. `refine_query_tool(original_query: str)`

**Purpose**: Uses OpenAI to rewrite a user's query to be more effective for vector/semantic search.  
**Input**: The original query string  
**Output**: A refined (improved) query string

**Example:**
```
Input:  "What are the latest news about the president of Sri Lanka?"
Output: "Sri Lanka president recent statements policies decisions"
```

**Key safety logic**: If the refined query is more than 3× longer than the original, it falls back to the original. This prevents the LLM from turning a short query into a paragraph.

```python
if len(refined_query) > len(original_query) * 3:
    return original_query
```

---

### 4. `check_relevance(question: str, text_chunk: str)`

**Purpose**: Given a user question and a retrieved document chunk, uses OpenAI to score how relevant the document is to the question.  
**Input**: `question` (the user's query), `text_chunk` (a news article excerpt)  
**Output**: A float between 0.0 and 1.0

**Scoring scale:**
- `0.0 – 0.3`: Not relevant
- `0.4 – 0.6`: Somewhat relevant
- `0.7 – 1.0`: Highly relevant

**How it parses the score**: The LLM is prompted to always include a line like `SCORE: 0.8` in its response, which is then parsed:

```python
score_line = [line for line in result_text.split('\n') if 'SCORE:' in line]
relevance_score = float(score_line[0].split('SCORE:')[1].strip())
```

**Why this matters for RAG**: After vector search retrieves the top-N documents, this tool re-ranks them by true semantic relevance — this is called **LLM-based re-ranking** and significantly improves answer quality.

---

## Transport Modes: SSE vs STDIO

The server supports two ways of communicating:

| Mode | How it works | When to use |
|------|-------------|-------------|
| **SSE** (Server-Sent Events) | Server runs as an HTTP service; client connects via URL | Production, when server and client are separate processes or machines |
| **STDIO** | Client launches the server as a subprocess; they communicate via stdin/stdout | Simpler setups, Claude Desktop, local scripting |

**This project uses SSE** — that's why you start the server first, then the client connects to `http://localhost:8050/sse`.

The client also has a `connect_to_server()` function (currently commented out in `main()`) that demonstrates STDIO mode:

```python
# STDIO mode — server.py is launched as a subprocess
await connect_to_server("server.py")

# SSE mode — connects to a running server (currently active)
async with sse_client("http://localhost:8050/sse") as (read, write):
    ...
```

To switch to STDIO, uncomment `connect_to_server()` and comment out the `sse_client` block.

---

## Extending This Project

### Adding a new MCP Tool

In `server.py`, just add a decorated function:

```python
@mcp.tool()
def my_new_tool(input_text: str) -> str:
    """Describe what this tool does (OpenAI reads this description!)."""
    # Your logic here
    return "result"
```

The description is critical — OpenAI reads the docstring to decide when to use this tool.

### Connecting to MongoDB + Pinecone for Full RAG

1. Store your documents in MongoDB (three collections are pre-configured in `config.py`)
2. Create embeddings (using OpenAI's `text-embedding-3-small` or similar) and upsert to Pinecone
3. Add a new MCP tool `search_knowledge_base(query: str)` in `server.py` that:
   - Calls `refine_query_tool` to improve the query
   - Embeds the refined query
   - Queries Pinecone for top-N similar chunks
   - Calls `check_relevance` to re-rank results
   - Returns the most relevant chunks
4. The client's agentic loop will automatically use this tool when appropriate

### Changing the OpenAI Model

In your `.env` file:
```env
OPENAI_MODEL_NAME=gpt-4o        # More capable, higher cost
OPENAI_MODEL_NAME=gpt-4o-mini   # Default — fast and cost-effective
```

---

## Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Connection refused on port 8050` | Server is not running | Start `server.py` first |
| `OPENAI_API_KEY not found` | `.env` file missing or not loaded | Copy `.env.sample` → `.env` and add your key |
| `nest_asyncio` warning in terminal | Not an error — just a notice | Safe to ignore; it enables nested async loops |
| `AttributeError: 'NoneType' has no attribute 'list_tools'` | Client called a tool before `session.initialize()` | Ensure `await session.initialize()` runs before any tool calls |
| `Model not found` | Wrong model name in `.env` | Use `gpt-4o-mini`, `gpt-4o`, or another valid OpenAI model name |
| Tool returns `{"error": "..."}` | OpenAI call inside the tool failed | Check your `OPENAI_API_KEY` and model name; check OpenAI rate limits |

---

## Architecture Summary

```
.env
 └─ Secrets/config
       │
       ▼
config.py ──────────────────► (MongoDB + Pinecone settings, ready to use)
       
server.py (FastMCP Server, port 8050)
 ├─ get_time_with_prefix()       ← no LLM call
 ├─ extract_entities_tool()      ← calls OpenAI
 ├─ refine_query_tool()          ← calls OpenAI  
 └─ check_relevance()            ← calls OpenAI
       ↑ SSE (HTTP)
       │
mcp-client.py (AI Agent)
 ├─ connect via sse_client()
 ├─ get_mcp_tools() → convert to OpenAI function format
 └─ process_query()
      ├─ Ask OpenAI (with tools list)
      ├─ OpenAI says "call tool X"
      ├─ Execute tool on MCP server
      ├─ Send result back to OpenAI
      └─ Return final answer
```

---

## License

MIT — feel free to use, modify, and build on top of this.

---

*Built with [FastMCP](https://github.com/jlowin/fastmcp) · [OpenAI Python SDK](https://github.com/openai/openai-python) · [Model Context Protocol](https://modelcontextprotocol.io)*
