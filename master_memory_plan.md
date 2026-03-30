# Master Long-Term Memory & RAG Plan

## 1. The Database Selection: Consolidating into Supabase
Originally, one might consider maintaining two databases: a spatial DB for Overture Maps POIs and Qdrant for RAG Vector Memory. However, maintaining two disparate database clusters doubles your DevOps overhead, doubles your monthly server bills, and makes relational cross-queries impossible.

**The Recommendation: Unified Supabase (`pgvector` + `PostGIS`)**
- **Why it wins:** Supabase (built on PostgreSQL) allows you to store your Overture Maps POIs in one table, and your RAG User Vectors in another table. When the AI searches memory, it can execute a single SQL query that dynamically joins the user's vector preferences against the exact physical POIs around them. While Qdrant is marginally faster for pure vectors at mega-scale, the introduction of the new `pgvectorscale` extension makes Supabase blisteringly fast for RAG, meaning you get the best of both worlds with zero added infrastructure.

## 2. The Data Schema (How to Store Preferences)
Because the iOS app streams binary audio directly to ElevenLabs via LiveKit WebRTC, the iPhone never possesses the raw text transcript of the conversation. Therefore, the iOS app physically cannot upload transcripts.

Instead, every time a LiveKit voice session concludes:
1. The iOS app sends ONLY the `conversation_id` and `user_id` to a **Supabase Edge Function** (`/extract-memory`).
2. The Edge Function uses a hidden backend Admin API Key to hit the ElevenLabs REST API securely, downloading the full text transcript server-side.
3. The Edge Function runs this transcript through an LLM (`gpt-4o-mini`) to extract "Memory Chunks" and automatically embeds them into the database. 

Each chunk is inserted into your Supabase `user_memories` table as a Vector with the following rich metadata schema:

```json
{
  "user_id": "uuid",
  "memory_text": "User is deathly allergic to peanuts and dislikes crowded cafes.",
  "category": "dietary_preference",
  "confidence": 0.95,  // Scored by the extracting LLM (0.0 to 1.0)
  "timestamp": 1711756800, // For recency decay
  "vector": [0.012, -0.044, ...] // Embedded via text-embedding-3-small
}
```

## 3. Context Window Management (The 3-Tier Architecture)
To ensure the LLM context window doesn't explode (saving API costs and preventing hallucination), memory must be tiered:

1. **Short-Term (In-Context RAM):** The active messages managed automatically by the **LiveKit WebRTC** session.
2. **Mid-Term (Spatial/Session Context):** Mapbox POIs injected dynamically into the system prompt upon Geofence triggers.
3. **Long-Term (Supabase RAG):** Historical profile data. *This is never loaded entirely.*

## 4. The Implementation: Retrieval via Tool Calling (Fixing the RAG Loop)
In older designs, the iOS app would intercept text and query the database. However, since the iOS app only streams binary audio natively to ElevenLabs, the iPhone cannot unilaterally decide when to query Supabase for memories because the iPhone physically doesn't know what the user just said!

**The Solution:** Long-Term Memory is exposed to the AI natively as an ElevenLabs Client Tool.

### The Algorithm:
1. User asks: *"I'm looking for a place to eat nearby, what would I like?"* over the microphone.
2. **The Brain Decides:** The ElevenLabs LLM processes the audio in the cloud and realizes it needs to know the user's historical food preferences.
3. **Tool Invocation:** The AI interrupts its standard flow and emits a tool call down the WebRTC socket: `search_user_memory(query: "food preferences")`.
4. **The iOS Mediator:** `VoiceStateManager` receives the tool call JSON, pauses the mic, and sends the query text over HTTP directly to the Supabase Edge Function (`/rag-search`).
5. **Server-Side Heuristic Scoring:** The Edge Function securely hits the OpenAI API to convert the query string into a vector, and queries `pgvector`. It calculates a final priority score for each memory:
   `Final Score = (Cosine_Similarity * 0.5) + (Confidence * 0.3) + (Recency_Decay_Multiplier * 0.2)`
   *Note: Recency decay means a memory from yesterday gets a `1.0` multiplier, while a memory from 3 years ago gets a `0.2` multiplier, ensuring the user's current tastes beat old tastes.*
6. **Prompt Delivery:** Supabase returns the Top 5 highest-scoring memory snippets as JSON to the iPhone.
7. **The WebRTC Return:** The iPhone executes `sendToolResult()` to push the retrieved memories back up the socket to ElevenLabs.
8. **The Output:** The AI immediately synthesizes: *"Since you're deathly allergic to peanuts, let's avoid the Thai place and check out this French cafe instead."*

This guarantees infinite memory depth, zero context-window bloat, and flawless WebRTC compatibility.
