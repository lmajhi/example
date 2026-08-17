# AI Chat Session Management — Initial Architecture Design

## 1. Purpose

This document defines the initial architecture for a ChatGPT-like AI chat/session management system.

The initial deployment constraint is:

- **Windows VMs**
- No cloud-native infrastructure initially
- No Kubernetes initially
- No requirement for microservices initially
- Architecture should remain modular enough to migrate to cloud infrastructure later

The primary goal is to build a clean foundation for:

- Persistent chat conversations
- Streaming AI responses
- Agent execution
- Tool/MCP execution
- Conversation history
- Context management
- Attachments
- Background processing
- Future semantic search and memory

---

# 2. Core Architectural Principle

The most important design decision is:

> **A conversation is durable user-facing history. LLM context is a derived runtime representation.**

They should not be treated as the same thing.

```text
Conversation
     |
     | canonical history
     v
Context Manager
     |
     +-- Recent messages
     +-- Conversation summary
     +-- Relevant historical messages
     +-- User memory
     +-- System instructions
     +-- Tools
     |
     v
LLM / Agent Context
```

The canonical conversation history must remain independently recoverable.

---

# 3. Initial High-Level Architecture

```text
                         +----------------------+
                         |      React Client    |
                         |   Web / Mobile UI    |
                         +----------+-----------+
                                    |
                                  HTTPS
                                    |
                                    v
                         +----------------------+
                         |     IIS / Nginx      |
                         |   Reverse Proxy      |
                         +----------+-----------+
                                    |
                                    v
                  +-------------------------------------+
                  |           Chat Backend              |
                  |                                     |
                  | Conversation API                    |
                  | Message API                         |
                  | Run API                             |
                  | Attachment API                      |
                  | SSE Streaming                       |
                  +------------+---------------+--------+
                               |               |
                               |               |
                    +----------v-----+    +----v-------+
                    |   PostgreSQL   |    |   Redis    |
                    |                |    |            |
                    | Conversations |    | Cache      |
                    | Messages      |    | Locks      |
                    | Runs          |    | Queue      |
                    | Tool Calls    |    | Rate Limit |
                    +----------------+    +-----+------+
                                                |
                                                v
                                        +---------------+
                                        |    Worker     |
                                        |               |
                                        | Title         |
                                        | Summary       |
                                        | Embeddings    |
                                        +---------------+

                  Chat Backend
                       |
                       v
              +-------------------+
              |   Agent Runtime   |
              |      Python       |
              |                   |
              | Context Manager   |
              | Orchestration     |
              | MCP               |
              | Tools             |
              | LLM Calls         |
              +---------+---------+
                        |
               +--------+--------+
               |                 |
               v                 v
          LLM Providers      APIs / MCP
```

---

# 4. Initial Deployable Components

The initial system should have approximately four logical deployable components.

## 4.1 Chat Backend

Recommended:

- Node.js
- TypeScript
- Fastify or Express

Responsibilities:

- Authentication/authorization integration
- Conversation management
- Message persistence
- Run lifecycle
- Attachment metadata
- SSE streaming
- Idempotency
- API validation
- Client-facing APIs

The Chat Backend should **not contain complex agent orchestration logic**.

---

## 4.2 Agent Runtime

Recommended:

- Python
- Google ADK or the selected agent framework
- MCP support

Responsibilities:

- Agent orchestration
- Context construction
- LLM invocation
- Tool execution
- MCP execution
- Agent-to-agent communication
- Execution state
- Retry policies
- Runtime-level error handling

The Agent Runtime should be independent of the UI.

The Chat Backend should be able to say:

```text
Execute run:
    conversation = X
    trigger message = Y
```

without needing to know how the agent reaches the final answer.

---

## 4.3 Worker

Responsibilities:

- Conversation title generation
- Conversation summarization
- Embedding generation
- Search indexing
- Cleanup
- Other asynchronous operations

These operations should not block the main chat request.

Example:

```text
Message Created
       |
       +----> Chat response
       |
       +----> Background job
                    |
                    +--> Generate title
                    +--> Update summary
                    +--> Generate embedding
```

---

## 4.4 Reverse Proxy

Initially:

- IIS or Nginx

Responsibilities:

- TLS termination
- Routing
- Static frontend hosting if required
- Request limits
- Proxying SSE connections
- Basic security headers

---

# 5. Persistence Architecture

## PostgreSQL — Source of Truth

PostgreSQL should be the canonical persistence layer.

Initial entities:

```text
User
Conversation
Message
MessagePart
AgentRun
LLMInvocation
ToolExecution
Attachment
ConversationSummary
```

Conceptually:

```text
User
 |
 +-- Conversation
       |
       +-- Message
       |     |
       |     +-- MessagePart
       |
       +-- AgentRun
       |     |
       |     +-- LLMInvocation
       |     +-- ToolExecution
       |
       +-- Attachment
       |
       +-- ConversationSummary
```

---

# 6. Redis

Redis is not the source of truth.

Initial use cases:

- Conversation execution locks
- Short-lived cache
- Rate limiting
- Background job queue
- Temporary execution state

Example:

```text
conversation:{conversationId}:lock
```

This can be used to initially enforce:

> Only one active generation run per conversation.

This keeps concurrency behavior predictable.

---

# 7. File Storage

Initially, attachments can be stored on the Windows VM filesystem.

Example:

```text
D:\AIChat\data\attachments\
    user-{userId}\
        conversation-{conversationId}\
            file-001.pdf
            image-001.png
```

PostgreSQL stores only metadata:

```text
attachment_id
conversation_id
message_id
filename
mime_type
size
storage_path
created_at
```

The application should expose a storage abstraction:

```text
FileStorage
    |
    +-- LocalFileStorage       <-- initial
    +-- S3FileStorage          <-- future
    +-- GCSFileStorage         <-- future
    +-- AzureBlobStorage       <-- future
```

This avoids coupling the domain model to the local filesystem.

---

# 8. Conversation Model

A conversation is a durable user-facing entity.

Example:

```text
Conversation
---------------------------
id
user_id
title
status
created_at
updated_at
last_message_at
metadata
```

A conversation is **not** equivalent to an LLM context window.

A conversation may contain hundreds or thousands of messages.

---

# 9. Message Model

Messages represent user-visible conversational history.

Initial roles:

```text
USER
ASSISTANT
SYSTEM
TOOL
```

A message should support multiple content types instead of assuming it is always a text string.

Example:

```json
{
  "messageId": "msg_123",
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "Here is the architecture..."
    },
    {
      "type": "citation",
      "documentId": "doc_123"
    }
  ]
}
```

Potential content types:

```text
text
image
file
citation
structured_data
tool_result
```

---

# 10. Message Branching

The system should consider supporting:

```text
parent_message_id
```

even if branching is not exposed in the initial UI.

This allows future support for:

```text
A
|
B
|
C
```

becoming:

```text
       B -> C -> D
      /
A ---+
      \
       B' -> C'
```

This enables features such as:

- Edit message
- Regenerate response
- Fork conversation
- Alternate responses

---

# 11. Run Model

A **Run** represents an attempt to process a user request.

This is separate from the final assistant message.

Example:

```text
User Message
      |
      v
   Agent Run
      |
      +-- LLM Invocation
      |
      +-- Tool Execution
      |
      +-- LLM Invocation
      |
      v
Assistant Message
```

Initial run fields:

```text
AgentRun
---------------------------
id
conversation_id
trigger_message_id
status
started_at
completed_at
model
metadata
```

Possible statuses:

```text
QUEUED
RUNNING
COMPLETED
FAILED
CANCELLED
```

---

# 12. LLM Invocation

Each actual model invocation should be independently traceable.

```text
LLMInvocation
---------------------------
id
run_id
provider
model
input_tokens
output_tokens
latency_ms
status
started_at
completed_at
metadata
```

This allows tracking:

- Model performance
- Token consumption
- Cost
- Latency
- Failures
- Provider-specific behavior

---

# 13. Tool Execution

Tool calls should also be independently persisted.

```text
ToolExecution
---------------------------
id
run_id
tool_name
input
output
status
started_at
completed_at
error
metadata
```

Example execution:

```text
Agent Run
   |
   +-- LLM #1
   |
   +-- Tool: get_customer
   |
   +-- Tool: get_orders
   |
   +-- LLM #2
   |
   +-- Final response
```

This becomes particularly important for MCP and multi-agent systems.

---

# 14. Context Management

The Context Manager is one of the most important components.

Its job is to convert durable conversation history into an efficient LLM context.

```text
                   Conversation
                        |
                        v
                +----------------+
                | Context Manager|
                +----------------+
                  |     |     |
                  |     |     |
                  v     v     v
              Summary Recent Retrieval
                  \      |      /
                   \     |     /
                    v    v    v
                  LLM Context
                       |
                       v
                      LLM
```

It may eventually consider:

- System instructions
- Recent messages
- Conversation summary
- Relevant historical messages
- User memory
- Tool definitions
- Agent state
- Current user message

The Context Manager should not replace canonical history.

---

# 15. Streaming Architecture

Use **Server-Sent Events (SSE)** initially.

```text
Client
   |
   | POST message
   v
Chat Backend
   |
   v
Agent Runtime
   |
   | execution events
   v
Chat Backend
   |
   | SSE
   v
Client
```

The event stream should represent semantic execution events rather than only raw tokens.

Example:

```text
run.started

message.started

message.delta
message.delta
message.delta

tool.started
tool.completed

message.delta

message.completed

run.completed
```

This allows the UI to represent:

- Thinking/processing
- Tool execution
- Streaming response
- Completion
- Failure

without tightly coupling the UI to a specific LLM provider.

---

# 16. Request Lifecycle

The initial end-to-end flow:

```text
1. User enters message
           |
           v
2. React Client
           |
           v
3. Chat Backend
           |
           +--> Validate authentication
           |
           +--> Validate conversation ownership
           |
           +--> Check idempotency
           |
           +--> Persist USER message
           |
           v
4. Create Agent Run
           |
           v
5. Agent Runtime
           |
           v
6. Context Manager
           |
           v
7. LLM / Tool / MCP execution
           |
           v
8. Stream events through SSE
           |
           v
9. Persist final ASSISTANT message
           |
           v
10. Mark Run COMPLETED
           |
           v
11. Publish background jobs
           |
           +--> Title
           +--> Summary
           +--> Embedding
           +--> Analytics
```

---

# 17. Idempotency

This is required from the beginning.

The client should generate an idempotency key:

```text
Idempotency-Key: abc-123
```

The server must ensure that retries do not create duplicate logical messages or runs.

Example:

```text
Client
  |
  | Send message
  v
Server
  |
  +--> request succeeds
  |
  X response lost
  |
  Client retries
  |
  v
Server
  |
  +--> detect same idempotency key
  |
  +--> return existing result
```

Without this, unreliable networks can cause duplicate AI responses.

---

# 18. Concurrency

Initially, use serialized execution per conversation.

```text
Conversation A

Message 1
   |
   v
Run 1
   |
   v
Complete
   |
   v
Message 2
   |
   v
Run 2
```

Avoid initially allowing:

```text
Message 1 --> Run 1
Message 2 --> Run 2
Message 3 --> Run 3
```

simultaneously in the same conversation.

It creates ambiguity around context ordering.

Redis can provide the conversation-level lock.

---

# 19. Pagination

Never load an entire conversation by default.

Use cursor-based pagination.

Example:

```http
GET /v1/conversations/{conversationId}/messages
    ?limit=50
    &before={messageCursor}
```

The client should load recent messages first and fetch older messages on demand.

---

# 20. Conversation List

The sidebar should not retrieve every message.

Use a lightweight conversation endpoint:

```http
GET /v1/conversations
```

Example response:

```json
{
  "id": "conv_123",
  "title": "Spring Boot Architecture",
  "lastMessagePreview": "The recommended architecture...",
  "lastMessageAt": "2026-08-17T10:20:00Z"
}
```

This keeps the conversation list fast.

---

# 21. Background Processing

Do not perform secondary work inside the synchronous chat request.

Instead:

```text
Message Completed
       |
       v
Background Queue
       |
       +--> Title generation
       |
       +--> Summary generation
       |
       +--> Embedding
       |
       +--> Search indexing
       |
       +--> Analytics
```

Redis can initially act as the queue.

A dedicated event broker can be introduced later if scale requires it.

---

# 22. Search and Vector Storage

Do not introduce a dedicated vector database initially unless required.

Start with:

```text
PostgreSQL
     |
     +-- Normal relational storage
     |
     +-- pgvector (when required)
```

Potential future architecture:

```text
PostgreSQL
      |
      +--> pgvector
      |
      +--> Dedicated Search Engine
      |
      +--> Dedicated Vector DB
```

Only introduce these when actual search/memory requirements justify them.

---

# 23. Authentication and Authorization

The backend must derive the user identity from the authenticated session/token.

Do not trust:

```text
user_id
```

supplied by the client.

Every conversation access should effectively enforce:

```text
conversation.user_id == authenticated_user.id
```

Authorization should apply to:

- Conversation
- Message
- Attachment
- Run
- Tool execution
- Shared conversation

---

# 24. Deletion

Deleting a conversation must account for all associated data.

```text
Conversation
 |
 +-- Messages
 +-- Runs
 +-- Tool executions
 +-- Attachments
 +-- Summaries
 +-- Embeddings
 +-- Search index
 +-- Cache
```

Initial deletion flow:

```text
DELETE REQUESTED
       |
       v
Mark conversation deleted
       |
       v
Background cleanup
       |
       +--> Files
       +--> Embeddings
       +--> Cache
       +--> Search index
```

The system should define whether deletion is:

- Soft delete
- Hard delete
- Delayed hard delete

---

# 25. Observability

Every execution should have a correlation chain:

```text
request_id
    |
    v
conversation_id
    |
    v
run_id
    |
    +--> llm_invocation_id
    |
    +--> tool_execution_id
```

This enables debugging questions such as:

> Why did this response take 15 seconds?

Example:

```text
Total:               15.2 sec

Context creation:     0.4 sec
LLM invocation #1:    2.0 sec
Tool execution:       7.2 sec
LLM invocation #2:    5.4 sec
Persistence:          0.2 sec
```

Use structured logs from the beginning.

---

# 26. Initial Technology Stack

| Area | Initial Technology |
|---|---|
| Frontend | React + TypeScript |
| Chat Backend | Node.js + TypeScript |
| API Framework | Fastify or Express |
| Agent Runtime | Python |
| Agent Framework | Google ADK / selected agent framework |
| MCP | MCP-compatible runtime |
| Primary DB | PostgreSQL |
| Cache | Redis |
| Background Queue | Redis-backed queue |
| File Storage | Windows filesystem initially |
| Streaming | SSE |
| Reverse Proxy | IIS or Nginx |
| Deployment | Windows VM |
| Containers | Docker where practical |
| Logging | Structured application logs |
| Metrics | Basic application + VM metrics |
| Future Vector Search | pgvector |
| Future Analytics | Dedicated analytics store |
| Future Event Broker | Pub/Sub / Kafka / RabbitMQ |

---

# 27. What We Are Deliberately NOT Introducing Initially

Avoid premature infrastructure complexity.

```text
No Kubernetes
No GKE
No Cloud Run
No Kafka
No Pub/Sub
No Elasticsearch
No dedicated vector DB
No Cassandra
No MongoDB
No large microservice fleet
No dedicated conversation database
No dedicated message database
```

The initial system should be simple enough to operate on Windows VMs.

---

# 28. Modular Application Structure

Although the system has logical services, do not immediately turn every logical service into a deployable microservice.

### Chat Backend

```text
chat-backend/
|
+-- auth/
+-- conversation/
+-- message/
+-- run/
+-- attachment/
+-- streaming/
+-- persistence/
+-- api/
+-- common/
```

### Agent Runtime

```text
agent-runtime/
|
+-- orchestration/
+-- agents/
+-- context/
+-- llm/
+-- tools/
+-- mcp/
+-- execution/
+-- common/
```

These are modules first.

They can become independently deployable services later.

---

# 29. Important Architectural Boundary

The most important boundary is:

```text
                 EXPERIENCE PLANE
                 ---------------

       React Client
            |
            v
       Chat Backend
            |
            |
            v

                 EXECUTION PLANE
                 ---------------

       Agent Runtime
            |
       +----+----+-------+
       |    |    |       |
      LLM  MCP Tools  Agents
```

The Chat Backend should not need to know whether the response was produced by:

```text
Simple LLM
```

or:

```text
Agent A
   |
   +--> Agent B
   |
   +--> MCP
   |
   +--> Tool
   |
   +--> LLM
```

The execution details belong to the Agent Runtime.

The conversation system records the user-visible result and execution metadata.

---

# 30. Future Migration Path

The architecture should allow this:

### Initial

```text
Windows VM
|
+-- Chat Backend
+-- Agent Runtime
+-- Worker
+-- PostgreSQL
+-- Redis
```

### Later

```text
Cloud / Data Center

Chat Backend
      |
      +--> Managed PostgreSQL
      |
      +--> Managed Redis
      |
      +--> Object Storage
      |
      +--> Event Bus
      |
      +--> Agent Runtime
      |
      +--> Search / Vector Store
```

The goal is to make this a deployment change rather than a domain redesign.

---

# 31. Recommended Initial Deployment

For an early/internal system:

```text
                    Windows VM #1
                    ----------------

                    IIS / Nginx
                         |
                         v
                   Chat Backend
                         |
                  +------+------+
                  |             |
                  v             v
             PostgreSQL       Redis
                  |
                  |
                  v
             Agent Runtime
                  |
            +-----+-----+
            |           |
            v           v
          LLMs       MCP/Tools
```

If load increases:

```text
Windows VM #1
----------------
IIS / Nginx
Chat Backend
Agent Runtime


Windows VM #2
----------------
PostgreSQL
Redis


Windows VM #3
----------------
Workers
```

The application should use configuration to determine where each dependency lives.

---

# 32. Initial Design Principles

The following principles should guide implementation:

1. **PostgreSQL is the source of truth.**
2. **Conversation history is independent from LLM context.**
3. **Runs are separate from messages.**
4. **Agent execution is independent from the Chat API.**
5. **Messages should support multimodal content.**
6. **Use idempotency for message submission.**
7. **Serialize runs per conversation initially.**
8. **Use cursor pagination.**
9. **Stream responses using SSE.**
10. **Keep secondary processing asynchronous.**
11. **Do not store large files in PostgreSQL.**
12. **Keep Redis non-authoritative.**
13. **Use structured logging and correlation IDs.**
14. **Keep logical modules separate before making them microservices.**
15. **Avoid cloud-specific dependencies in the domain layer.**
16. **Design storage interfaces so local storage can later become object storage.**
17. **Keep the Chat Backend independent from the underlying agent implementation.**
18. **Make derived data rebuildable from canonical data.**

---

# 33. Recommended Next Design Phase

The next phase should focus on the **domain model and request lifecycle**, before infrastructure expansion.

Specifically design:

```text
1. Conversation
2. Message
3. MessagePart
4. AgentRun
5. LLMInvocation
6. ToolExecution
7. Attachment
8. ConversationSummary
```

Then walk through these scenarios:

```text
Create conversation
        ↓
Send message
        ↓
Stream response
        ↓
Successful completion

Send message
        ↓
LLM failure
        ↓
Retry

Send message
        ↓
Tool failure
        ↓
Agent retry

Send message
        ↓
Client timeout
        ↓
Client retry
        ↓
Idempotency

Edit message
        ↓
Branch conversation
        ↓
Generate new response

Delete conversation
        ↓
Cleanup derived data
```

Once these flows are defined, the following can be derived cleanly:

```text
Domain Model
      ↓
Database Schema
      ↓
API Contracts
      ↓
Execution Model
      ↓
Concurrency Model
      ↓
Event Model
      ↓
Infrastructure
```

This order should be preferred over starting with infrastructure components.
