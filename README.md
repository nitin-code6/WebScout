# WebScout Agentic Web

An autonomous AI agent platform that browses, reasons, and executes multi-step tasks across the web on your behalf.

---

## Overview

WebScout is a full-stack autonomous web agent that bridges large language models with browser automation. Instead of just querying static APIs, WebScout uses a dynamic Reasoning and Acting (ReAct) execution loop combined with headless browser automation to navigate complex websites, parse dynamic JavaScript content, extract structured data, and interact with web forms.

The platform provides two ways to run:
1. Interactive Web Interface: A browser-based dashboard with live execution visualization and step-by-step progress tracking.
2. REST and WebSocket API: A production-ready FastAPI backend that streams agent thought processes and actions in real time.

---

## Architecture

The system operates on a continuous feedback loop:

User Input (Natural Language Goal)
  -> Task Decomposition
  -> LLM Planner (Azure OpenAI GPT-4o or OpenAI GPT-4o)
  -> ReAct Execution Loop (Think -> Act -> Observe -> Evaluate)
  -> Tool Execution Layer
       - Browser Automation (Playwright headless Chromium)
       - Web Search (DuckDuckGo and Bing Search)
       - Structured Data Extraction (LLM with BeautifulSoup)
       - Web Form Automation (Intelligent field detection and submission)
  -> Memory Layer
       - Short-Term Session Memory (Sliding context window)
       - Long-Term Vector Memory (Pinecone semantic storage)
  -> Real-Time WebSocket Streaming & Structured Output

### Key Design Principles

- ReAct Reasoning Loop: At every step, the agent analyzes its current progress, decides on the next appropriate action, executes the action using its tool suite, and observes the result.
- Resilient Error Recovery: If an action fails (such as an altered selector or page timeout), the error is added to context and the agent dynamically recalculates an alternative strategy.
- Real-Time Streaming: Agent reasoning, tool actions, and browser state changes are streamed via WebSockets as they happen.
- Dual-Tier Memory: Short-term memory prevents circular actions within a session, while vector database integration retains solutions for similar future queries.

---

## Project Structure

```
webscout-agent/
|-- index.html                        # Main frontend dashboard
|-- src/
|   |-- styles/
|   |   `-- main.css                  # UI styling and layout
|   `-- utils/
|       |-- agent-simulation.js       # Client-side agent simulation engine
|       |-- animations.js             # UI animations and counters
|       `-- main.js                   # Application state and event handling
|-- backend/
|   |-- main.py                       # FastAPI application (REST + WebSocket endpoints)
|   |-- config.py                     # Configuration and environment settings
|   |-- requirements.txt              # Python dependencies
|   |-- .env.example                  # Environment variable template
|   |-- agent/
|   |   |-- core.py                   # Central ReAct agent engine
|   |   |-- tools/
|   |   |   |-- browser.py            # Playwright browser automation
|   |   |   |-- search.py             # DuckDuckGo and Bing web search
|   |   |   |-- extractor.py          # LLM-powered structured data extraction
|   |   |   `-- form_filler.py        # Automated form filling and submission
|   |   `-- memory/
|   |       |-- session_memory.py     # Short-term context memory
|   |       `-- vector_store.py       # Long-term Pinecone vector memory
|   `-- utils/
|       `-- logger.py                 # Structured application logging
`-- README.md
```

---

## Quick Start

### 1. Frontend Demo (No Installation Required)

Open index.html directly in any web browser, or serve it locally:

```bash
# Using Python
python -m http.server 5500

# Using Node.js
npx serve .
```

Open http://localhost:5500 in your browser to test the interactive simulation dashboard.

### 2. Full-Stack Agent Setup

#### Prerequisites
- Python 3.10 or higher
- Node.js (optional, for local static serving)
- Azure OpenAI or OpenAI API credentials

#### Step 1: Set Up Backend Environment

```bash
cd backend

# Create and activate virtual environment
python -m venv venv

# On Linux / macOS:
source venv/bin/activate

# On Windows:
venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Install Playwright browser binaries
playwright install chromium
```

#### Step 2: Configure Environment Variables

Create a .env file inside the backend directory:

```bash
cp .env.example .env
```

Fill in your configuration:

```env
# Azure OpenAI (Preferred)
AZURE_OPENAI_KEY=your_azure_openai_key
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com/
AZURE_OPENAI_DEPLOYMENT=gpt-4o
AZURE_OPENAI_VERSION=2024-08-01-preview

# Standard OpenAI (Fallback)
OPENAI_API_KEY=your_openai_api_key

# Pinecone Vector Store (Optional, for long-term memory)
PINECONE_API_KEY=your_pinecone_api_key
PINECONE_INDEX=webscout-memory

# Bing Search API (Optional fallback for search)
BING_SEARCH_KEY=your_bing_search_key
```

#### Step 3: Run the Backend Server

```bash
cd backend
uvicorn main:app --reload --port 8000
```

The interactive API documentation will be available at http://localhost:8000/docs.

---

## API Reference

### REST Endpoints

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| GET | /api/health | Health check and system status |
| POST | /api/run | Trigger a new background agent task |
| GET | /api/status/{job_id} | Poll the status and results of a task |
| DELETE | /api/cancel/{job_id} | Cancel a running task |

#### Start a Task (POST /api/run)

Request:
```json
{
  "task": "Find top 5 artificial intelligence companies funded recently with their founder details",
  "mode": "autonomous",
  "max_steps": 20,
  "output_format": "json"
}
```

Response:
```json
{
  "job_id": "7b8f9e01-2a3b-4c5d-8e6f-1a2b3c4d5e6f",
  "status": "queued",
  "message": "Agent task started. Connect to /ws/agent/{job_id} for real-time stream."
}
```

### WebSocket Streaming

Connect to receive the agent's thought process, tool execution steps, and progress in real time:

```javascript
const socket = new WebSocket(
  "ws://localhost:8000/ws/agent?task=Compare laptop specifications under 1000 dollars&max_steps=20"
);

socket.onmessage = (event) => {
  const payload = JSON.parse(event.data);
  // payload.type: "connected" | "step" | "done" | "error"
  // payload.data: detailed step information including thought, tool, and observations
  console.log(payload);
};
```

---

## Supported Task Categories

- Market Research: Aggregate company insights, fundings, and industry metrics from multiple sources.
- E-Commerce Comparison: Extract specifications, customer ratings, and pricing across retail platforms.
- Travel Planning: Gather itinerary data, ticket availability, and accommodation comparisons.
- Content Aggregation: Monitor news, summarize technical updates, and generate structured summaries.
- Form Automation: Populate multi-step web forms, applications, and profile fields accurately.

---

## Safety and Guardrails

- Execution Modes:
  - autonomous: Runs end-to-end within defined step boundaries.
  - supervised: Pauses before executing state-changing actions for confirmation.
  - readonly: Restricts the agent to navigation and extraction, disabling form submissions.
- Step Cap: Configurable maximum iteration limit to prevent runaway loops.
- Hard Timeout: Automated process termination if an external site hangs.
- Domain Restrictions: Optional domain filtering to contain browsing within approved scopes.

---

## License

This project is licensed under the MIT License. See the LICENSE file for details.
