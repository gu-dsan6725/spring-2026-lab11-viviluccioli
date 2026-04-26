# Task 3 Observations: A2A Communication and Agent Discovery

## A2A Messages Exchanged Between Agents

The Travel Assistant receives a user request over the A2A protocol and, when it needs booking capabilities, sends an A2A message to the Flight Booking Agent.

**Inbound request to Travel Assistant (from test client):**
```json
{
  "jsonrpc": "2.0",
  "id": "test-I want to b",
  "method": "message/send",
  "params": {
    "message": {
      "role": "user",
      "parts": [{"kind": "text", "text": "I want to book flight ID 1. I need you to reserve 2 seats, confirm the reservation, and process the payment. You don't have these booking capabilities yourself, so you'll need to find and use an agent that can handle flight reservations and confirmations."}],
      "messageId": "msg-I want to b"
    }
  }
}
```

**Outbound request from Travel Assistant to Flight Booking Agent (A2A delegation):**
The Travel Assistant constructs a similar JSON-RPC 2.0 `message/send` payload addressed to `http://127.0.0.1:10002/` with a natural-language instruction such as:
```json
{
  "jsonrpc": "2.0",
  "id": "<generated-id>",
  "method": "message/send",
  "params": {
    "message": {
      "role": "user",
      "parts": [{"kind": "text", "text": "Reserve 2 seats on flight ID 1, confirm the booking, and process the payment."}],
      "messageId": "<generated-id>"
    }
  }
}
```

**Response from Flight Booking Agent back to Travel Assistant:**
```json
{
  "id": "<generated-id>",
  "jsonrpc": "2.0",
  "result": {
    "artifacts": [
      {
        "artifactId": "<uuid>",
        "name": "agent_response",
        "parts": [{"kind": "text", "text": "Reservation confirmed. Booking number BK…"}]
      }
    ],
    "status": {"state": "completed"}
  }
}
```

## How the Travel Assistant Discovered the Flight Booking Agent

Discovery happens in three steps:

1. **Query the Registry Stub** — The Travel Assistant calls its `discover_remote_agents` tool, which sends a `POST` request to the Registry Stub at `http://127.0.0.1:7861/api/agents/discover/semantic?query=book+flights&max_results=5`.

2. **Registry returns agent metadata** — The Registry Stub responds with the Flight Booking Agent's record, including its URL (`http://127.0.0.1:10002`), description, and skill list. In this stub, the same agent is returned for any query; a real registry would use semantic/vector search over all registered agents.

3. **Cache and invoke** — The Travel Assistant caches the discovered agent entry and uses the `invoke_remote_agent` tool to send the A2A message to the Flight Booking Agent's URL.

## JSON-RPC Request/Response Format Observed

All agent-to-agent communication uses **JSON-RPC 2.0** over HTTP POST:

| Field | Value / Purpose |
|-------|----------------|
| `jsonrpc` | Always `"2.0"` |
| `method` | `"message/send"` — the A2A method for sending a message |
| `id` | Unique request identifier for correlating responses |
| `params.message.role` | `"user"` for messages sent to an agent |
| `params.message.parts` | Array of content parts; each part has `"kind": "text"` and a `"text"` field |
| `params.message.messageId` | Unique message identifier |

Responses follow the same envelope with a `result` field containing:
- `artifacts` — the agent's output, each with `parts` containing the response text
- `history` — the full conversation history for the task
- `status` — task state (`"completed"`, etc.) and timestamp
- `contextId` / `id` — task tracking identifiers

## What Was in the Agent Card and How It Was Used

Each agent exposes its agent card at `/.well-known/agent-card.json`. The card contains:

| Field | Travel Assistant | Flight Booking Agent |
|-------|-----------------|---------------------|
| `name` | Travel Assistant Agent | Flight Booking Agent |
| `description` | Flight search and trip planning agent with dynamic agent discovery | Flight booking and reservation management agent |
| `url` | `http://127.0.0.1:10001/` | `http://127.0.0.1:10002/` |
| `protocolVersion` | `0.3.0` | `0.3.0` |
| `preferredTransport` | `JSONRPC` | `JSONRPC` |
| `capabilities.streaming` | `true` | `true` |
| `skills` | search_flights, check_prices, get_recommendations, create_trip_plan, discover_remote_agents, view_cached_remote_agents, invoke_remote_agent | check_availability, reserve_flight, confirm_booking, process_payment, manage_reservation |

**How the card was used:** When the Travel Assistant discovers the Flight Booking Agent from the registry, it fetches the agent card to confirm the agent's endpoint URL and verify its skills before delegating. The skill list tells the Travel Assistant exactly what capabilities (check_availability, reserve_flight, confirm_booking, process_payment) the remote agent offers, allowing it to frame the delegation message appropriately.

## Benefits and Limitations of This Approach

### Benefits

- **Loose coupling** — Agents only need to know the registry address, not each other's addresses. A new specialist agent can be added to the registry and immediately discoverable by any other agent without code changes.
- **Capability-based routing** — The agent card's `skills` list lets a consumer agent decide at runtime whether the discovered agent can actually handle its request, rather than hardcoding assumptions.
- **Standardized protocol** — JSON-RPC 2.0 over HTTP is language- and framework-agnostic. Any agent that speaks the A2A protocol can interoperate, regardless of implementation language.
- **Natural-language delegation** — The invoking agent sends a plain-text message to the remote agent. The remote agent interprets it with its own LLM, so the caller does not need to know the remote API's parameter schema.
- **Streaming support** — Both agents declare `"streaming": true`, enabling incremental response delivery for long-running tasks.

### Limitations

- **Stub registry is not semantic** — The current registry always returns the Flight Booking Agent regardless of the query. A production system would need real vector/embedding-based semantic search to handle multiple registered agents and correctly match capabilities.
- **No authentication or authorization** — The A2A messages and agent card endpoints have no authentication. A malicious caller could impersonate an agent or enumerate all registered agents.
- **No schema validation on delegation** — The Travel Assistant sends a free-form natural-language string to the Flight Booking Agent. There is no structured contract (e.g., function signatures or typed parameters), so the quality of the interaction depends on prompt engineering and LLM interpretation.
- **Single-hop discovery** — The system only supports one level of delegation. There is no mechanism for chained discovery (an agent discovering another agent that itself discovers a third agent) with proper context propagation.
- **In-memory state** — Both agents store bookings and trip plans in memory. Restarting a service loses all state; a production system would need persistent storage.
- **Latency** — Each A2A message round-trip involves an LLM inference call (5–7 seconds observed). Multi-step workflows accumulate this latency, making the system noticeably slower than direct API calls.
