# Web-API
[![Web API](https://raw.githubusercontent.com/oxylabs/Web-API/main/web-api-logo.png)](https://oxylabs.io/products/web-api?utm_content=oxylabs-web-scraper-api&groupid=877)

<p align="left">
  <a href="https://dashboard.oxylabs.io"><img src="https://img.shields.io/badge/%F0%9F%9A%80_Get_Started-FF6B35?style=for-the-badge" alt="Get started"></a>
  <a href="https://developers.oxylabs.io/products/web-api"><img src="https://img.shields.io/badge/%F0%9F%93%9A_Documentation-4A90E2?style=for-the-badge" alt="Documentation"></a>
  <a href="https://dashboard.oxylabs.io"><img src="https://img.shields.io/badge/%F0%9F%93%8A_Dashboard-2EA44F?style=for-the-badge" alt="Dashboard"></a>
  <a href="https://discord.gg/Pds3gBmKMH"><img src="https://img.shields.io/badge/%F0%9F%92%AC_Discord-5865F2?style=for-the-badge" alt="Discord"></a>
  <a href="https://www.youtube.com/@oxylabs"><img src="https://img.shields.io/badge/%E2%96%B6%EF%B8%8F_YouTube-FF0000?style=for-the-badge" alt="YouTube"></a>
</p>

# Web API
- [What is Web API?](#what-is-web-api)
- [Quick start](#quick-start)
- [Features](#features)
- [Agent-native web access: AI agent integrations](#agent-native-web-access-ai-agent-integrations)
- [Skills or MCP?](#skills-or-mcp)
- [When to use Web API](#when-to-use-web-api)
- [How Web API differs from other Oxylabs scrapers](#how-web-api-differs-from-other-oxylabs-scrapers)
- [Resources](#resources)
- [Support](#support)

Welcome to the official repository overview for Oxylabs' [Web API](https://developers.oxylabs.io/products/web-api), a reliable, fresh web data collection API for applications and AI agents. This guide is a developer-focused, technical reference covering what Web API is, how it works, its key features, agent integrations, and when it fits your web scraping, web searching, or RAG pipeline workflows. It also serves as the central hub linking to all Web API-related repositories, code examples, and integrations.

## What is Web API?

Oxylabs Web API is a REST API that lets your code or your AI agent search the web and read web pages in real time. You send an HTTP request; it handles JavaScript rendering, server-side traffic challenges, proxy rotation, and geo-targeting, and returns the result in the format you ask for.

It exposes two endpoints, one for each stage of web data acquisition:

| Endpoint | Used for | Input |
| --- | --- | --- |
| [`/search`](https://developers.oxylabs.io/products/web-api/search) | Discovering and ranking organic web results (the Web Search API) | Keyword / query |
| [`/scrape`](https://developers.oxylabs.io/products/web-api/scrape) | Rendering and extracting page data as Markdown, HTML, JSON, or screenshot (web scraping and data extraction) | Page URL |

The endpoints are designed to be used together: search to find, scrape to read. Search results contain short snippets, so anything that needs to be accurate should come from a `/scrape` call.

### How it works

1. Send a query to `/v1/search` and get a ranked list of results with titles, URLs, and snippets.
2. Pick the pages worth reading and send their URLs to `/v1/scrape`, requesting `markdown` output.
3. Use the returned content in your application, your AI agent, or an LLM pipeline.

All requests go to `https://webapi.oxylabs.io` and are authenticated with an API key sent as a Bearer token.

## Quick start

You can be making requests in three steps – get an API key, search your query, then scrape.

### 1. Get an API key

In the [Oxylabs dashboard](https://dashboard.oxylabs.io), create a Web API instance, copy the API key from that instance, and export it in your terminal or project environment:

```bash
export OXYLABS_WEB_API_KEY=your_api_key_here
```

See the [Authentication](https://developers.oxylabs.io/products/web-api/authentication) steps for key handling and rotation.

### 2. Search the web

```bash
curl https://webapi.oxylabs.io/v1/search \
  -H "Authorization: Bearer $OXYLABS_WEB_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"query": "eu ai act compliance deadlines", "max_results": 3}'
```

The response contains a ranked `results` array. Each result has a `title`, a `url`, an `outline` (snippet), and its `metadata.position`:

```json
{
  "state": "done",
  "results": [
    {
      "title": "Implementation Timeline | EU Artificial Intelligence Act",
      "outline": ["Date 2 August 2025 Providers: need to be compliant with the AI Act by 2 August 2027..."],
      "url": "https://artificialintelligenceact.eu/implementation-timeline/",
      "metadata": { "position": 1 }
    }
  ],
  "params": { "query": "eu ai act compliance deadlines", "location": null, "max_results": 3 },
  "metadata": { "timestamp": 1787431568, "request_id": "1788875675880097469" }
}
```

### 3. Scrape a page

Read a page returned by `/search`, or any URL from your own list:

```bash
curl https://webapi.oxylabs.io/v1/scrape \
  -H "Authorization: Bearer $OXYLABS_WEB_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{
      "url": "https://artificialintelligenceact.eu/implementation-timeline/",
      "output": ["markdown"]
  }'
```

The page content is returned in `results[0].markdown`.

### Python: search-to-scrape in one workflow

Search snippets are truncated, so they are not enough for deep analysis or LLM pipelines. The standard pattern is to query `/v1/search` to discover links, then send those URLs to `/v1/scrape` with Markdown output:

```python
import os, requests

API = "https://webapi.oxylabs.io"
HEADERS = {"Authorization": f"Bearer {os.environ['OXYLABS_WEB_API_KEY']}"}


def research(question: str, read_top: int = 2) -> list[dict]:
    # 1. Discover target URLs
    hits = requests.post(
        f"{API}/v1/search",
        headers=HEADERS,
        json={"query": question, "max_results": 10},
        timeout=120,
    )
    hits.raise_for_status()
    # 2. Extract full page content as Markdown
    pages = []
    for hit in hits.json()["results"][:read_top]:
        page = requests.post(
            f"{API}/v1/scrape",
            headers=HEADERS,
            json={"url": hit["url"], "output": ["markdown"]},
            timeout=120,
        )
        if not page.ok:
            print(f"skipped {hit['url']}: {page.status_code}")
            continue
        pages.append(
            {
                "url": hit["url"],
                "title": hit["title"],
                "content": page.json()["results"][0]["markdown"],
            }
        )
    return pages


for page in research("eu ai act compliance deadlines"):
    print(f"--- {page['title']} ({page['url']})")
    print(page["content"][:500])
```

Node.js and Python snippets for each endpoint are in the [official documentation](https://developers.oxylabs.io/products/web-api). Using an AI agent instead? Jump to [Agent-native web access](#agent-native-web-access-ai-agent-integrations).

**Ready to try it?** Claim the welcome credits and [start for free](https://dashboard.oxylabs.io).

## Features

Web API combines web searching, web scraping, and data extraction behind one REST interface. Access handling (JavaScript rendering, traffic challenges, proxy rotation, and geo-targeting) is done by the service, so your code only sends a query or a URL.

### Web Search API: `/v1/search`

Returns ranked organic web results for a keyword or query. Each result includes a title, a URL, a short snippet (`outline`), and its position. Set `max_results` to control how many results you get; the response echoes the `query`, `location`, and `max_results` it used. Search is synchronous only.

Snippets are truncated, so use `/search` to *discover* pages and `/scrape` to *read* them. Full parameters are in the [/search documentation](https://developers.oxylabs.io/products/web-api/search).

### Web scraping: `/v1/scrape`

Fetches a page by URL and handles headless rendering, proxy rotation, access challenges, and localization. Request parameters:

| Parameter | What it does | Notes |
| --- | --- | --- |
| `url` | Absolute `http` / `https` page to fetch | Required |
| `output` | Response format: `markdown`, `html`, `json`, `screenshot`; several can be combined | Default: `html` |
| `json` | Custom AI parsing with a `prompt` or a `schema` | Used with `output: ["json"]` |
| `location` | Two-letter country code (e.g. `US`, `DE`, `JP`) for geo-targeted results | Optional |
| `device` | `desktop` or `mobile` user-agent and viewport emulation | Default: `desktop` |
| `run_js` | Execute the page's JavaScript before capture | Default: `false` |

**Output formats**

- **`markdown`**: the page converted to clean, structured Markdown, optimized for LLM context windows to minimize token usage.
- **`html`**: raw page markup, for when you need attributes, embedded JSON-LD, or inline data.
- **`json`**: structured data extracted in the same request as the fetch, with no selectors to write or maintain.
- **`screenshot`**: a base64-encoded PNG of the rendered page. Requires `run_js: true`.

**JavaScript rendering (`run_js`).** Set `run_js: true` for client-side scripts, dynamic requests, and single-page applications. Rendering increases request time, so use a client timeout of at least 150 seconds and skip it for static pages and feeds.

**Geo-targeting (`location`).** Requests are routed through localized proxy exit nodes so that regional pricing, language variations, and region-exclusive content are served correctly.

Full parameters and the output dictionary are in the [/scrape documentation](https://developers.oxylabs.io/products/web-api/scrape).

### AI-powered data extraction (AI Parsing)

Request `output: ["json"]` and add a `json` object to extract custom fields from any page in the same call. Choose one of two modes per request:

| Mode | You provide | Best for |
| --- | --- | --- |
| `prompt` | A natural-language description of the fields to extract | Prototyping, exploring a page, simple key-value extraction |
| `schema` | An OpenAPI / JSON Schema object the result must match | Production pipelines and typed applications |

```json
{
  "url": "https://example.com/product/10928",
  "output": ["json"],
  "json": {
    "prompt": "Extract the main product title, current sale price as a number, currency code, and a list of image URLs."
  }
}
```

> **Note:** AI parsing uses significantly more account credit than a regular `/scrape` request. See [AI Parsing](https://developers.oxylabs.io/products/web-api/scrape/ai-parsing) for schema patterns and localized extraction.

### Dedicated scrapers

Beyond generic URL scraping, target-specific scrapers are available at `POST /v1/scrape/{target}/{collection}`. They take target-specific parameters (for example ASINs or store zip codes) and return structured fields under `results[].json`. Target overviews cover Google, Amazon, YouTube, Bing, AI & LLMs, and Commerce. Head over to [Dedicated Scrapers](https://developers.oxylabs.io/products/web-api/scrape/dedicated-scrapers) documentation to learn about specific targets.

```bash
curl https://webapi.oxylabs.io/v1/scrape/amazon/product \
  -H "Authorization: Bearer $OXYLABS_WEB_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "B0935DN1BN",
    "output": ["json"],
    "location": "10115",
    "domain": "de",
    "currency": "EUR"
  }'
```

### Delivery modes

Three ways to receive `/scrape` results, depending on volume and latency needs:

| Delivery | Endpoint | Max items per request | Used for |
| --- | --- | --- | --- |
| Realtime (synchronous) | `POST /v1/scrape/{source}` | 1 | Low-latency requests, testing |
| Async single (polling) | `POST /v1/async/scrape/{source}` | 1 | High concurrency and volume |
| Async batch (bulk queue) | `POST /v1/async/scrape/batch/{source}` | 5,000 | Bulk ingestion |

Async results can be retrieved by polling, by webhook (`callback_url`), or by delivery to cloud storage (`s3`, `s3_gzip`, `s3_compatible`, `gcs`, or `tos`). Batch routing is available for select sources only, and some endpoints (for example YouTube downloads, ChatGPT, Gemini, and Perplexity) do not support real-time delivery. Visit the [Delivery Modes](https://developers.oxylabs.io/products/web-api/scrape/delivery-modes) page for more information.

### Authentication, errors, and request IDs

Every request uses a Bearer API key from a Web API instance (check the [Authentication](https://developers.oxylabs.io/products/web-api/authentication) page). Each response carries a `metadata.request_id`; include it in support tickets. For HTTP status codes and retry guidance, refer to [Troubleshooting](https://developers.oxylabs.io/products/web-api/troubleshooting).

## Agent-native web access: AI agent integrations

Web API is built to be plugged into AI agents, coding assistants (such as Claude Code and Cursor), and custom agent frameworks. There are three ways to connect it, and they can be combined.

| | Integration prompt | Agent skills | MCP server |
| --- | --- | --- | --- |
| What it is | A prompt you paste into an AI coding agent so it builds the integration for your project | Instructions that teach an agent how to use Web API | A server that exposes Web API as callable tools |
| Gives the agent | A spec to follow (base URL, auth, schemas, retry rules, checklist) | Judgment: when to search, which sources to trust, how to cite | Typed tools it can call directly |
| Needs a process | No | No | Yes |
| Works in | Agents that can read a URL or a pasted page | Agents that load skill directories | Any MCP client |

### Option 1: Integration prompt

Paste the integration prompt from the [Web API documentation](https://developers.oxylabs.io/products/web-api) into an AI coding agent (Claude Code, Cursor, Copilot). The agent reads the single-page [For Agents](https://developers.oxylabs.io/products/web-api/for-agents) specification, which contains the base URL, authentication, both endpoint schemas, reference code, error and retry rules, and a checklist. The agent decides whether your project's code will call the API at runtime or whether the agent itself needs web access in the current session, and then proceeds accordingly.

If your agent can't fetch URLs, `for-agents.md` is plain, self-contained Markdown that you can paste in directly.

### Option 2: Agent skills

```bash
npx skills add oxylabs/web-api-skills
```

The [`oxylabs/web-api-skills`](https://github.com/oxylabs/web-api-skills) repository contains three skills:

| Skill | Covers |
| --- | --- |
| `oxylabs-web-api` | Mechanics: MCP tools, parameters, latency, errors, retries, plus a dependency-free CLI helper |
| `oxylabs-web-research` | The research method: search, pick sources, read, cite |
| `migrate-to-oxylabs` | Porting a codebase off Tavily, Exa, Firecrawl, Perplexity, Brave, or Linkup |

Set `OXYLABS_WEB_API_KEY` from a Web API instance (a key from another Oxylabs product will return `401`), start a new session, and run `/skills` to confirm all three loaded. Details can be found in the [Agent Skills](https://developers.oxylabs.io/ai-workflows/agent-skills) page.

### Option 3: MCP server

The remote and self-hosted [Web API MCP server](https://github.com/oxylabs/web-api-mcp) (`oxylabs-web-api-mcp`) lets any MCP-compliant client search the web, read pages as Markdown, extract fields as JSON, and call dedicated scrapers.

```bash
uv tool install git+https://github.com/oxylabs/web-api-mcp

# Claude Code
claude mcp add oxylabs-web-api -- oxylabs-web-api-mcp
```

For Claude Desktop or Cursor, add this to your MCP configuration:

```json
{
  "mcpServers": {
    "oxylabs-web-api": {
      "command": "oxylabs-web-api-mcp",
      "env": { "OXYLABS_WEB_API_KEY": "your_api_key_here" }
    }
  }
}
```

**Available tools**

| Tool | What it does |
| --- | --- |
| `search` | Search the live web (`query`, `max_results`, `location`) |
| `scrape` | Read one URL, Markdown by default (`url`, `format`, `location`, `device`, `run_js`) |
| `extract` | Pull named fields off a page as JSON (`url`, `prompt`, `location`, `run_js`) |
| `check_scrape` | Collect a JavaScript-rendering job (`job_id`) |
| `read_scraped` | Read a large page that was offloaded to disk, in chunks |
| `list_scrapers` | List implemented scrape endpoints, or describe one |
| `scrape_target` | Call a target-specific endpoint (`endpoint`, `params`) |

Prerequisites are a Web API key plus `uv` and Python 3.10 or higher. The server can also run over HTTP for a team or for agents that can't start local processes; it has no authentication of its own, so keep it behind a VPN or an authenticated ingress. Configuration variables (rate limit, timeout, retries) are listed in the [MCP documentation](https://developers.oxylabs.io/ai-workflows/mcp).

**Claude Code plugin.** Claude Code can take the skills repository as a plugin, which wires the skills and the MCP server together:

```bash
/plugin marketplace add oxylabs/web-api-skills
/plugin install oxylabs-web-api
```

## Skills or MCP?

Installing both is a reasonable default. The MCP server bundles the `oxylabs-web-api` skill and serves it as the `oxylabs://skill/web-api` resource, so connecting the server alone gives an agent both halves. Install the skills separately when you want them without a running server, or when you want to `migrate-to-oxylabs`.

### Agent tips from the docs

- If a scrape comes back empty or skeletal, retry once with `run_js: true`. If the site is on a country TLD (such as `.lt`, `.es`, or `.co.uk`) and it is still empty, retry once more with `run_js` and `location` set to that country.
- Scrape the two or three results that plausibly answer the question rather than all ten, and raise concurrency gradually until a `429` appears.
- Never print or hard-code the API key; keep it in an environment variable.

### Using Web API in a RAG pipeline

For LLM workflows such as a RAG pipeline, use the search-to-scrape pattern from the [Quick start](#quick-start): `/v1/search` discovers authoritative links, then `/v1/scrape` with `output: ["markdown"]` returns clean page content that is optimized for LLM context windows. When the pipeline needs typed records rather than text, request `output: ["json"]` with a `schema` instead.

## When to use Web API

Reach for Web API when your application or AI agent needs live web data rather than a static dataset. Each use case below maps to scenarios shown in the documentation.

| Use case | How Web API fits | What to use |
| --- | --- | --- |
| Live web research for AI agents | The agent searches, picks sources, reads them in full, and cites them | `/search` then `/scrape` (Markdown), via skills or MCP |
| Grounding an LLM or RAG pipeline in current pages | Clean Markdown keeps token usage low when pages go into a context window | `/scrape` with `output: ["markdown"]` |
| Extracting structured fields from any page | Describe fields in plain language or pin a schema; no selectors to maintain | `/scrape` with `output: ["json"]` and `prompt` or `schema` |
| Scraping JavaScript-heavy pages and SPAs | Render the page in a headless browser before capture | `run_js: true`, optionally `screenshot` output |
| Localized pricing, language, and regional content | Route requests through country-specific exit nodes | `location`, `device` |
| Search results, e-commerce, and video data from known targets | Pre-built scrapers return parsed JSON for supported targets | Dedicated scrapers (Google, Amazon, YouTube, Bing, and more) |
| High-volume collection | Queue jobs and collect results by polling, webhook, or cloud storage | Async single or async batch (up to 5,000 jobs per request) |
| Article and blog metadata for a database | Extract headline, author, publish date, and tags into typed records | `schema`-based AI parsing |
| Moving an existing integration | An agent skill ports a codebase off Tavily, Exa, Firecrawl, Perplexity, Brave, or Linkup | `migrate-to-oxylabs` skill |

### Which approach fits?

- **You know the page URL:** call `/scrape` directly.
- **You only have a question or keyword:** call `/search` first, then `/scrape` the best results.
- **You need a few custom fields from an unusual layout:** use AI parsing with a `schema`. It costs more credit than a plain scrape, so use it where extraction logic would otherwise need maintenance.
- **You need standardized fields from a supported target** (for example an Amazon product or a Google search page): use a dedicated scraper.
- **You process thousands of URLs:** use async batch delivery rather than restricting real-time requests.

## How Web API differs from other Oxylabs scrapers

Oxylabs has several products for collecting web data. Web API is the one organized around two steps, search and scrape, with Markdown and AI-parsed JSON output and ready-made connections for AI agents. The table summarizes how it compares with the other scraping products; pick the most specific product that fits your task.

| Criteria | Web API | Web Scraper API | Web Unblocker | Agent Browser |
| --- | --- | --- | --- | --- |
| Main purpose | Web search and page data collection through one REST interface | All-in-one data collection from public websites: send a request, get structured results | Proxy-style access to raw web content with automatic access handling | Full remote browser automation |
| Request input | Keyword (`/search`) or page URL (`/scrape`) | URL or query, plus a `source` | URL through a proxy connection | An automation script (Puppeteer, Playwright, CDP) |
| Output | Ranked search results; page as Markdown, HTML, JSON, or screenshot | Raw HTML, structured JSON, screenshot, or Markdown | Full HTML | HTML, screenshots, and whatever your script extracts |
| Custom field extraction | AI parsing with a `prompt` or `schema` in the same request | Dedicated parsers and Custom Parser | You parse the HTML yourself | Your script |
| Authentication | Bearer API key from a Web API instance | HTTP Basic Auth (username and password) | Username and password | Username and password |
| AI agent connections | Integration prompt, agent skills, and MCP server | Agent skill in `oxylabs/agent-skills` | Agent skill in `oxylabs/agent-skills` | Agent skill and MCP server |

### Web API vs. Web Scraper API

Both return page data without you managing proxies or rendering. They differ in how requests are made and what comes with them:

| Criteria | Web API | Web Scraper API |
| --- | --- | --- |
| Request model | Endpoint-based: `POST /v1/search` and `POST /v1/scrape` | Source-based: a `source` such as `universal` or a dedicated source (for example `amazon_product`) |
| Base URL | `https://webapi.oxylabs.io` | `https://realtime.oxylabs.io/v1/queries` (Realtime), `https://data.oxylabs.io/v1/queries` (Push-Pull) |
| Authentication | Bearer API key | HTTP Basic Auth (username and password) |
| Web search | Web Search API (`/v1/search`) | Search engine dedicated sources (Google, Bing) |
| Output | Markdown, HTML, JSON, screenshot | Raw HTML, structured JSON, screenshot, Markdown |
| JavaScript rendering | `run_js` | `render` |
| Custom field extraction | AI parsing with a `prompt` or `schema` | Dedicated parsers and Custom Parser (`parse`) |
| Dedicated targets | `/v1/scrape/{target}/{collection}` (Google, Amazon, YouTube, Bing, and more) | Dedicated sources (search engines, AI platforms, e-commerce, and more) |
| Delivery | Realtime, async single, async batch (up to 5,000 jobs) | Realtime, Push-Pull (batch up to 5,000), Proxy Endpoint |
| Extras | Integration prompt, agent skills, MCP server | Browser instructions, Scheduler, OxyCopilot |

### Web API vs. Web Unblocker and Agent Browser

Pick by how much of the pipeline you want to own:

| Product | Choose it when | What you handle |
| --- | --- | --- |
| Web API | You want search results or page content returned over a plain REST call | The request and what you do with the result |
| Web Unblocker | You already have a scraping pipeline and want a proxy-style layer that handles server-side defense challenges | Parsing the HTML |
| Agent Browser | You need real browser control: clicks, form fills, login state, multi-step interactions | The automation script |

**Credentials are product-specific.** An API key from another Oxylabs product won't work with Web API (it returns `401`). Create a Web API instance in the [dashboard](https://dashboard.oxylabs.io) and generate the key from it.

### Oxylabs services

| If you need to… | Use |
| --- | --- |
| Give an AI agent live web search and page reading, with cited sources | [Web API](https://developers.oxylabs.io/products/web-api) |
| Get structured data from supported targets, search results, or e-commerce pages | [Web Scraper API](https://developers.oxylabs.io/products/web-scraper-api) |
| Fetch protected pages without browser interactions | [Web Unblocker](https://developers.oxylabs.io/products/web-unblocker) |
| Click, fill forms, take screenshots, or automate with Playwright or Puppeteer | [Agent Browser](https://developers.oxylabs.io/products/agent-browser) |
| Use your own HTTP client with proxy authentication, rotation, or sticky sessions | [Proxies](https://developers.oxylabs.io/products/proxies) |

## Resources

### Try Web API

- Start a trial in the  [Oxylabs dashboard](https://dashboard.oxylabs.io): create a Web API instance and generate your API key.
- Read the [documentation](https://developers.oxylabs.io/products/web-api): the full, up-to-date reference for every endpoint and parameter.

### Learning resources for developers

| Resource | What you will learn |
| --- | --- |
| [Web API overview](https://developers.oxylabs.io/products/web-api) | Search-to-scrape workflow, manual integration, and AI agent integration |
| [Authentication](https://developers.oxylabs.io/products/web-api/authentication) | Key handling, header requirements, rotation, and troubleshooting |
| [/search](https://developers.oxylabs.io/products/web-api/search) | Request parameters and response schema for the Web Search API |
| [/scrape](https://developers.oxylabs.io/products/web-api/scrape) | Request parameters, output formats, JavaScript rendering, and geo-targeting |
| [Dedicated Scrapers](https://developers.oxylabs.io/products/web-api/scrape/dedicated-scrapers) | Target-specific parsed endpoints (Google, Amazon, YouTube, Bing, and more) |
| [AI Parsing](https://developers.oxylabs.io/products/web-api/scrape/ai-parsing) | Structured JSON extraction with prompts and JSON Schemas |
| [Delivery Modes](https://developers.oxylabs.io/products/web-api/scrape/delivery-modes) | Realtime, async single-job, and batch delivery; webhooks and cloud storage |
| [For Agents](https://developers.oxylabs.io/products/web-api/for-agents) | A single-page specification written for AI agents |
| [Agent Skills](https://developers.oxylabs.io/ai-workflows/agent-skills) | Installing and choosing between skills and MCP |
| [MCP Server](https://developers.oxylabs.io/ai-workflows/mcp) | Tools, installation, hosting, and configuration |
| [Troubleshooting](https://developers.oxylabs.io/products/web-api/troubleshooting) | HTTP status codes, retry logic, and troubleshooting tips |
| [API Reference](https://developers.oxylabs.io/api-reference) | Endpoint-level reference, including [Core Endpoints](https://developers.oxylabs.io/api-reference/core-endpoints) |

Every documentation page is also available as Markdown by adding `.md` to its URL, and a complete index for LLMs is published at [llms.txt](https://developers.oxylabs.io/llms.txt).

### Related repositories

| Repository | Description |
| --- | --- |
| [`oxylabs/web-api-mcp`](https://github.com/oxylabs/web-api-mcp) | Self-hosted MCP server for Web API (search, scrape, extract, dedicated scrapers) |
| [`oxylabs/web-api-skills`](https://github.com/oxylabs/web-api-skills) | Agent skills for Web API: mechanics, web research method, and migration |
| [`oxylabs/web-scraper-api`](https://github.com/oxylabs/web-scraper-api) | Overview repository for Oxylabs Web Scraper API |
| [`oxylabs/agent-skills`](https://github.com/oxylabs/agent-skills) | Agent skills for Proxies, Agent Browser, Web Unblocker, Video Data, and Web Scraper API |
| [`oxylabs/oxylabs-ab-mcp`](https://github.com/oxylabs/oxylabs-ab-mcp) | MCP server for Agent Browser |

## Support

If you have any questions or issues, email us in [support@oxylabs.io](mailto:support@oxylabs.io) or use the live chat in the [Oxylabs dashboard](https://dashboard.oxylabs.io). When you report a problem, include the `metadata.request_id` from the response.

Follow us on [LinkedIn](https://www.linkedin.com/company/oxylabs-io/), [X](https://x.com/oxylabs), or subscribe to our [YouTube](https://www.youtube.com/@oxylabs) channel.
