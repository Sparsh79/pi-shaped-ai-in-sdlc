# Stay Finder Agent
### Day 2 – AI in SDLC Workshop | `pi-shaped-ai-in-sdlc/day2-agents-workshop`

---

## Use Case

**Problem:** Finding the right Airbnb or homestay at the right price is time-consuming. Users open multiple tabs, manually filter by price, read reviews, and repeat the process every time their preferences change, especially frustrating when planning trips on a budget.

**Agent Solution:** A conversational AI agent called **Dora The Explorer** that takes your destination, budget, and preferences,  then instantly fetches stay options, checks the weather at the destination, and calculates cost splits for groups. The agent remembers all your preferences within the session so you never have to repeat yourself.

**Primary User:** A 25–28 year old planning a weekend or holiday trip, looking for budget-conscious quality stays without the research overhead.

**Example Inputs the User Might Send:**
1. `Find me an Airbnb in Manali, ₹2500 per night for 2 people`
2. `What about Kasol instead? Same budget`
3. `We are 5 friends, villa costs ₹9000 per night for 2 nights,  cost per person?`
4. `Show me pet-friendly options with a mountain view`
5. `How is the weather there this weekend?`

---

## LLM Used

**Model:** `models/gemini-2.5-flash-lite`

**Why this model:**
Gemini 2.5 Flash is Google's stable Flash model available via AI Studio with a free tier,  no credit card required. It supports function and tool calling, has a 1 million token context window which is ideal when search results return large JSON payloads, and responds quickly. Zero cost for a demo use case like this.

---

## Tools Used

### Tool 1: `search_stays` (HTTP Request Tool)
Calls the **SerpAPI Google Search API** to find Airbnb listings, homestays, and short-term rentals at a given location within a budget. The agent dynamically constructs a rich search query based on the user's location, budget, guest count, and preferences like mountain view, pet-friendly, or riverside. SerpAPI returns Google search results which Gemini parses and summarises into ranked recommendations.

- **API:** `https://serpapi.com/search`
- **Key parameters:** `q={query}`, `engine=google`, `gl=in` (India locale)
- **Free tier:** 100 searches/month at serpapi.com

### Tool 2: `get_weather` (HTTP Request Tool)
Calls **wttr.in**, a completely free and open weather API requiring no API key. Returns current temperature, weather condition, humidity, wind speed, and a 3-day forecast in JSON format. The agent always calls this tool first when a destination is mentioned so users get critical context like snowfall warnings or ideal conditions before booking.

- **API:** `https://wttr.in/{location}?format=j1`
- **Free tier:** Unlimited, no signup required

### Tool 3: `calculate` (n8n Built-in Calculator Tool)
The native n8n Calculator node. Used for cost splits among group members, total trip cost calculations, and currency conversions (INR to USD, EUR etc.). Requires zero configuration — just connect it to the AI Agent node.

---

## Memory Used

**Memory Node:** `Window Buffer Memory` (last 20 messages)

**How it helps:**
Window Buffer Memory stores the last 20 messages of the conversation in memory. This means when a user says "I prefer no parties and a mountain view" in Turn 1, the agent automatically applies those preferences when the user asks "what about Kasol instead?" in Turn 5. Budget and guest count are remembered throughout,  the user never has to repeat them. Follow-ups like "too expensive" or "show me something more central" are understood in full context.

The key trade-off is that it resets when the session ends, preferences are not saved between separate conversations. For a demo this is perfectly sufficient and requires zero infrastructure setup compared to Postgres or Redis memory.

---

## Workflow Screenshot

> <img width="1166" height="567" alt="Screenshot 2026-04-23 084959" src="https://github.com/user-attachments/assets/596e1eb8-9599-440a-959a-016272b1eff4" />


The workflow structure:

```
[Chat Trigger]
      │
      ▼
[AI Agent — "Asha"]
      │
      ├── ai_languageModel ◄── [Google Gemini 2.5 Flash]
      ├── ai_memory        ◄── [Window Buffer Memory]
      ├── ai_tool          ◄── [search_stays]
      ├── ai_tool          ◄── [get_weather]
      └── ai_tool          ◄── [calculate]
```

| Node | Type | Purpose |
|------|------|---------|
| Chat Trigger | Trigger | Starts workflow when user sends a chat message |
| AI Agent | Agent Core | Orchestrates LLM reasoning, memory, and tool calls |
| Google Gemini 2.5 Flash | Chat Model | Powers reasoning, summarisation, and conversation |
| Window Buffer Memory | Memory | Retains last 20 messages for context continuity |
| search_stays | HTTP Request Tool | Fetches stay listings via SerpAPI |
| get_weather | HTTP Request Tool | Fetches destination weather via wttr.in |
| calculate | Calculator Tool | Performs cost splits and currency conversions |

---

## Agent in Action

### Screenshot 1 — Tool Invoked + Weather Check

**Input:**
> `Find me an Airbnb in Manali, ₹2500 per night for 2 people`

**What happens internally:**
1. Agent calls `get_weather` for Manali → returns temperature and forecast
2. Agent calls `search_stays` with query "best airbnb Manali under 2500 INR per night 2 guests"
3. Agent formats results into scored cards with weather note included

> <img width="1850" height="1034" alt="image" src="https://github.com/user-attachments/assets/44abf1aa-a3b9-4396-9865-80e2d688a724" />


---

### Screenshot 2 — Memory Working

**Input (later in same session):**
> `What about Kasol instead?`

**What to observe:** Agent does not ask for budget or guest count again, it remembers ₹2500 and 2 guests from earlier. It immediately calls weather and search for Kasol with the same parameters.

> <img width="1850" height="1040" alt="image" src="https://github.com/user-attachments/assets/30219265-ed72-49e6-8364-73a9e8f76c71" />


---

### Screenshot 3 — Calculator Tool Invoked

**Input:**
> `We are 5 friends, found a cottage for ₹9000 per night for 2 nights, cost per person?`

**What happens:** Agent calls `calculate` with `(9000 × 2) / 5` and returns ₹3,600 per person.

> <img width="1826" height="1046" alt="image" src="https://github.com/user-attachments/assets/8560e446-dfd5-4415-8172-aae078d267ca" />


---

## Reflection

### What does your agent do well?
The agent handles multi-turn conversations naturally, users never need to repeat their budget, guest count, or preferences across messages. Pairing every stay recommendation with real-time weather context significantly improves decision quality. The calculator integration handles the most common travel question ("what's the split?") seamlessly, and the scoring system gives users a quick way to compare options at a glance.

### What are its current limitations?
SerpAPI returns Google search result snippets rather than direct Airbnb API data, so prices and availability are approximate and not live. Window Buffer Memory resets when the session ends, preferences are lost between separate conversations. Search quality also degrades for very small or niche locations with limited Google index coverage.

### What would you improve or extend with more time?
With more time I would integrate RapidAPI's Airbnb scraper for live listings, add a Google Calendar tool to check availability before suggesting travel dates, add a Gmail tool to email formatted recommendation summaries, switch to Postgres Memory for cross-session preference recall, and add a flight search tool so users get a full trip cost estimate including transport.

### How does memory improve the agent's usefulness?
Memory transforms this from a one-shot search tool into a genuine travel planning companion. Without memory, the user would re-state their budget, guest count, and preferences in every message. With Window Buffer Memory, the conversation flows naturally, "what about Kasol instead?" works perfectly because the agent already has full context. This mirrors how a real travel agent conversation works and dramatically reduces friction.

### Did the tool behave as expected? Were there any edge cases?
The search_stays tool worked well for popular destinations but returned sparse results for very small villages, solved by broadening the query to the nearest larger town. The get_weather tool via wttr.in was reliable and required zero configuration. The calculate tool worked perfectly for arithmetic. The biggest challenge was Gemini's strict tool parameter validation, it required explicit Placeholder Definitions using `{placeholder}` syntax rather than `$fromAI()` expressions, which took several iterations to resolve.

---

## Core Concept Questions

### 1. What is the difference between a simple LLM call and an LLM-powered agent?

A simple LLM call is a single stateless input-output operation, you send a prompt, the model returns a response, and the interaction ends. There is no ability to take action, access external data, or remember previous exchanges.

An LLM-powered agent is a reasoning loop. The LLM decides when to invoke external tools, examines the results, determines whether to call another tool or loop again, and only returns a final response when it has sufficient information. It can also maintain memory across multiple turns. In this workflow, Asha decides to check the weather, constructs a search query, parses the results, applies remembered preferences, scores the options, and formulates a structured response, all as part of handling a single user message.

### 2. What role does the system prompt play in shaping agent behaviour?

The system prompt is the agent's identity, operating rules, and output instructions combined. It defines who the agent is (Asha, a savvy Indian travel companion), which tools to use and when (always call get_weather first), how to format output (stay card structure with emoji and scores), what to never do (never fabricate listings), and how to behave conversationally (warm, friendly, always end with a follow-up question). Without a well-crafted system prompt the agent responds generically, skips tools when it shouldn't, produces inconsistent output, and ignores the scoring logic.

### 3. How does tool use extend what an LLM can do on its own?

LLMs have a training data cutoff and cannot access real-time information. Without tools, Asha could only suggest stays from training data which would be outdated and potentially hallucinated. Tools give the agent three capabilities it lacks on its own: real-time data access via search_stays and get_weather, precise computation via the calculator (LLMs can make arithmetic errors), and grounded responses that eliminate hallucination risk for factual claims by constraining the agent to only present tool-returned results.

### 4. What are the trade-offs between short-term (window buffer) and long-term (database) memory in agents?

Window Buffer Memory is zero-setup, free, and works instantly, but resets when the session ends and is capped at N messages. Database memory like Postgres or Redis persists across sessions so the agent can recall preferences from a user's previous conversations, but requires infrastructure setup, ongoing hosting costs, and introduces data privacy considerations. For a demo or single-session use case, window buffer is ideal. For a production app where users return repeatedly, database memory is essential to deliver genuine personalisation over time.

### 5. What is the purpose of the AI Agent node in n8n compared to a simple LLM Chain node?

The LLM Chain node is a linear pipeline, input enters, the LLM processes it once, output exits. It cannot call tools, make decisions, or iterate.

The AI Agent node implements a reasoning loop, it can decide whether to invoke a tool, examine the result, decide if it needs another tool call, and only return a final answer once it has enough information. It is the difference between a pipeline and an autonomous decision-making system. This workflow relies entirely on the agent node's ability to chain weather check, stay search, and cost calculation within a single user message turn.

### 6. What risks or failure modes did you observe when testing your agent?

The most significant failure was Gemini's strict tool parameter validation, it rejected all tools with empty property keys, requiring a specific `{placeholder}` syntax in URLs with matching Placeholder Definitions. Importing the workflow JSON also created duplicate AI Agent nodes, requiring a clean canvas and fresh import. Weather data from wttr.in occasionally overloaded the context with large JSON when the agent didn't summarise it. Niche location searches via SerpAPI returned irrelevant results. Overall the Gemini and n8n tool calling integration required significantly more configuration precision than OpenAI equivalents.

### 7. How would you evaluate whether your agent is performing well in a real production setting?

Key metrics would include: relevance accuracy (are recommended stays actually within stated budget and location, verified by spot-checking 10% of results), tool invocation correctness (is the right tool called at the right time, measured via execution logs), memory utilisation rate (are preferences from earlier turns applied to later responses, tracked by monitoring repeated preference re-statements), hallucination rate (are any listed properties fabricated. verified by cross-referencing against actual search results), task completion rate (what percentage of sessions result in at least one suitable stay found), conversation efficiency (average turns before a satisfactory result, fewer turns indicates better context retention), and error rate (frequency of tool failures or bad request responses in production logs).

---

## Setup Instructions

**Prerequisites:**
- n8n account at [n8n.io](https://n8n.io) or self-hosted
- Google AI Studio API key — free at [aistudio.google.com](https://aistudio.google.com)
- SerpAPI key — free tier at [serpapi.com](https://serpapi.com)

**Steps:**
1. Import `stayFinder.json` into n8n via top right menu → Import from file
2. Open the **Google Gemini Chat Model** node → select your Gemini credential
3. Open the **search_stays** node → replace `REPLACE_WITH_YOUR_SERPAPI_KEY` with your actual key
4. Activate the workflow using the toggle at the top right of the canvas
5. Click **Open Chat** and start searching

**Tool configuration quick reference:**

search_stays node — URL: `https://serpapi.com/search` | Query param `q`: `{query}` | Placeholder: `query` = "Search query for stays including location budget guests and preferences"

get_weather node — URL: `https://wttr.in/{location}?format=j1` | No query parameters | Placeholder: `location` = "Destination city name such as Manali Goa Coorg Kasol"

calculate node — No configuration needed, built-in n8n tool

---

## Repository Structure

```
pi-shaped-ai-in-sdlc/
└── day2-agents-workshop/
    ├── README.md                      ← This file
    ├── stayFinder.json       ← Final working n8n workflow
    └── screenshots/
        ├── workflow-canvas.png
        ├── test-1-weather-and-search.png
        ├── test-2-memory-working.png
        └── test-3-calculator.png
```

---
