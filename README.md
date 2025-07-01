# n8n AI-Powered Chatbot Workflow

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Workflow Components](#workflow-components)
3. [Supabase Schema](#supabase-schema)
4. [n8n Workflow Configuration](#n8n-workflow-configuration)
5. [Frontend Integration & Customization](#frontend-integration--customization)
6. [Logo & Theme Overrides](#logo--theme-overrides)
7. [Step-by-Step Setup](#step-by-step-setup)

---

## Architecture Overview
This workflow stitches together multiple systems for a seamless chat experience:

<img src="https://primary-production-2548.up.railway.app/wp-content/uploads/2025/07/vector-chat-bot.png" alt="Chatbot Vector" width="600"/>


```mermaid
flowchart TD
  subgraph Frontend
    A[User Chat Widget]
  end
  A -->|POST /webhook| B[n8n: Webhook Trigger]
  B --> C{Fetch Memory?}
  C -->|Yes| D[n8n: Supabase Query]
  C -->|No| E[Skip]
  D --> F[n8n: Google Gemini Embed]
  F --> G[n8n: Supabase Upsert Embedding]
  G --> H[n8n: Gemini Chat]
  H --> I[n8n: Supabase Save Conversation]
  I -->|Return| J[Frontend: Render AI Reply]
````

* **Frontend**: Hosts a chat widget that POSTs incoming messages.
* **n8n**: Orchestrates triggers, embedding, retrieval, AI processing, and persistence.
* **Google Gemini**: Generates both embeddings (`text-embedding-004`) and chat responses (`gemini-2.5-flash-preview-05-20`).
* **Supabase**: Stores vectors in `documents` & `embeddings` tables, and logs conversations in `conversation_memory`.

## Workflow Components

1. **Webhook Trigger** (`ChatIncoming`): HTTP POST captures `{ userId, message, timestamp }`.
2. **Memory Retrieval** (`GetMemory`): SQL query against `conversation_memory` filtered by `userId` and recent timestamps.
3. **Text Embedding** (`CreateEmbedding`): Calls Gemini’s embedding endpoint.
4. **Vector Upsert** (`UpsertDocument`): Inserts or updates vector in Supabase + pgvector extension.
5. **AI Agent** (`GeminiChat`): Sends prompt with context from memory and retrieval.
6. **Save Memory** (`SaveConversation`): Persists both user message and AI reply.
7. **Response Webhook** (`Respond`): Returns `{ reply }` to frontend.

## Supabase Schema

```sql
-- documents table for static knowledge
CREATE TABLE documents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title TEXT NOT NULL,
  content TEXT NOT NULL
);

-- embeddings table for vectors
CREATE TABLE embeddings (
  id UUID PRIMARY KEY,
  vector VECTOR(1536),
  document_id UUID REFERENCES documents(id)
);

-- conversation memory log
CREATE TABLE conversation_memory (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id TEXT NOT NULL,
  role TEXT CHECK(role IN ('user','assistant')) NOT NULL,
  message TEXT NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);
```

## n8n Workflow Configuration

Import this JSON snippet into n8n (`Settings → Import → Workflow`):

```json
{
  "name": "AI Chatbot",
  "nodes": [
    {
      "parameters": {"httpMethod": "POST","path": "chat/webhook"},
      "name": "ChatIncoming","type": "n8n-nodes-base.webhook"
    },
    {
      "parameters": {"operation": "executeQuery","query": "SELECT message, role FROM conversation_memory WHERE user_id = :userId ORDER BY created_at DESC LIMIT 10"},
      "name": "GetMemory","type": "n8n-nodes-base.postgres","credentials": {"postgres": "Supabase DB"}
    },
    {
      "parameters": {"model": "text-embedding-004","input": "={{$json[\"message\"]}}"},
      "name": "CreateEmbedding","type": "n8n-nodes-base.openai"
    },
    {
      "parameters": {"operation": "UPSERT","table": "embeddings","columns": [
          {"name": "id","value": "={{$node[\\"ChatIncoming\\"].json.userId + '-' + $node[\\"ChatIncoming\\"].json.timestamp}}"},
          {"name": "vector","value": "={{$json.choices[0].embedding}}"},
          {"name": "document_id","value": "={{$node[\\"ChatIncoming\\"].json.userId}}"}
      ]},
      "name": "UpsertDocument","type": "n8n-nodes-base.supabase"
    },
    {
      "parameters": {"model": "gemini-2.5-flash-preview-05-20","prompt": "={{`Context:\n${$node.GetMemory.json.map(m=>m.role+': '+m.message).join('\n')}\nUser: ${$node.ChatIncoming.json.message}\nAssistant:`}}","temperature": 0.7},
      "name": "GeminiChat","type": "n8n-nodes-base.openai"
    },
    {
      "parameters": {"operation": "insert","table": "conversation_memory","columns": [
          {"name": "user_id","value": "={{$json.userId}}"},
          {"name": "role","value": "assistant"},
          {"name": "message","value": "={{$json.choices[0].message.content}}"}
      ]},
      "name": "SaveConversation","type": "n8n-nodes-base.postgres","credentials": {"postgres": "Supabase DB"}
    }
  ],
  "connections": {
    "ChatIncoming": {"main": [[{"node":"GetMemory"}]]},
    "GetMemory": {"main": [[{"node":"CreateEmbedding"}]]},
    "CreateEmbedding": {"main": [[{"node":"UpsertDocument"}]]},
    "UpsertDocument": {"main": [[{"node":"GeminiChat"}]]},
    "GeminiChat": {"main": [[{"node":"SaveConversation"}]]}
  }
}
```

## Frontend Integration & Customization

### Full HTML + JS snippet

```html
<link rel="stylesheet" href="https://unpkg.com/@n8n/chat/dist/styles.css" />
<div id="chat-container" style="width:100%;max-width:400px;height:600px;"></div>
<script src="https://unpkg.com/@n8n/chat/dist/index.min.js"></script>
<script>
  const chat = new n8nChat.ChatWidget({
    webhookUrl: 'https://your-n8n-domain/webhook/chat/webhook',
    userId: 'user-1234',
    styles: {'--primary-color': '#4A90E2','--font-family': 'Arial, sans-serif'},
    logoUrl: 'https://your.domain/logo.png'
  });
  chat.mount('#chat-container');
</script>
```

#### Elementor

1. Drag an **HTML** widget onto your page.
2. Paste the snippet above.
3. Override CSS variables in a `<style>` tag.

#### Framer

1. Add an **Embed** component with the HTML/JS snippet.
2. Allow external scripts (`unpkg.com`).

#### Webflow / Wix / Others

Embed via Custom Code; ensure `<head>` includes `<link>` and `<body>` includes `<script>`.

## Logo & Theme Overrides

```css
:root {
  --primary-color: #4A90E2;
  --secondary-color: #FFD700;
  --chat-border-radius: 12px;
  --chat-font-size: 14px;
}
```

## Step-by-Step Setup

1. Import workflow JSON into n8n.
2. Configure Supabase & Google Palm credentials.
3. Create tables & enable `pgvector`.
4. Deploy n8n with public webhook.
5. Add chat widget via CDN or npm.
6. Customize CSS variables & logo.
7. Test chat flow end-to-end.

---

Contact **[basalelr@gmail.com](mailto:basalelr@gmail.com)** for integration support.