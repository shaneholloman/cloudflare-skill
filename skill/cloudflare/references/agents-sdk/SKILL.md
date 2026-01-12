# Cloudflare Agents SDK

## Overview
Cloudflare Agents SDK enables building and deploying AI-powered agents that autonomously perform tasks, communicate in real-time, call AI models, persist state, schedule tasks, and run asynchronous workflows. Agents run on Durable Objects, providing stateful, globally distributed execution.

## When to Use This Skill
- Building AI agents with persistent state and memory
- Creating real-time interactive applications with WebSockets
- Implementing long-running workflows that span minutes/hours
- Developing chat interfaces with AI models
- Building stateful microservices at the edge
- Implementing scheduled/recurring tasks with agents
- Creating agents that query databases and maintain state

## Quick Start

### Installation
```bash
# Create new agent project
npm create cloudflare@latest agents-starter -- --template=cloudflare/agents-starter

# Or add to existing project
npm install agents
```

### Basic Agent Definition
```typescript
import { Agent, AgentNamespace } from "agents";

interface Env {
  // Your bindings
  AI?: Ai;
  MyAgent?: DurableObjectNamespace<MyAgent>;
}

export class MyAgent extends Agent<Env> {
  // Agent methods here
}
```

## Core Concepts

### 1. Agent Lifecycle Methods

#### onStart()
Called when agent first initializes or restarts after hibernation.

```typescript
export class MyAgent extends Agent<Env> {
  onStart() {
    // Initialize database tables
    this.sql`
      CREATE TABLE IF NOT EXISTS users (
        id TEXT PRIMARY KEY,
        name TEXT,
        created_at INTEGER
      )
    `;
    
    // Set initial state
    if (!this.state.initialized) {
      this.setState({ initialized: true, messages: [] });
    }
  }
}
```

#### onRequest(request: Request)
Handle HTTP requests to the agent.

```typescript
async onRequest(request: Request): Promise<Response> {
  const url = new URL(request.url);
  
  if (url.pathname === "/messages") {
    const messages = this.sql<{ text: string }>`
      SELECT * FROM messages ORDER BY timestamp DESC LIMIT 10
    `;
    return Response.json(messages);
  }
  
  if (url.pathname === "/user" && request.method === "POST") {
    const { userId, name } = await request.json();
    this.sql`INSERT INTO users (id, name, created_at) 
             VALUES (${userId}, ${name}, ${Date.now()})`;
    return Response.json({ success: true });
  }
  
  return new Response("Not found", { status: 404 });
}
```

#### onConnect(connection: Connection, ctx: ConnectionContext)
Handle WebSocket connections.

```typescript
async onConnect(connection: Connection, ctx: ConnectionContext) {
  const url = new URL(ctx.request.url);
  const userId = url.searchParams.get("userId");
  
  // Authenticate connection
  if (!userId) {
    connection.close(4001, "Missing userId");
    return;
  }
  
  // Accept connection
  connection.accept();
  
  // Set connection-specific state
  connection.setState({ userId, connectedAt: Date.now() });
  
  // Send initial data
  connection.send(JSON.stringify({ 
    type: "connected", 
    state: this.state 
  }));
}
```

#### onMessage(connection: Connection, message: WSMessage)
Handle incoming WebSocket messages.

```typescript
async onMessage(connection: Connection, message: WSMessage) {
  const msg = JSON.parse(message as string);
  
  if (msg.type === "chat") {
    // Add message to state
    this.setState({
      ...this.state,
      messages: [...this.state.messages, {
        userId: connection.state.userId,
        text: msg.text,
        timestamp: Date.now()
      }]
    });
    
    // Broadcast to all connections
    this.connections.forEach(conn => {
      conn.send(JSON.stringify({ 
        type: "new_message", 
        message: msg.text 
      }));
    });
  }
}
```

#### onEmail(email: AgentEmail)
Handle incoming emails (requires email routing configuration).

```typescript
async onEmail(email: AgentEmail) {
  console.log(`Received email from: ${email.from}`);
  console.log(`Subject: ${email.headers.get("subject")}`);
  
  // Process email content
  const text = await email.text();
  
  // Store in database
  this.sql`
    INSERT INTO emails (from_addr, subject, body, received_at)
    VALUES (${email.from}, ${email.headers.get("subject")}, ${text}, ${Date.now()})
  `;
  
  // Update state
  this.setState({
    ...this.state,
    unreadEmails: this.state.unreadEmails + 1
  });
}
```

### 2. State Management

#### State Synchronization
State is automatically synced between server and all connected clients.

```typescript
interface ChatState {
  messages: Array<{ sender: string; text: string; timestamp: number }>;
  participants: string[];
  settings: {
    allowAnonymous: boolean;
    maxHistoryLength: number;
  };
}

export class ChatAgent extends Agent<Env, ChatState> {
  initialState: ChatState = {
    messages: [],
    participants: [],
    settings: {
      allowAnonymous: false,
      maxHistoryLength: 100
    }
  };
  
  async addMessage(sender: string, text: string) {
    // Update state - automatically syncs to all clients
    this.setState({
      ...this.state,
      messages: [
        ...this.state.messages,
        { sender, text, timestamp: Date.now() }
      ].slice(-this.state.settings.maxHistoryLength)
    });
  }
  
  // Called when state updates from any source
  onStateUpdate(state: ChatState, source: "server" | Connection) {
    console.log(`State updated by ${source}`);
    
    // React to state changes
    if (state.messages.length > 0) {
      const lastMessage = state.messages[state.messages.length - 1];
      if (lastMessage.text.includes("@everyone")) {
        this.notifyAllParticipants(lastMessage);
      }
    }
  }
}
```

### 3. SQL Database Access

Every agent has a built-in SQLite database accessible via `this.sql`.

```typescript
export class UserAgent extends Agent<Env> {
  onStart() {
    // Create tables
    this.sql`
      CREATE TABLE IF NOT EXISTS users (
        id TEXT PRIMARY KEY,
        name TEXT NOT NULL,
        email TEXT UNIQUE,
        created_at INTEGER
      )
    `;
    
    this.sql`
      CREATE TABLE IF NOT EXISTS sessions (
        id TEXT PRIMARY KEY,
        user_id TEXT,
        expires_at INTEGER,
        FOREIGN KEY(user_id) REFERENCES users(id)
      )
    `;
  }
  
  async createUser(id: string, name: string, email: string) {
    // Parameterized queries prevent SQL injection
    this.sql`
      INSERT INTO users (id, name, email, created_at)
      VALUES (${id}, ${name}, ${email}, ${Date.now()})
    `;
  }
  
  async getUser(userId: string) {
    // Type-safe queries
    const [user] = this.sql<{ id: string; name: string; email: string }>`
      SELECT * FROM users WHERE id = ${userId}
    `;
    return user;
  }
  
  async getUserSessions(userId: string) {
    const sessions = this.sql<{ id: string; expires_at: number }>`
      SELECT * FROM sessions 
      WHERE user_id = ${userId} AND expires_at > ${Date.now()}
    `;
    return sessions;
  }
}
```

### 4. Task Scheduling

Schedule methods for one-time, delayed, or recurring execution.

```typescript
export class SchedulerAgent extends Agent<Env> {
  async scheduleExamples() {
    // Schedule at specific time
    await this.schedule(
      new Date("2026-12-25T00:00:00Z"), 
      "sendGreeting", 
      { message: "Merry Christmas!" }
    );
    
    // Schedule with delay (seconds)
    await this.schedule(60, "checkStatus", { check: "health" });
    
    // Recurring with cron expression (daily at midnight)
    await this.schedule("0 0 * * *", "dailyCleanup", { type: "logs" });
    
    // Every 5 minutes
    await this.schedule("*/5 * * * *", "syncData", {});
  }
  
  async sendGreeting(payload: { message: string }) {
    console.log(payload.message);
  }
  
  async checkStatus(payload: { check: string }) {
    console.log("Running check:", payload.check);
  }
  
  async dailyCleanup(payload: { type: string }) {
    // Clean up old records
    this.sql`DELETE FROM logs WHERE created_at < ${Date.now() - 86400000}`;
  }
  
  async syncData(payload: {}) {
    // Sync data from external source
  }
  
  // Manage schedules
  async listSchedules() {
    const schedules = await this.getSchedules();
    return schedules;
  }
  
  async cancelSchedule(scheduleId: string) {
    await this.cancelSchedule(scheduleId);
  }
}
```

### 5. AI Model Integration

#### Using Workers AI
```typescript
interface Env {
  AI: Ai;
}

export class AIAgent extends Agent<Env> {
  async generateResponse(prompt: string) {
    const response = await this.env.AI.run(
      "@cf/meta/llama-3.1-8b-instruct",
      { prompt }
    );
    return response;
  }
  
  // With AI Gateway for caching/routing
  async generateWithGateway(prompt: string) {
    const response = await this.env.AI.run(
      "@cf/deepseek-ai/deepseek-r1-distill-qwen-32b",
      { prompt },
      {
        gateway: {
          id: "my-gateway-id",
          skipCache: false,
          cacheTtl: 3600
        }
      }
    );
    return response;
  }
}
```

#### Streaming AI Responses via WebSocket
```typescript
import { OpenAI } from "openai";

export class StreamingAgent extends Agent<Env> {
  async onMessage(connection: Connection, message: WSMessage) {
    const msg = JSON.parse(message as string);
    
    if (msg.type === "prompt") {
      await this.streamAIResponse(connection, msg.prompt);
    }
  }
  
  async streamAIResponse(connection: Connection, userPrompt: string) {
    const client = new OpenAI({
      apiKey: this.env.OPENAI_API_KEY
    });
    
    try {
      const stream = await client.chat.completions.create({
        model: "gpt-4",
        messages: [{ role: "user", content: userPrompt }],
        stream: true
      });
      
      // Stream chunks back via WebSocket
      for await (const chunk of stream) {
        const content = chunk.choices[0]?.delta?.content || "";
        if (content) {
          connection.send(JSON.stringify({ 
            type: "chunk", 
            content 
          }));
        }
      }
      
      connection.send(JSON.stringify({ type: "done" }));
    } catch (error) {
      connection.send(JSON.stringify({ 
        type: "error", 
        error: String(error) 
      }));
    }
  }
}
```

### 6. AI Tools Definition

Define tools for AI agents with automatic/manual confirmation.

```typescript
import { tool } from "ai";
import { z } from "zod";

// Server-side tool requiring confirmation
const getWeatherTool = tool({
  description: "Get current weather for a city",
  inputSchema: z.object({
    city: z.string().describe("City name")
  })
  // No execute function - requires human approval
});

// Auto-executing tool
const getLocalNewsTool = tool({
  description: "Get local news for a location",
  inputSchema: z.object({ 
    location: z.string() 
  }),
  execute: async ({ location }) => {
    console.log(`Getting news for ${location}`);
    await new Promise(res => setTimeout(res, 2000));
    return `Breaking: ${location} kittens found drinking tea`;
  }
});

export const tools = {
  getWeather: getWeatherTool,
  getLocalNews: getLocalNewsTool
};
```

### 7. Connection Management

```typescript
export class MultiUserAgent extends Agent<Env> {
  async onConnect(connection: Connection, ctx: ConnectionContext) {
    connection.accept();
    
    // Set connection state
    connection.setState({ 
      joinedAt: Date.now(),
      userId: ctx.request.headers.get("X-User-ID")
    });
    
    // Notify others
    this.broadcast({
      type: "user_joined",
      userId: connection.state.userId
    });
  }
  
  broadcast(message: any) {
    const msg = JSON.stringify(message);
    this.connections.forEach(conn => conn.send(msg));
  }
  
  sendToUser(userId: string, message: any) {
    const connection = this.connections.find(
      conn => conn.state.userId === userId
    );
    if (connection) {
      connection.send(JSON.stringify(message));
    }
  }
  
  getActiveUsers() {
    return this.connections.map(conn => ({
      id: conn.state.userId,
      joinedAt: conn.state.joinedAt
    }));
  }
}
```

## Configuration

### wrangler.toml / wrangler.jsonc

```toml
name = "my-agents-app"

[[durable_objects.bindings]]
name = "MyAgent"
class_name = "MyAgent"

[[migrations]]
tag = "v1"
new_sqlite_classes = ["MyAgent"]

# Bindings
[ai]
binding = "AI"

[[kv_namespaces]]
binding = "KV"
id = "your-kv-id"

[[d1_databases]]
binding = "DB"
database_name = "my-database"
database_id = "your-db-id"
```

```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "name": "my-agents-app",
  "durable_objects": {
    "bindings": [
      {
        "name": "MyAgent",
        "class_name": "MyAgent"
      }
    ]
  },
  "migrations": [
    {
      "tag": "v1",
      "new_sqlite_classes": ["MyAgent"]
    }
  ],
  "ai": {
    "binding": "AI"
  }
}
```

### Worker Entry Point

```typescript
import { MyAgent } from "./agent";

export { MyAgent };

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Get agent ID from URL or generate
    const url = new URL(request.url);
    const agentId = url.searchParams.get("id") || "default";
    
    // Get agent stub
    const id = env.MyAgent.idFromName(agentId);
    const stub = env.MyAgent.get(id);
    
    // Forward request to agent
    return stub.fetch(request);
  }
};
```

## Common Patterns

### Chat Agent with History
```typescript
interface ChatState {
  messages: Array<{ role: string; content: string }>;
}

export class ChatAgent extends Agent<Env, ChatState> {
  initialState: ChatState = { messages: [] };
  
  async onMessage(connection: Connection, message: WSMessage) {
    const { prompt } = JSON.parse(message as string);
    
    // Add user message
    this.setState({
      messages: [...this.state.messages, { 
        role: "user", 
        content: prompt 
      }]
    });
    
    // Generate AI response
    const response = await this.env.AI.run(
      "@cf/meta/llama-3.1-8b-instruct",
      { messages: this.state.messages }
    );
    
    // Add assistant message
    this.setState({
      messages: [...this.state.messages, {
        role: "assistant",
        content: response.response
      }]
    });
    
    connection.send(JSON.stringify({ 
      type: "response", 
      content: response.response 
    }));
  }
}
```

### Rate Limiting Agent
```typescript
export class RateLimitedAgent extends Agent<Env> {
  onStart() {
    this.sql`
      CREATE TABLE IF NOT EXISTS rate_limits (
        user_id TEXT PRIMARY KEY,
        requests INTEGER,
        window_start INTEGER
      )
    `;
  }
  
  async checkRateLimit(userId: string, maxRequests = 10): Promise<boolean> {
    const windowMs = 60000; // 1 minute
    const now = Date.now();
    
    const [limit] = this.sql<{ requests: number; window_start: number }>`
      SELECT requests, window_start FROM rate_limits WHERE user_id = ${userId}
    `;
    
    if (!limit || now - limit.window_start > windowMs) {
      // New window
      this.sql`
        INSERT OR REPLACE INTO rate_limits (user_id, requests, window_start)
        VALUES (${userId}, 1, ${now})
      `;
      return true;
    }
    
    if (limit.requests >= maxRequests) {
      return false; // Rate limit exceeded
    }
    
    // Increment counter
    this.sql`
      UPDATE rate_limits 
      SET requests = requests + 1 
      WHERE user_id = ${userId}
    `;
    return true;
  }
  
  async onRequest(request: Request): Promise<Response> {
    const userId = request.headers.get("X-User-ID") || "anonymous";
    
    if (!(await this.checkRateLimit(userId))) {
      return new Response("Rate limit exceeded", { status: 429 });
    }
    
    // Process request
    return Response.json({ success: true });
  }
}
```

### Background Job Processor
```typescript
export class JobAgent extends Agent<Env> {
  onStart() {
    this.sql`
      CREATE TABLE IF NOT EXISTS jobs (
        id TEXT PRIMARY KEY,
        status TEXT,
        payload TEXT,
        created_at INTEGER,
        completed_at INTEGER
      )
    `;
    
    // Schedule job processor every minute
    this.schedule("* * * * *", "processJobs", {});
  }
  
  async addJob(jobId: string, payload: any) {
    this.sql`
      INSERT INTO jobs (id, status, payload, created_at)
      VALUES (${jobId}, 'pending', ${JSON.stringify(payload)}, ${Date.now()})
    `;
  }
  
  async processJobs() {
    const jobs = this.sql<{ id: string; payload: string }>`
      SELECT id, payload FROM jobs WHERE status = 'pending' LIMIT 10
    `;
    
    for (const job of jobs) {
      try {
        await this.processJob(JSON.parse(job.payload));
        
        this.sql`
          UPDATE jobs 
          SET status = 'completed', completed_at = ${Date.now()}
          WHERE id = ${job.id}
        `;
      } catch (error) {
        this.sql`
          UPDATE jobs SET status = 'failed' WHERE id = ${job.id}
        `;
      }
    }
  }
  
  async processJob(payload: any) {
    // Job processing logic
    console.log("Processing job:", payload);
  }
}
```

## Best Practices

### 1. State Design
- Keep state minimal and serializable
- Use SQL for complex queries and large datasets
- Design state for real-time synchronization needs
- Consider state size limits (128KB for Durable Objects)

### 2. SQL Usage
- Create tables in `onStart()`
- Use parameterized queries to prevent SQL injection
- Add indexes for frequently queried columns
- Clean up old data with scheduled tasks

### 3. WebSocket Handling
- Always accept connections explicitly with `connection.accept()`
- Handle connection state separately from agent state
- Implement reconnection logic on client side
- Use connection.state for per-connection data

### 4. Error Handling
- Wrap agent methods in try-catch blocks
- Return appropriate HTTP status codes
- Send error messages via WebSocket connections
- Log errors for debugging

### 5. Performance
- Minimize state updates (they sync to all clients)
- Batch SQL operations when possible
- Use SQL instead of state for large datasets
- Consider connection count limits

### 6. Scheduling
- Use cron expressions for recurring tasks
- Cancel schedules when no longer needed
- Handle schedule failures gracefully
- Avoid overlapping scheduled executions

## Deployment

### Development
```bash
npx wrangler dev
```

### Production
```bash
npx wrangler deploy
```

### Testing Locally
```bash
# Run with local mode
npx wrangler dev --local

# Test WebSocket connection
wscat -c ws://localhost:8787/websocket

# Test HTTP endpoint
curl http://localhost:8787/api/health
```

## Debugging

### Logging
```typescript
export class DebugAgent extends Agent<Env> {
  async onMessage(connection: Connection, message: WSMessage) {
    console.log("Received:", message);
    console.log("Connection state:", connection.state);
    console.log("Agent state:", this.state);
    console.log("Active connections:", this.connections.length);
  }
}
```

### Observability
```typescript
export class ObservableAgent extends Agent<Env> {
  // Disable default observability
  observability = undefined;
  
  // Or configure custom observability
  observability = {
    logLevel: "info",
    sampleRate: 1.0
  };
}
```

## Integration with Other Cloudflare Products

### Workers AI
```typescript
const response = await this.env.AI.run(model, input);
```

### AI Gateway
```typescript
const response = await this.env.AI.run(model, input, {
  gateway: { id: "gateway-id", skipCache: false, cacheTtl: 3600 }
});
```

### Vectorize
```typescript
const results = await this.env.VECTORIZE.query(embedding, { topK: 5 });
```

### D1 Database
```typescript
const results = await this.env.DB.prepare(
  "SELECT * FROM users WHERE id = ?"
).bind(userId).all();
```

### KV Storage
```typescript
await this.env.KV.put("key", "value");
const value = await this.env.KV.get("key");
```

### Workflows
Agents can integrate with Cloudflare Workflows for long-running, reliable task execution.

## Resources

- [Official Documentation](https://developers.cloudflare.com/agents/)
- [GitHub Repository](https://github.com/cloudflare/agents)
- [Examples](https://github.com/cloudflare/agents/tree/main/examples)
- [API Reference](https://developers.cloudflare.com/agents/api-reference/)
- [Durable Objects](https://developers.cloudflare.com/durable-objects/)
- [Workers AI](https://developers.cloudflare.com/workers-ai/)

## Key Differences from Standard Durable Objects

1. **Built-in State Management**: `this.state` with automatic sync
2. **SQL Access**: `this.sql` template tag for database queries
3. **Scheduling**: `this.schedule()` for recurring/delayed tasks
4. **Simplified WebSockets**: Connection management via `onConnect`/`onMessage`
5. **Lifecycle Hooks**: `onStart`, `onRequest`, `onConnect`, `onMessage`, `onEmail`
6. **Built-in Hibernation**: Agents automatically hibernate when idle

## Common Pitfalls

1. **Not creating tables in onStart()**: Always initialize DB schema
2. **Forgetting connection.accept()**: WebSockets won't connect without it
3. **Large state objects**: Use SQL for large datasets, state for UI sync
4. **Blocking operations**: Use async operations, don't block the event loop
5. **Missing error handling**: Always handle errors in lifecycle methods
6. **Hardcoded agent IDs**: Use dynamic IDs based on user/session
7. **Not cleaning up schedules**: Cancel schedules when no longer needed

## Advanced Topics

### Hibernation
Agents automatically hibernate when idle to save resources. State and SQL persist across hibernation.

```typescript
export class MyAgent extends Agent<Env> {
  static options = { 
    hibernate: true // Enable hibernation (default)
  };
}
```

### Cross-Agent Communication
```typescript
// Get another agent
const otherId = this.env.OtherAgent.idFromName("other-agent");
const otherStub = this.env.OtherAgent.get(otherId);

// Call it
const response = await otherStub.fetch(
  new Request("https://fake-host/data")
);
```

### Agent-to-Agent (A2A) Protocol
Use A2A for structured agent communication patterns.

```typescript
import { A2AHonoApp } from "agents/a2a";

export class A2AAgent extends Agent<Env> {
  async onRequest(request: Request): Promise<Response> {
    const appBuilder = new A2AHonoApp(this);
    const app = appBuilder.setupRoutes(new Hono());
    return app.fetch(request);
  }
}
```
