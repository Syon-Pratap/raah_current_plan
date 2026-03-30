# Master Hyperlocal Web Search Plan

## 1. The Real-Time Latency Constraint
Currently, the app uses `WebSearchService.swift` to ping the Brave Search API directly from the iPhone. 
- **The Problem:** Brave only returns short snippets (like a Google search page). It doesn't actually click the link and read the specific Reddit thread. 
- **The Trap:** If you try to fix this by searching on Brave, and then passing the URL to a heavy web scraper (like an iOS SwiftSoup integration) to read the full page locally, the API roundtrip will take **5 to 8 seconds**. In a live voice conversation, an 8-second pause forces the user to think the app crashed.

## 2. The Objective Best Option: Tavily API
Based on current AI engineering infrastructure, **Tavily** is objectively the best solution for Real-Time LLM Voice Agents. 

- **Why it wins:** Tavily was explicitly built for agentic AI. In a **single API call**, it executes a semantic search across the internet, actively visits the top 5 links (including Reddit), extracts the core text, strips out all HTML ads and sidebars, and returns pure, condensed Markdown.
- **Latency Guarantee:** It executes this entire discovery-and-extraction loop in **~1.5 seconds**, which is fast enough to keep a voice conversation flowing naturally.
- **Reddit Specifics:** Tavily strongly indexes forum discussions. For highly targeted Reddit threads, you can append `.json` to any Reddit URL to instantly extract the exact comments without needing a headless browser.

---

## 3. The Implementation Blueprint (How to Build It)

You absolutely must **remove `WebSearchService.swift` from the iOS App.** Making external search API calls directly from the user's phone exposes your private API keys to hackers and wastes mobile battery. 

Instead, we shift this to the Cloud Engine via **Client WebRTC Tool Calling**.

### Step 1: Create the Supabase Edge Function
Create a Supabase Edge Function called `/hyperlocal-search`.
- This function receives a query from the iOS App (e.g., `"Reddit opinions on the best coffee near the Eiffel Tower"`).
- The Edge Function securely triggers the Tavily API using your private server key hidden from the public.
- Tavily returns the compressed Markdown of the forum discussions.
- The Edge Function passes this Markdown straight back to the iOS App over standard HTTP.

### Step 2: Configure the ElevenLabs Tool Call
In the ElevenLabs Developer Console, you will register a **Client Tool** called `search_hyperlocal_internet`. 

You must define the exact JSON Schema so the AI knows when to trigger it:
```json
{
  "name": "search_hyperlocal_internet",
  "description": "Use this tool ONLY if the user asks for highly specific, recent, or native opinions about a location. It searches Reddit and local forums.",
  "parameters": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "The search term, explicitly including the word 'Reddit' or 'local forums' to ensure hyperlocal context."
      }
    },
    "required": ["query"]
  }
}
```

### Step 3: The Seamless Handoff UX (Solving the Routing Loop)
*ElevenLabs does not trigger the Supabase function directly via the cloud. It is physically impossible for the client API key to trigger server webhooks mid-stream. It routes through the local iOS iPhone via WebRTC.*

When the user asks, *"Are there any secret lookout spots around here?"*
1. **The Brain Decides:** The ElevenLabs LLM realizes it doesn't know the answer. 
2. It interrupts the voice stream to natively send a `tool_call` JSON blob down the LiveKit WebRTC socket to the iPhone.
3. **The iOS Mediator:** `VoiceStateManager` receives the payload. It pauses the user's microphone to prevent echo, and fires an HTTP Request up to the `/hyperlocal-search` Supabase Edge function. 
4. **The UX Trick:** While the iPhone awaits Supabase to scrape Tavily, the ElevenLabs agent can be configured to say a filler phrase over WebRTC: *"Let me scan the local forums for you... ah, okay."*
5. Supabase returns the Markdown JSON to the iPhone.
6. The iPhone executes `sendToolResult()` to push the Markdown back UP the WebRTC socket to ElevenLabs.
7. **The Output:** The AI immediately processes the text and resumes the conversation natively: *"Locals on Reddit are saying there's a hidden staircase right behind the museum... "*

---
**Summary:** By abandoning direct iOS API calls and combining **Tavily's Extraction** with **Supabase Edge Functions** and **WebRTC Client Tool Routing**, you achieve true hyperlocal intelligence with precisely zero exposed API keys and flawless low-latency execution.
