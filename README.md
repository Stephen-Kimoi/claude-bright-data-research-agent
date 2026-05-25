# Startup Research Agent

A fully programmatic AI agent that researches any startup and produces a structured report. Built with the **Claude API** (tool use) and **Bright Data** (SERP API + Web Unlocker).

This is the companion repository for the [Lablab.ai tutorial]([https://lablab.ai](https://lablab.ai/ai-tutorials/claude-bright-data-research-agent-for-ai-hackathons) — *Build a Startup Research Agent with Claude API and Bright Data*.

---

## What it does

Give the agent a company name. It will:
1. Fire multiple Google searches via Bright Data's SERP API
2. Scrape relevant pages (About pages, Crunchbase profiles, news articles) via Bright Data's Web Unlocker
3. Feed all gathered data back to Claude, which synthesizes a structured research report covering founders, funding, business model, recent news, and notable customers

---

## Architecture

```
User Input (company name)
        │
        ▼
┌───────────────────┐
│   Claude API      │  ← claude-sonnet-4-6
│   (tool_use loop) │
└────────┬──────────┘
         │ decides which tool to call
         ▼
┌─────────────────────────────────┐
│         Tool Dispatcher         │
├─────────────────┬───────────────┤
│  search_web()   │  scrape_url() │
│                 │               │
│  Bright Data    │  Bright Data  │
│  SERP API       │  Web Unlocker │
│  (Google JSON)  │  (raw HTML)   │
└─────────────────┴───────────────┘
         │
         ▼ tool results fed back to Claude
┌───────────────────┐
│  Final Report     │  ← structured markdown
└───────────────────┘
```

### Key design decisions

- **Agentic tool-use loop**: Claude drives the research autonomously. It decides how many searches to run, which URLs to scrape, and when it has enough data to write the report. The loop runs until `stop_reason == "end_turn"`.
- **Two Bright Data products**: SERP API returns structured JSON (title, URL, snippet) from Google — no proxy setup needed. Web Unlocker bypasses bot protection on any target URL, returning clean HTML that the agent strips and reads.
- **Streaming events**: `run_agent_stream()` is a Python generator that yields typed events (`tool_call`, `tool_result`, `thinking`, `report`). The Flask app wraps this in Server-Sent Events (SSE) so the UI updates in real time without polling.

---

## Project structure

```
claude-bright-data-research-agent/
├── agent.py          # Core agent logic + Bright Data tool implementations
├── app.py            # Flask web server with SSE streaming endpoint
├── templates/
│   └── index.html    # Frontend UI (vanilla JS, no framework)
├── requirements.txt
├── .env              # API keys (not committed)
└── .gitignore
```

---

## Prerequisites

- Python 3.10+
- An [Anthropic API key](https://console.anthropic.com)
- A [Bright Data account](https://brightdata.com) with:
  - A **SERP API** zone created (note the zone name)
  - A **Web Unlocker** zone created (note the zone name)

---

## Setup

**1. Clone the repo**
```bash
git clone https://github.com/Stephen-Kimoi/claude-bright-data-research-agent.git
cd claude-bright-data-research-agent
```

**2. Create and activate a virtual environment**
```bash
python3 -m venv venv
source venv/bin/activate        # macOS/Linux
venv\Scripts\activate           # Windows
```

**3. Install dependencies**
```bash
pip install -r requirements.txt
```

**4. Configure environment variables**

Create a `.env` file in the project root:
```
ANTHROPIC_API_KEY=your_anthropic_api_key
BRIGHT_DATA_API_KEY=your_bright_data_api_key
SERP_ZONE=your_serp_zone_name
UNLOCKER_ZONE=your_web_unlocker_zone_name
```

Getting your Bright Data credentials:
- Log in at [brightdata.com](https://brightdata.com)
- Go to **Web Access API** in the left sidebar
- Create a **SERP API** zone — copy the zone name
- Create a **Web Unlocker** zone — copy the zone name
- Your API key is shown on the dashboard home page

---

## Running the web UI

```bash
python app.py
```

Open [http://localhost:5000](http://localhost:5000) in your browser. Type a company name and click **Research**.

## Running from the terminal

```bash
python agent.py "Mistral AI"
```

---

## How the agent loop works

```python
# Simplified version of the core loop in agent.py

messages = [{"role": "user", "content": f"Research {company_name}"}]

while True:
    response = client.messages.create(
        model="claude-sonnet-4-6",
        tools=TOOLS,          # search_web + scrape_url definitions
        messages=messages,
    )

    if response.stop_reason == "end_turn":
        return response  # Claude is done — final report is in here

    # Claude called a tool — execute it and feed the result back
    for tool_call in response.tool_use_blocks:
        result = search_web(tool_call.input) or scrape_url(tool_call.input)
        messages += [assistant_turn, tool_result_turn]
    # loop continues
```

Each iteration Claude receives fresh tool results and decides whether to search more, scrape more, or write the final report.

---

## Example output

Researching **Perplexity AI** produces a report with:
- 9 funding rounds ($1.72B total raised, $20B valuation)
- All 4 founders with backgrounds
- Full product suite (answer engine, Sonar API, Comet browser)
- Legal controversies (NYT lawsuit, web scraping allegations)
- Major partnerships (SoftBank, Snap, Telefónica)

---

## Extending the agent

A few ideas for going further:

- **Add a `save_report` tool** — let Claude write the report directly to a PDF or Notion page
- **Multi-company comparison** — run the agent on a list and generate a comparison table
- **Competitor discovery** — add a tool that takes a company and returns its top 5 competitors
- **Export to CSV** — extract structured fields (funding, founders) from the markdown report

---

## Built with

| Tool | Purpose |
|---|---|
| [Claude API](https://docs.anthropic.com) | Orchestrating research via tool use |
| [Bright Data SERP API](https://brightdata.com/products/serp-api) | Real-time Google search results |
| [Bright Data Web Unlocker](https://brightdata.com/products/web-unlocker) | Bypassing bot protection on target pages |
| [Flask](https://flask.palletsprojects.com) | Web server + SSE streaming |

---

## License

MIT
