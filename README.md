![AMTP Project Logo](logo.png)

# AMTP Protocol Specification v1.0 (Draft)

## Table of Contents

1. [Introduction](#1-introduction)
2. [Protocol Architecture](#2-protocol-architecture)
3. [Addressing and Discovery](#3-addressing-and-discovery)
4. [Message Format](#4-message-format)
5. [Transport Layer](#5-transport-layer)
6. [Schema Framework Integration](#6-schema-framework-integration)
7. [Multi-Agent Coordination](#7-multi-agent-coordination)
8. [Reliability and Delivery Guarantees](#8-reliability-and-delivery-guarantees)
9. [Security Model](#9-security-model)
10. [Gateway Implementation](#10-gateway-implementation)
11. [Protocol Negotiation and Bridging](#11-protocol-negotiation-and-bridging)
12. [API Specifications](#12-api-specifications)
13. [Error Handling](#13-error-handling)
14. [Compliance and Conformance](#14-compliance-and-conformance)

---

## 1. Introduction

### 1.1 Protocol Overview

The Agent Message Transfer Protocol (AMTP) is a federated, asynchronous communication protocol designed for reliable agent-to-agent communication across organizational boundaries. AMTP extends the email addressing model with native support for structured data, multi-agent coordination, and guaranteed delivery semantics.

### 1.2 Key Features

- Universal addressing using `agent@domain` format
- Transparent protocol upgrade with SMTP bridging
- At-least-once delivery with idempotency guarantees
- Standard schema integration via AGNTCY framework
- Multi-agent workflow coordination
- Federated architecture with DNS-based discovery

### 1.3 Protocol Versions

This document specifies AMTP Protocol Version 1.0. Version negotiation is handled through DNS TXT records and HTTP headers.

---

## 2. Protocol Architecture

### 2.1 System Components

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Agent A       │────▶│  AMTP Gateway   │────▶│  AMTP Gateway   │
│ (sender.com)    │     │  (sender.com)   │     │ (receiver.com)  │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                                 │                        │
                                 ▼                        ▼
                        ┌─────────────────┐     ┌─────────────────┐
                        │ Message Queue   │     │   Agent B       │
                        │ (Optional)      │     │ (receiver.com)  │
                        └─────────────────┘     └─────────────────┘
```

Messages MAY include a logical schema identifier (e.g., schema:"agntcy:commerce.order.v2"), but AMTP Core makes no assumptions about how that identifier is minted, stored, versioned, or resolved. Gateways and participants MAY validate payloads against schemas using any mechanism (or none).

AMTP does not mandate any particular schema  or agent registry. Each implementation is free to choose their own registry architecture: centralized, federated, file-based, database-driven, or any other approach that meets their organizational needs.


### 2.2 Protocol Stack

```
┌─────────────────────────────────────┐
│        Application Layer            │ ← Agent Logic
├─────────────────────────────────────┤
│         AMTP Protocol               │ ← Message Format & Coordination
├─────────────────────────────────────┤
│      Transport Layer (HTTPS)       │ ← Reliable Delivery
├─────────────────────────────────────┤
│       Discovery Layer (DNS)        │ ← Addressing & Capabilities
├─────────────────────────────────────┤
│         Network Layer (IP)          │ ← Standard Internet
└─────────────────────────────────────┘
```

---

## 3. Addressing and Discovery

### 3.1 Address Format

AMTP uses the standard email address format: `local-part@domain`

**Examples:**
- `purchase-agent@acme.com`
- `logistics.coordinator@shipper.org`
- `ai-assistant@company.com`

### 3.2 DNS Discovery

#### 3.2.1 AMTP Capability Advertisement

Domain operators MUST publish a DNS TXT record at `_amtp.{domain}` to advertise AMTP capabilities:

```dns
_amtp.example.com. IN TXT "v=amtp1;gateway=https://amtp.example.com:443"
```

**Parameters:**
- `v`: Protocol version (required)
- `gateway`: AMTP gateway endpoint URL (required)
- `auth`: Authentication methods (`cert`, `oauth`, `apikey`) (optional)
- `max-size`: Maximum message size in bytes (optional, default: 10MB)

#### 3.2.2 Discovery Algorithm

1. Query DNS TXT record for `_amtp.{domain}`
2. If AMTP record exists and is valid, use AMTP protocol
3. Otherwise, fall back to standard MX record lookup for SMTP
4. Cache discovery results with TTL from DNS record

### 3.3 Gateway Endpoint Resolution

The `gateway` parameter in the DNS TXT record specifies the HTTPS endpoint for AMTP message delivery. The endpoint MUST:
- Use HTTPS with valid TLS certificate for the domain
- Accept HTTP POST requests to `/v1/messages` and GET requests to `/v1/discovery/agents`, `/v1/inbox/*`
- Support HTTP/2 for improved performance

### 3.4 Agent Discovery (Optional)

Agent discovery provides flexibility for different messaging scenarios. Gateways MAY implement agent discovery endpoints to support various use cases.

#### 3.4.1 Use Cases

**Direct Addressing:**
When senders already know the destination address (e.g., `sales@somecompany.com`), agent discovery is not required. Messages can be sent directly to known addresses.

**True Discovery:**
When senders need to find appropriate agents for a domain (e.g., "who handles purchase orders at company.com"), discovery enables programmatic agent selection based on capabilities, roles, or supported schemas.

#### 3.4.2 Discovery Endpoint

**Endpoint:** `GET https://{gateway}/v1/discovery/agents`

**Query Parameters:**
- `address` (optional): Query specific agent address for validation (e.g., `sales@example.com`)
- `role` (optional): Filter by business role
- `delivery_mode` (optional): Filter agents by delivery mode (`pull` or `push`)
- `active_only` (optional): Only return agents active within the last 30 days (`true` or `false`)
- `capability` (optional): Filter by semantic capabilities
- `schema` (optional): Filter by supported schema patterns (e.g., `agntcy:commerce.*`)

**Response Examples:**

*Full agent listing:*
```json
{
  "agents": [
    {
      "address": "sales@example.com",
      "delivery_mode": "pull",
      "capabilities": ["sales_inquiries", "product_demos"],
      "supported_schemas": ["agntcy:commerce.inquiry.*"],
      "role": "sales",
      "description": "Handles sales inquiries and product demonstrations",
      "last_active": "2024-01-15T10:30:00Z"
    },
    {
      "address": "support@example.com", 
      "delivery_mode": "push",
      "webhook_url": "https://example.com/webhooks/amtp",
      "capabilities": ["technical_support", "issue_resolution"],
      "supported_schemas": ["agntcy:support.ticket.*"],
      "role": "support",
      "description": "Technical support and issue resolution",
      "last_active": "2024-01-15T14:22:00Z"
    }
  ],
  "agent_count": 2,
  "domain": "example.com",
  "timestamp": "2024-01-15T15:00:00Z"
}
```

*Address validation (using `?address=sales@example.com`):*
```json
{
  "agents": [
    {
      "address": "sales@example.com",
      "delivery_mode": "pull",
      "active": true,
      "last_active": "2024-01-15T10:30:00Z"
    }
  ],
  "agent_count": 1,
  "domain": "example.com",
  "timestamp": "2024-01-15T15:00:00Z"
}
```

#### 3.4.3 Implementation Flexibility

Gateways MAY implement discovery with different levels of detail:

- **Minimal**: Address validation only
- **Basic**: Simple agent listing without metadata
- **Enhanced**: Rich metadata with semantic search capabilities

Clients SHOULD cache discovery responses appropriately to reduce load on gateways. Discovery is entirely optional - agents can communicate effectively using only direct addressing.

---

## 4. Message Format

### 4.1 Message Structure

AMTP messages use JSON format with the following structure:

```json
{
  "version": "1.0",
  "message_id": "01H8X9Z2K3M4N5P6Q7R8S9T0",
  "idempotency_key": "uuid-v4-string",
  "timestamp": "2025-08-14T10:30:00.000Z",
  "sender": "agent@sender.com",
  "recipients": ["agent1@receiver.com", "agent2@partner.com"],
  "subject": "Purchase Order Confirmation",
  "schema": "agntcy:commerce.order.v2",
  "coordination": {
    "type": "parallel",
    "timeout": 3600,
    "required_responses": ["agent1@receiver.com"]
  },
  "headers": {
    "priority": "normal",
    "reply-to": "agent@sender.com",
    "custom-header": "custom-value"
  },
  "payload": {
    // Schema-validated content
  },
  "attachments": [
    {
      "filename": "invoice.pdf",
      "content_type": "application/pdf",
      "size": 1024000,
      "hash": "sha256:abc123...",
      "url": "https://attachments.sender.com/xyz789"
    }
  ],
  "signature": {
    "algorithm": "RS256",
    "value": "base64-encoded-signature"
  }
}
```

### 4.2 Field Specifications

#### 4.2.1 Core Required Fields

The following fields are MANDATORY for all AMTP implementations and ensure basic interoperability:

- **`version`**: AMTP protocol version (string, e.g., "1.0")
- **`sender`**: Sender address in `agent@domain` format
- **`recipients`**: Array of recipient addresses
- **`payload`**: Message content (any valid JSON object or primitive)

#### 4.2.2 Implementation-Defined Fields (Optional)

The following fields MAY be implemented based on specific requirements and use cases:

**Reliability & Tracking:**
- **`message_id`**: UUIDv7 for time-ordered unique identification (recommended for message tracking)
- **`idempotency_key`**: UUIDv4 for duplicate detection (required for at-least-once delivery semantics)
- **`timestamp`**: ISO 8601 UTC timestamp (may be added by gateways)

**Message Metadata:**
- **`subject`**: Human-readable message summary
- **`headers`**: Additional custom metadata
- **`in_reply_to`**: Reference to original message for responses

**Schema & Validation:**
- **`schema`**: Schema identifier for payload validation (e.g., "agntcy:commerce.order.v2")

**Advanced Features:**
- **`coordination`**: Multi-agent workflow configuration
- **`attachments`**: External file references
- **`signature`**: Digital signature for non-repudiation

### 4.3 Message Size Limits

- Maximum message size: 10MB (configurable via DNS)
- Attachments: Referenced by URL, not embedded
- Large payloads: Use chunking or external storage

---

## 5. Transport Layer

### 5.1 Delivery Paths

AMTP Core defines delivery semantics independent of storage mechanisms. Implementations MAY choose between two paths per deployment or per message policy:

- **Immediate Path (Optional):** The gateway processes and forwards messages directly without requiring durable store-and-forward prior to acknowledging the sender. This path targets low-latency, real-time scenarios and is best-effort with documented durability limits.

- **Durable Path (Recommended):** The gateway persists messages (e.g., WAL/disk or equivalent) before acknowledging acceptance to the sender, providing at-least-once delivery across restarts and transient failures.

Gateways MAY route messages to either path based on local policy. No new headers are required by AMTP Core to use either path.


### 5.2 HTTP Transport

AMTP uses HTTPS as the primary transport protocol.

#### 5.2.1 Message Delivery Endpoint

**Endpoint:** `POST /v1/messages`

**Request Headers:**
```http
Content-Type: application/json
Content-Length: {size}
User-Agent: AMTP/1.0 ({implementation})
X-AMTP-Version: 1.0
Authorization: Bearer {token} (if required)
```

**Response Codes:**
- `200 OK`: **Immediate Path only.** The gateway has obtained a correlated reply within a short processing window and returns the reply payload in the response body. AMTP remains asynchronous at the protocol level; this is an implementation optimization.
- `202 Accepted`: Message accepted for delivery (either Immediate or Durable Path). Sender MAY poll status or await a reply message referencing `in_reply_to`.
- `400 Bad Request`: Invalid message format
- `401 Unauthorized`: Authentication required
- `413 Payload Too Large`: Message exceeds size limit
- `429 Too Many Requests`: Rate limit exceeded
- `503 Service Unavailable`: Gateway temporarily unavailable


#### 5.2.2 Status Query Endpoint

**Endpoint:** `GET /v1/messages/{message_id}/status`

**Response:**
```json
{
  "message_id": "01H8X9Z2K3M4N5P6Q7R8S9T0",
  "status": "delivered",
  "recipients": [
    {
      "address": "agent@receiver.com",
      "status": "delivered",
      "timestamp": "2025-08-14T10:31:45.000Z"
    }
  ],
  "attempts": 1,
  "next_retry": null
}
```

#### 5.2.3 Inbox Management (Pull Mode)

For agents using pull-based delivery, gateways provide inbox endpoints for message retrieval and acknowledgment.

**Inbox Retrieval:** `GET /v1/inbox/{recipient}`

Agents can retrieve pending messages from their inbox with optional filtering and pagination. Authentication is required via API key to ensure agents can only access their own messages.

**Message Acknowledgment:** `DELETE /v1/inbox/{recipient}/{message_id}`

After processing a message, agents acknowledge receipt to remove it from their inbox. This ensures at-least-once delivery semantics while preventing message duplication.

**Security Model:**
- Each agent requires a unique API key for inbox access
- Agents can only access messages addressed to their own local-part
- Acknowledged messages are permanently removed from the inbox
- Unacknowledged messages remain available for retrieval

See section 12.1.5 for detailed API specifications.

### 5.3 Store-and-Forward Architecture (Optional)

#### 5.3.1 Message Queue (optional)

Each AMTP gateway maintains optional persistent message queues for:
- Outbound messages awaiting delivery
- Inbound messages for local delivery
- Dead letter queue for failed messages

#### 5.3.2 Retry Logic

**Retry Schedule:**
- Initial delay: 1 second
- Exponential backoff: 2^n seconds (capped at 1 hour)
- Maximum retries: 168 (7 days with hourly retries)
- Jitter: ±25% randomization to prevent thundering herd

---

## 6. Schema Framework Integration

### 6.1 AGNTCY Schema Support

AMTP integrates with the AGNTCY standard schema framework for payload validation and semantic interoperability.

#### 6.1.1 Schema Identifier Format

`agntcy:{domain}.{entity}.{version}`

**Examples:**
- `agntcy:commerce.order.v2`
- `agntcy:finance.payment.v1`
- `agntcy:logistics.shipment.v3`

#### 6.1.2 Schema Resolution

Schema resolution is **implementation-defined**. A typical flow might include:

1. Parse schema identifier from message
2. Resolve schema definition via the receiving gateway's Schema Engine from a trusted local cache; the gateway MAY fetch from a configured registry (AGNTCY, internal, or other) to populate/refresh the cache; fetched schemas MUST be signature-verified before use
3. Validate payload against schema
4. Reject invalid messages with detailed error response

The protocol does not specify registry endpoints, discovery mechanisms, or storage formats. Each implementation chooses their own schema management approach.

### 6.2 Schema Negotiation

#### 6.2.1 Schema Discovery

Schema discovery mechanisms are **implementation-defined**. The protocol does not mandate how agents or senders discover which schemas are supported by recipients.

**Common implementation approaches may include:**
- Agent-level advertisement through discovery endpoints
- Out-of-band negotiation and configuration
- Schema registries (centralized or federated)
- Static configuration files
- Runtime schema probing and error handling

#### 6.2.2 Version Compatibility

- Minor version differences: Backward compatible
- Major version differences: Explicit negotiation required
- Unsupported schema: Return error with supported alternatives

---

## 7. Multi-Agent Coordination

### 7.1 Coordination Patterns

#### 7.1.1 Parallel Execution

All recipients process the message simultaneously:

```json
{
  "coordination": {
    "type": "parallel",
    "timeout": 3600,
    "required_responses": ["critical@agent.com"],
    "optional_responses": ["notify@agent.com"]
  }
}
```

#### 7.1.2 Sequential Execution

Recipients process in specified order:

```json
{
  "coordination": {
    "type": "sequential",
    "timeout": 7200,
    "sequence": ["first@agent.com", "second@agent.com", "final@agent.com"],
    "stop_on_failure": true
  }
}
```

#### 7.1.3 Conditional Execution

Execution based on response conditions:

```json
{
  "coordination": {
    "type": "conditional",
    "timeout": 1800,
    "conditions": [
      {
        "if": "approval@manager.com responds with approve=true",
        "then": ["execute@system.com"],
        "else": ["reject@system.com"]
      }
    ]
  }
}
```

### 7.2 Workflow State Management

#### 7.2.1 State Tracking

The originating gateway maintains workflow state:

```json
{
  "workflow_id": "01H8X9Z2K3M4N5P6Q7R8S9T0",
  "status": "in_progress",
  "created_at": "2025-08-14T10:30:00.000Z",
  "updated_at": "2025-08-14T10:35:15.000Z",
  "participants": [
    {
      "address": "agent1@domain.com",
      "status": "completed",
      "response": {...}
    },
    {
      "address": "agent2@domain.com", 
      "status": "pending",
      "deadline": "2025-08-14T11:30:00.000Z"
    }
  ]
}
```

#### 7.2.2 Response Handling

Responses reference the original workflow:

```json
{
  "version": "1.0",
  "message_id": "01H8X9Z2K3M4N5P6Q7R8S9T1",
  "in_reply_to": "01H8X9Z2K3M4N5P6Q7R8S9T0",
  "sender": "agent1@domain.com",
  "recipients": ["coordinator@sender.com"],
  "response_type": "workflow_response",
  "payload": {
    "status": "approved",
    "data": {...}
  }
}
```

---

## 8. Reliability and Delivery Guarantees

### 8.1 Delivery Semantics

AMTP provides **at-least-once** delivery guarantees across federated gateways.

#### 8.1.1 Idempotency

- Every message includes a unique `idempotency_key`
- Receiving gateways MUST deduplicate based on this key
- Idempotency window: 7 days minimum

#### 8.1.2 Acknowledgment Protocol

1. Gateway receives message → Returns `202 Accepted`
2. Gateway attempts delivery → Updates internal status
3. Successful delivery → Mark as delivered
4. Failed delivery → Schedule retry with exponential backoff

### 8.2 Failure Handling

#### 8.2.1 Dead Letter Queue

Messages that fail after maximum retries are moved to DLQ:

```json
{
  "message_id": "01H8X9Z2K3M4N5P6Q7R8S9T0",
  "original_message": {...},
  "failure_reason": "recipient_unavailable",
  "attempts": 168,
  "first_attempt": "2025-08-14T10:30:00.000Z",
  "last_attempt": "2025-08-21T10:30:00.000Z",
  "dlq_timestamp": "2025-08-21T11:00:00.000Z"
}
```

#### 8.2.2 Error Reporting

Failed deliveries generate error reports to sender:

```json
{
  "version": "1.0",
  "message_type": "delivery_failure",
  "original_message_id": "01H8X9Z2K3M4N5P6Q7R8S9T0",
  "failed_recipients": ["unreachable@domain.com"],
  "error_code": "recipient_unavailable",
  "error_message": "Domain gateway not responding",
  "retry_count": 168,
  "final_attempt": "2025-08-21T10:30:00.000Z"
}
```

---

## 9. Security Model

### 9.1 Transport Security

- **TLS 1.3**: Mandatory for all HTTPS connections
- **Certificate Validation**: Gateway certificates MUST be valid for the domain
- **HTTP Strict Transport Security (HSTS)**: Recommended

### 9.2 Authentication

#### 9.2.1 Domain-Based Authentication

Default authentication uses domain ownership verification through TLS certificates.

#### 9.2.2 Additional Authentication Methods

**API Keys:**
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**OAuth 2.0:**
```http
Authorization: Bearer oauth-access-token
```

**Mutual TLS:**
Client certificate authentication for high-security environments.

### 9.3 Message Integrity

#### 9.3.1 Digital Signatures

Optional message signing using RS256:

```json
{
  "signature": {
    "algorithm": "RS256",
    "keyid": "sender.com:key1",
    "value": "base64-encoded-signature"
  }
}
```

#### 9.3.2 End-to-End Encryption

Optional payload encryption:

```json
{
  "payload": {
    "encrypted": true,
    "algorithm": "AES-256-GCM",
    "encrypted_data": "base64-encoded-data",
    "recipients": [
      {
        "keyid": "receiver.com:pubkey1",
        "encrypted_key": "base64-encoded-key"
      }
    ]
  }
}
```

### 9.4 Access Control

#### 9.4.1 Domain Policies

Configure allowed senders and schemas:

```yaml
# amtp-policy.yaml
version: "1.0"
default_policy: "deny"
rules:
  - sender_pattern: "*.trusted-partner.com"
    action: "allow"
    schemas: ["agntcy:commerce.*"]
  - sender_pattern: "public@anyone.com"
    action: "allow"
    schemas: ["agntcy:public.*"]
    rate_limit: "100/hour"
```

---

## 10. Gateway Implementation

### 10.1 Core Components

```
┌─────────────────────────────────────────────────────────────┐
│                    AMTP Gateway                             │
├─────────────────┬─────────────────┬─────────────────────────┤
│  HTTP Server    │  Message Queue  │    Protocol Bridge     │
│  - Receive      │  - Persistence  │    - AMTP ↔ SMTP       │
│  - Send         │  - Retry Logic  │    - Schema Conversion  │
│  - Status API   │  - DLQ          │    - Format Translation │
├─────────────────┼─────────────────┼─────────────────────────┤
│  DNS Resolver   │  Schema Engine  │    Coordination Engine  │
│  - Discovery    │  - Validation   │    - Workflow State     │
│  - Caching      │  - AGNTCY API   │    - Multi-Agent Logic  │
├─────────────────┼─────────────────┼─────────────────────────┤
│  Auth Manager   │  Policy Engine  │    Monitoring          │
│  - TLS Certs    │  - Access Rules │    - Metrics           │
│  - API Keys     │  - Rate Limits  │    - Logging           │
└─────────────────┴─────────────────┴─────────────────────────┘
```

### 10.2 Delivery to Local Agents

When an AMTP gateway receives a message for an address in its own domain
(e.g., `orders@receiver.com`), it is responsible for delivery to the local
agent identified by the local-part. AMTP Core does not prescribe how this
delivery is implemented; each domain is free to choose.

Gateways MAY deliver messages using one or both of the following models:

- **Push Model:** The gateway invokes a registered delivery target for the
  local-part, such as an HTTPS webhook, gRPC service, or internal worker queue.
  If the target is temporarily unavailable, the gateway MAY retry delivery
  according to its durability policy.

- **Pull Model:** The gateway buffers messages internally and exposes them
  through a retrieval API (e.g., `GET /v1/inbox/{recipient}/...`). Agents fetch
  messages at their convenience and acknowledge receipt. This model is useful
  for agents that cannot expose inbound endpoints.

**Normative requirement:** Regardless of delivery model, the gateway MUST
either (a) deliver the message to a local agent, or (b) return a structured
error such as `AGENT_UNKNOWN` if no mapping exists for the local-part.

---

## 11. Protocol Negotiation and Bridging

### 11.1 Discovery Process

```mermaid
flowchart TD
    A[Agent sends to agent@domain.com] --> B[Query DNS for _amtp.domain.com]
    B --> C{AMTP record found?}
    C -->|Yes| D[Use AMTP protocol]
    C -->|No| E[Query MX records]
    E --> F[Use SMTP Bridging]
    D --> G[Send AMTP message]
    G --> H{Delivery successful?}
    H -->|Yes| I[Done]
    H -->|No| J[Retry with exponential backoff]
    J --> K{Max retries exceeded?}
    K -->|No| G
    K -->|Yes| F
```

### 11.2 SMTP Bridge

#### 11.2.1 AMTP to Email Conversion

When falling back to SMTP, convert AMTP message to email format:

**Email Headers:**
```
From: amtp-bridge@sender.com
To: agent@receiver.com
Subject: [AMTP] Original Subject
X-AMTP-Message-ID: 01H8X9Z2K3M4N5P6Q7R8S9T0
X-AMTP-Schema: agntcy:commerce.order.v2
X-AMTP-Sender: agent@sender.com
Content-Type: multipart/mixed
```

**Email Body:**
```
This message was sent via AMTP protocol but delivered via SMTP bridging.

Original Message:
Subject: Purchase Order Confirmation
From: agent@sender.com
Schema: agntcy:commerce.order.v2

Human-readable summary:
A purchase order for 100 widgets has been confirmed.

Structured Data (JSON):
{
  "order_id": "12345",
  "items": [...]
}

To upgrade to native AMTP support, add DNS TXT record:
_amtp.receiver.com. IN TXT "v=amtp1;gateway=https://amtp.receiver.com"
```

#### 11.2.2 Email to AMTP Conversion

Process incoming emails and attempt AMTP delivery:

1. Parse email headers for AMTP metadata
2. Extract structured data from email body
3. Reconstruct AMTP message format
4. Deliver via AMTP if recipient supports it
5. Otherwise, deliver as standard email

---

## 12. API Specifications

### 12.1 Gateway REST API

#### 12.1.1 Send Message

**Endpoint:** `POST /v1/messages`

**Request Body:**
```json
{
  "version": "1.0",
  "sender": "agent@sender.com",
  "recipients": ["agent@receiver.com"],
  "subject": "Test Message",
  "schema": "agntcy:test.message.v1",
  "payload": {
    "text": "Hello, World!"
  }
}
```

**Response:**
```json
{
  "message_id": "01H8X9Z2K3M4N5P6Q7R8S9T0",
  "status": "accepted",
  "recipients": [
    {
      "address": "agent@receiver.com",
      "status": "queued"
    }
  ]
}
```

#### 12.1.2 Query Message Status

**Endpoint:** `GET /v1/messages/{message_id}`

**Response:**
```json
{
  "message_id": "01H8X9Z2K3M4N5P6Q7R8S9T0",
  "status": "delivered",
  "created_at": "2025-08-14T10:30:00.000Z",
  "delivered_at": "2025-08-14T10:30:15.000Z",
  "recipients": [
    {
      "address": "agent@receiver.com",
      "status": "delivered",
      "delivered_at": "2025-08-14T10:30:15.000Z",
      "attempts": 1
    }
  ]
}
```

#### 12.1.3 List Messages

**Endpoint:** `GET /v1/messages`

**Query Parameters:**
- `status`: Filter by status (pending, delivered, failed)
- `sender`: Filter by sender address
- `recipient`: Filter by recipient address
- `since`: Messages since timestamp
- `limit`: Number of results (default: 100, max: 1000)
- `offset`: Pagination offset

#### 12.1.4 Webhook Configuration

**Endpoint:** `POST /v1/webhooks`

Configure webhooks for delivery notifications:

```json
{
  "url": "https://my-app.com/amtp-webhook",
  "events": ["message.delivered", "message.failed"],
  "secret": "webhook-secret-key"
}
```

#### 12.1.5 Inbox Management (Pull Mode)

For agents using pull-based message delivery, gateways provide inbox management endpoints.

##### Get Inbox Messages

**Endpoint:** `GET /v1/inbox/{recipient}`

**Headers:**
```http
Authorization: Bearer {agent_api_key}
```

**Query Parameters:**
- `limit` (optional): Number of messages to retrieve (default: 100, max: 1000)
- `offset` (optional): Pagination offset
- `since` (optional): Only return messages since timestamp
- `status` (optional): Filter by message status (`unread`, `read`)

**Response:**
```json
{
  "messages": [
    {
      "message_id": "01H8X9Z2K3M4N5P6Q7R8S9T0",
      "sender": "agent@sender.com",
      "subject": "Process Order",
      "schema": "agntcy:commerce.order.v2",
      "timestamp": "2025-08-14T10:30:00.000Z",
      "status": "unread",
      "payload": {
        "order_id": "12345",
        "customer": {...},
        "items": [...]
      }
    }
  ],
  "message_count": 1,
  "recipient": "orders@receiver.com",
  "has_more": false
}
```

**Security**: Requires the agent's API key. Each agent can only access their own inbox.

##### Acknowledge Message

**Endpoint:** `DELETE /v1/inbox/{recipient}/{message_id}`

**Headers:**
```http
Authorization: Bearer {agent_api_key}
```

**Response:**
```json
{
  "message_id": "01H8X9Z2K3M4N5P6Q7R8S9T0",
  "status": "acknowledged",
  "timestamp": "2025-08-14T10:35:00.000Z"
}
```

**Security**: Requires the agent's API key. Agents can only acknowledge messages in their own inbox.

### 12.2 Agent Integration API

#### 12.2.1 Receive Message Callback

Agents register endpoints to receive AMTP messages:

**Endpoint:** `POST /amtp/receive` (agent-defined)

**Request:**
```json
{
  "message_id": "01H8X9Z2K3M4N5P6Q7R8S9T0",
  "sender": "agent@sender.com",
  "subject": "Process Order",
  "schema": "agntcy:commerce.order.v2",
  "payload": {
    "order_id": "12345",
    "customer": {...},
    "items": [...]
  },
  "coordination": {
    "workflow_id": "01H8X9Z2K3M4N5P6Q7R8S9T0",
    "type": "parallel",
    "requires_response": true
  }
}
```

**Response:**
```json
{
  "status": "accepted",
  "response": {
    "order_status": "confirmed",
    "estimated_delivery": "2025-08-21"
  }
}
```

---

## 13. Error Handling

### 13.1 Error Codes

#### 13.1.1 Protocol Errors

- `INVALID_MESSAGE_FORMAT`: Message doesn't conform to AMTP format
- `UNSUPPORTED_VERSION`: Protocol version not supported
- `SCHEMA_VALIDATION_FAILED`: Payload doesn't match specified schema
- `MESSAGE_TOO_LARGE`: Message exceeds size limits
- `INVALID_RECIPIENT`: Recipient address format invalid

#### 13.1.2 Delivery Errors

- `RECIPIENT_UNAVAILABLE`: Target gateway not responding
- `RECIPIENT_NOT_FOUND`: No route to recipient domain
- `AUTHENTICATION_FAILED`: Authentication rejected by recipient
- `RATE_LIMIT_EXCEEDED`: Too many messages sent
- `TEMPORARY_FAILURE`: Temporary issue, will retry

#### 13.1.3 Coordination Errors

- `WORKFLOW_TIMEOUT`: Coordination timeout exceeded
- `REQUIRED_RESPONSE_MISSING`: Required participant didn't respond
- `COORDINATION_FAILED`: Multi-agent coordination failed

### 13.2 Error Response Format

```json
{
  "error": {
    "code": "SCHEMA_VALIDATION_FAILED",
    "message": "Payload validation failed for schema agntcy:commerce.order.v2",
    "details": {
      "schema": "agntcy:commerce.order.v2",
      "validation_errors": [
        {
          "field": "order.items[0].price",
          "error": "must be a positive number"
        }
      ]
    },
    "timestamp": "2025-08-14T10:30:00.000Z",
    "request_id": "req_01H8X9Z2K3M4N5P6Q7R8S9T0"
  }
}
```

---

## 14. Compliance and Conformance

### 14.1 Conformance Levels

#### 14.1.1 Core Conformance

Implementations MUST support:
- DNS discovery mechanism (`_amtp.{domain}` TXT records)
- Core message format with mandatory fields: `version`, `sender`, `recipients`, `payload`
- HTTPS transport with TLS 1.3
- Basic message delivery (Immediate Path)
- JSON message format validation

#### 14.1.2 Extended Conformance

Implementations MAY support:

**Reliability Features:**
- Message tracking (`message_id` field)
- At-least-once delivery semantics (`idempotency_key` field)
- Durable Path with persistent storage
- Message status tracking and retry logic

**Integration Features:**
- SMTP bridging capability
- Schema validation (AGNTCY or custom frameworks)
- Agent discovery endpoints

**Advanced Features:**
- Multi-agent coordination workflows
- Digital signatures and message integrity
- End-to-end encryption
- Webhook notifications
- Custom headers and metadata

### 14.2 Testing and Validation

#### 14.2.1 Interoperability Tests

Standard test suite for validating AMTP implementations:

1. **Basic Delivery Test**: Send message between different implementations
2. **SMTP Bridging Test**: Verify SMTP bridging with AMTP
3. **Schema Validation Test**: Test schema-based message validation
4. **Coordination Test**: Multi-agent workflow execution
5. **Error Handling Test**: Proper error reporting and retry logic

#### 14.2.2 Performance Benchmarks

- Message throughput: >1000 messages/second
- Delivery latency: <100ms for same-datacenter delivery
- Storage efficiency: <1KB overhead per message
- Memory usage: <10MB per 10,000 queued messages

### 14.3 Security Considerations

#### 14.3.1 Threat Model

**Addressed Threats:**
- Message tampering (TLS, digital signatures)
- Unauthorized access (domain authentication, access policies)
- Denial of service (rate limiting, resource controls)
- Man-in-the-middle attacks (certificate validation)

**Out of Scope:**
- Advanced persistent threats
- Side-channel attacks
- Social engineering attacks
- Endpoint compromise

#### 14.3.2 Security Recommendations

1. **Use TLS 1.3** with valid certificates
2. **Implement rate limiting** to prevent abuse
3. **Validate all inputs** including message payloads
4. **Monitor and log** all activities
5. **Regular security updates** for gateway software
6. **Network segmentation** for gateway infrastructure

---

## Appendix A: Message Examples

### A.1 Minimal Core Message

```json
{
  "version": "1.0",
  "sender": "assistant@company.com",
  "recipients": ["customer-service@client.com"],
  "payload": {
    "message": "Hello, this is a minimal AMTP message"
  }
}
```

### A.2 Simple Message with Optional Fields

```json
{
  "version": "1.0",
  "message_id": "01H8X9Z2K3M4N5P6Q7R8S9T0",
  "idempotency_key": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2025-08-14T10:30:00.000Z",
  "sender": "assistant@company.com",
  "recipients": ["customer-service@client.com"],
  "subject": "Simple Notification",
  "payload": {
    "message": "Hello, this is a simple AMTP message",
    "timestamp": "2025-08-14T10:30:00.000Z"
  }
}
```

### A.3 Schema-Validated Commerce Message

```json
{
  "version": "1.0",
  "message_id": "01H8X9Z2K3M4N5P6Q7R8S9T1",
  "idempotency_key": "550e8400-e29b-41d4-a716-446655440001",
  "timestamp": "2025-08-14T11:00:00.000Z",
  "sender": "purchase-agent@retailer.com",
  "recipients": ["order-processor@supplier.com"],
  "subject": "Purchase Order PO-2025-001234",
  "schema": "agntcy:commerce.order.v2",
  "headers": {
    "priority": "high",
    "department": "procurement"
  },
  "payload": {
    "order_id": "PO-2025-001234",
    "order_date": "2025-08-14",
    "customer": {
      "company_name": "Retailer Corp",
      "contact_email": "purchasing@retailer.com",
      "billing_address": {
        "street": "123 Business Ave",
        "city": "Commerce City",
        "state": "CA",
        "zip": "90210",
        "country": "US"
      }
    },
    "items": [
      {
        "sku": "WIDGET-001",
        "description": "Premium Widget",
        "quantity": 100,
        "unit_price": 29.99,
        "total_price": 2999.00
      },
      {
        "sku": "GADGET-002", 
        "description": "Smart Gadget",
        "quantity": 50,
        "unit_price": 149.99,
        "total_price": 7499.50
      }
    ],
    "total_amount": 10498.50,
    "currency": "USD",
    "payment_terms": "NET30",
    "delivery_date": "2025-08-28"
  }
}
```

### A.4 Multi-Agent Coordination Message

```json
{
  "version": "1.0",
  "message_id": "01H8X9Z2K3M4N5P6Q7R8S9T2",
  "idempotency_key": "550e8400-e29b-41d4-a716-446655440002",
  "timestamp": "2025-08-14T12:00:00.000Z",
  "sender": "orchestrator@logistics.com",
  "recipients": [
    "warehouse@supplier.com",
    "shipping@carrier.com", 
    "tracking@delivery.com"
  ],
  "subject": "Coordinate Shipment SHIP-789",
  "schema": "agntcy:logistics.shipment.v1",
  "coordination": {
    "type": "sequential",
    "timeout": 7200,
    "sequence": [
      "warehouse@supplier.com",
      "shipping@carrier.com",
      "tracking@delivery.com"
    ],
    "stop_on_failure": true,
    "required_responses": [
      "warehouse@supplier.com",
      "shipping@carrier.com"
    ]
  },
  "payload": {
    "shipment_id": "SHIP-789",
    "order_reference": "PO-2025-001234",
    "pickup_location": {
      "address": "456 Warehouse Blvd, Industrial City, CA 90123",
      "contact": "warehouse@supplier.com",
      "hours": "08:00-17:00 PST"
    },
    "delivery_location": {
      "address": "123 Business Ave, Commerce City, CA 90210",
      "contact": "receiving@retailer.com",
      "hours": "09:00-16:00 PST"
    },
    "items": [
      {
        "sku": "WIDGET-001",
        "quantity": 100,
        "weight_kg": 50.0,
        "dimensions_cm": {
          "length": 60,
          "width": 40,
          "height": 30
        }
      }
    ],
    "service_level": "standard",
    "requested_pickup_date": "2025-08-15",
    "requested_delivery_date": "2025-08-17"
  }
}
```

### A.5 Workflow Response Message

```json
{
  "version": "1.0",
  "message_id": "01H8X9Z2K3M4N5P6Q7R8S9T3",
  "idempotency_key": "550e8400-e29b-41d4-a716-446655440003",
  "timestamp": "2025-08-14T12:15:00.000Z",
  "sender": "warehouse@supplier.com",
  "recipients": ["orchestrator@logistics.com"],
  "subject": "Re: Coordinate Shipment SHIP-789",
  "in_reply_to": "01H8X9Z2K3M4N5P6Q7R8S9T2",
  "response_type": "workflow_response",
  "schema": "agntcy:logistics.pickup_confirmation.v1",
  "payload": {
    "shipment_id": "SHIP-789",
    "status": "confirmed",
    "pickup_scheduled": "2025-08-15T10:00:00.000Z",
    "packages": [
      {
        "tracking_number": "TRK123456789",
        "weight_kg": 50.0,
        "dimensions_verified": true
      }
    ],
    "warehouse_notes": "Items ready for pickup at dock 3",
    "contact_person": "John Smith",
    "contact_phone": "+1-555-0123"
  }
}
```

## Appendix B: DNS Configuration Examples

### B.1 Basic AMTP Capability Advertisement

```dns
; Basic AMTP support
_amtp.example.com. 300 IN TXT "v=amtp1;gateway=https://amtp.example.com:443"

; With authentication and size limits
_amtp.retailer.com. 300 IN TXT "v=amtp1;gateway=https://amtp.retailer.com;auth=cert,oauth"

; Enterprise configuration with custom limits
_amtp.enterprise.com. 300 IN TXT "v=amtp1;gateway=https://amtp-gw.enterprise.com;auth=cert,oauth;max-size=50000000"
```

### B.2 Bridge MX Records

```dns
; AMTP with SMTP bridging
_amtp.hybrid.com. 300 IN TXT "v=amtp1;gateway=https://amtp.hybrid.com"
hybrid.com. 300 IN MX 10 mail.hybrid.com.
```

## Appendix C: Implementation Guidelines

### C.1 Gateway Deployment Checklist

#### C.1.1 Infrastructure Requirements

- **Load Balancer**: SSL termination, health checks
- **Application Server**: AMTP gateway implementation
- **Message Queue**: Redis/PostgreSQL for persistence  
- **Database**: Message storage and workflow state
- **Monitoring**: Metrics, logging, alerting
- **DNS**: TXT record configuration

#### C.1.2 Security Hardening

- **Network**: Firewall rules, VPC isolation
- **TLS**: Certificate management, HSTS headers
- **Authentication**: API key rotation, OAuth integration
- **Authorization**: Role-based access control
- **Audit**: Comprehensive logging and monitoring

### C.2 Agent Integration Patterns

#### C.2.1 Simple Agent

```python
import requests
import json
from datetime import datetime

class FluxAgent:
    def __init__(self, gateway_url, agent_address):
        self.gateway_url = gateway_url
        self.agent_address = agent_address
    
    def send_message(self, recipients, subject, payload, schema=None):
        message = {
            "sender": self.agent_address,
            "recipients": recipients,
            "subject": subject,
            "payload": payload,
            "timestamp": datetime.utcnow().isoformat() + "Z"
        }
        
        if schema:
            message["schema"] = schema
        
        response = requests.post(
            f"{self.gateway_url}/v1/messages",
            json=message,
            headers={"Content-Type": "application/json"}
        )
        
        return response.json()
    
    def handle_incoming_message(self, message):
        # Process incoming AMTP message
        print(f"Received: {message['subject']}")
        
        # Return response if coordination requires it
        if message.get("coordination", {}).get("requires_response"):
            return {
                "status": "processed",
                "timestamp": datetime.utcnow().isoformat() + "Z"
            }

# Usage example
agent = FluxAgent("https://amtp.mycompany.com", "ai-assistant@mycompany.com")

# Send a message
result = agent.send_message(
    recipients=["support@partner.com"],
    subject="System Alert",
    payload={"alert_type": "warning", "message": "Database CPU at 85%"},
    schema="agntcy:monitoring.alert.v1"
)

print(f"Message sent: {result['message_id']}")
```

#### C.2.2 Workflow Coordinator

```python
class WorkflowCoordinator:
    def __init__(self, amtp_agent):
        self.amtp_agent = amtp_agent
        self.active_workflows = {}
    
    def start_parallel_workflow(self, recipients, payload, timeout=3600):
        message_id = self.amtp_agent.send_message(
            recipients=recipients,
            subject="Parallel Workflow",
            payload=payload,
            coordination={
                "type": "parallel",
                "timeout": timeout,
                "required_responses": recipients
            }
        )["message_id"]
        
        self.active_workflows[message_id] = {
            "type": "parallel",
            "recipients": recipients,
            "responses": {},
            "status": "pending"
        }
        
        return message_id
    
    def handle_response(self, response_message):
        workflow_id = response_message["in_reply_to"]
        sender = response_message["sender"]
        
        if workflow_id in self.active_workflows:
            workflow = self.active_workflows[workflow_id]
            workflow["responses"][sender] = response_message["payload"]
            
            # Check if workflow is complete
            if len(workflow["responses"]) == len(workflow["recipients"]):
                workflow["status"] = "completed"
                self.on_workflow_complete(workflow_id, workflow)
    
    def on_workflow_complete(self, workflow_id, workflow):
        print(f"Workflow {workflow_id} completed with {len(workflow['responses'])} responses")
```

---

This completes the comprehensive AMTP Protocol Specification v1.0. The document provides detailed technical specifications for implementing a federated, asynchronous communication protocol that enhances email with structured data support, multi-agent coordination, and reliable delivery guarantees while maintaining universal addressing and backward compatibility.
