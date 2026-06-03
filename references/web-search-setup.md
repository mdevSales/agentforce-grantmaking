# Web Search Setup

The **General Web Search Action** is a standard Salesforce asset — no custom code needed. Use it when the agent needs to search the internet for current information (federal grant programs, current eligibility rates, live government resources).

---

## When to Recommend Web Search

| Signal | Use web search |
|---|---|
| "Search for federal grant programs matching our applicants" | Yes |
| "Find current eligibility income limits from federal guidelines" | Yes |
| "Look up rates from HUD / USDA / SBA websites" | Yes |
| "Answer questions from our program guide" | No — use ADL instead |
| "Check the status of an application" | No — use TrackApplicationStatus Flow |

**Key rule:** Web search for external/live information. ADL for your org's own documents.

---

## Setup: Add General Web Search Action to Agent

### Step 1: Find the action in Setup
1. Setup → Einstein → Agent Actions (or: Setup → Agents → select your agent → Actions tab)
2. Search for "Search the Web" or "General Web Search"
3. The action API name is `sfdc_ai__GeneralWebSearch` (standard Salesforce asset)

### Step 2: Add to your agent topic
1. Select the topic where web search applies (typically the FAQ or Program Information topic)
2. Click "Add Action" → select "Search the Web"
3. Configure:
   - Max results: 3-5 (start with 3 — more results = larger context window consumption)
   - Safe search: Enabled (recommended for government/public-facing agents)
   - Domain restrictions: Optional — e.g., restrict to `.gov` domains for official sources

### Step 3: Configure agent instructions
In the topic instructions, tell the agent when to use web search vs. ADL:
> "Use the Search the Web action when the user asks about federal grant programs, current income limits from federal guidelines, or information from government websites. Use the Grant Programs Knowledge Base for questions about our own programs and policies."

---

## NGA Agent Script: Web Search Action

```
action SearchFederalGrants {
  type: StandardAction
  actionApiName: "sfdc_ai__GeneralWebSearch"
  inputs {
    query: "federal small business grants [state] current eligibility"
    maxResults: 3
  }
  outputs {
    results -> @webSearchResults
  }
}
```

---

## Limiting Scope to Government Sources

For a government agency agent, you may want to restrict web search to official government sources only:

**In the action configuration:**
- Domain filter: `site:*.gov OR site:*.mil OR site:grants.gov`

**In the agent instructions:**
> "When searching the web, prioritize results from government websites (.gov, .mil) and official grant databases like grants.gov. Clearly label information as coming from external sources."

---

## Pricing / Token Consumption Note

Web search results consume agent context window. Each result adds ~200-500 tokens. With max results = 5, that's up to 2,500 tokens per search call. For high-volume service agents, monitor usage.

If web search is only needed occasionally (e.g., "find new federal programs"), scope it to a specific topic rather than the entire agent.
