To effectively and safely process AI payload in Envoy, we need to be able to:
a. efficiently buffer the full HTTP payload so that the full json message is
   seen to protect against field smuggling.
a. group fields into routing related small informational fields like model name,
   token count, temperature, background processing; and the payload that can
   potentially be large.
a. the separation is analogous to HTTP headers and HTTP payload.

Looking at OpenAI's Chat / Response API, OpenAI's Completion API, Anthropic message API, and Gemini's
generateContent API, we can get a list of common fields, and map them to Envoy's
internal representation.

### 1. Core Canonical Fields
Fields supported directly across virtually all providers that map 1-to-1 to Envoy's core AI filter representation.

| OpenAI Chat / Response | OpenAI Completion | Anthropic | Gemini | AI Filter Representation | Payload Handling |
| :--------------------- | :---------------- | :-------- | :----- | :----------------------- | :--------------- |
| `model` **[REQ]** | `model` **[REQ]** | `model` **[REQ]** | `model` **[REQ]** | `model` **[REQ]** | **Bounded Metadata** |
| `input` / `prompt` / `text` **[REQ]** | `prompt` **[REQ]** | `messages` **[REQ]** | `contents` **[REQ]** | `messages` **[REQ]** | ⚠️ **Streaming / Heavy Payload** |
| `max_output_tokens` / `max_tokens` **[OPT]** | `max_tokens` **[OPT]** | `max_tokens` **[REQ]** | `maxOutputTokens` **[OPT]** | `max_tokens` **[REQ]** *(synthesized if missing)* | **Bounded Metadata** |
| `temperature` **[OPT]** | `temperature` **[OPT]** | `temperature` **[OPT]** | `temperature` **[OPT]** | `temperature` **[OPT]** | **Bounded Metadata** |
| `top_p` **[OPT]** | `top_p` **[OPT]** | `top_p` **[OPT]** | `topP` **[OPT]** | `top_p` **[OPT]** | **Bounded Metadata** |
| `stop` **[OPT]** | `stop` **[OPT]** | `stop_sequences` **[OPT]** | `stopSequences` **[OPT]** | `stop_sequences` **[OPT]** | **Bounded Metadata** |
| `stream` **[OPT]** | `stream` **[OPT]** | `stream` **[OPT]** | *(via alt endpoint)* **[OPT]** | `stream` **[OPT]** | **Bounded Metadata** |
| `tools` **[OPT]** | N/A | `tools` **[OPT]** | `tools` **[OPT]** | `tools` **[OPT]** | ⚠️ **Streaming / Heavy Payload** |
| `tool_choice` **[OPT]** | N/A | `tool_choice` **[OPT]** | `toolConfig` **[OPT]** | `tool_choice` **[OPT]** | **Bounded Metadata** |
| `metadata` **[OPT]** | N/A | `metadata` **[OPT]** | N/A | `metadata` **[OPT]** | **Bounded Metadata** |

### 2. Abstracted Functional Fields
Fields present across multiple providers that require structural transformation or generic abstraction.

| OpenAI Chat / Response | OpenAI Completion | Anthropic | Gemini | AI Filter Representation | Payload Handling |
| :--------------------- | :---------------- | :-------- | :----- | :----------------------- | :--------------- |
| `instructions` / `messages[role=system]` **[OPT]** | N/A | `system` **[OPT]** | `systemInstruction` **[OPT]** | `system_instruction` **[OPT]** | ⚠️ **Streaming / Heavy Payload** |
| `reasoning` / `reasoning_effort` **[OPT]** | N/A | `thinking` **[OPT]** | `thinkingConfig` **[OPT]** | `reasoning_config` **[OPT]** | **Bounded Metadata** |
| `response_format` / `include` **[OPT]** | N/A | `output_config` **[OPT]** | `responseMimeType` / `responseSchema` **[OPT]** | `output_config` **[OPT]** | ⚠️ **Streaming / Heavy Payload** |
| `service_tier` **[OPT]** | N/A | `service_tier` **[OPT]** | N/A | `service_tier` **[OPT]** | **Bounded Metadata** |
| `parallel_tool_calls` **[OPT]** | N/A | *(default behavior)* **[OPT]** | N/A | `parallel_tool_calls` **[OPT]** | **Bounded Metadata** |
| `stream_options` **[OPT]** | N/A | N/A | N/A | `stream_options` **[OPT]** | **Bounded Metadata** |
| `context_management` **[OPT]** | N/A | `context_management` **[OPT]** | N/A | `context_management` **[OPT]** | **Bounded Metadata** |

### 3. Vendor-Specific Extensions
Fields unique to a provider that belong in Envoy's `vendor_extensions` opaque payload bag. All fields in this section are **[OPT]** (Optional).

| OpenAI Chat / Response | OpenAI Completion | Anthropic | Gemini | AI Filter Representation | Payload Handling |
| :--------------------- | :---------------- | :-------- | :----- | :----------------------- | :--------------- |
| `prompt_cache_key`, `prompt_cache_options` **[OPT]** | N/A | `cache_control` **[OPT]** | `cachedContent` **[OPT]** | `vendor_extensions.prompt_caching` **[OPT]** | **Bounded Metadata** *(Top-level keys)* / ⚠️ **Streaming** *(Nested Anthropic `cache_control`)* |
| `moderation`, `safety_identifier` **[OPT]** | N/A | N/A | `safetySettings` **[OPT]** | `vendor_extensions.safety` **[OPT]** | **Bounded Metadata** |
| `conversation`, `previous_response_id`, `store`, `background` **[OPT]** | N/A | N/A | N/A | `vendor_extensions.state` **[OPT]** | **Bounded Metadata** |
| N/A | N/A | `speed`, `inference_geo`, `fallbacks`, `fallback_credit_token` **[OPT]** | N/A | `vendor_extensions.routing` **[OPT]** | **Bounded Metadata** |
| N/A | N/A | `mcp_servers`, `container`, `diagnostics` **[OPT]** | N/A | `vendor_extensions.environment` **[OPT]** | **Bounded Metadata** |
| `top_logprobs` **[OPT]** | `logprobs` **[OPT]** | `top_k` *(deprecated)* **[OPT]** | `topK` *(deprecated)* **[OPT]** | `vendor_extensions.logprobs` **[OPT]** | **Bounded Metadata** |
| N/A | `echo`, `best_of`, `suffix`, `logit_bias` **[OPT]** | N/A | N/A | `vendor_extensions.completion_options` **[OPT]** | **Bounded Metadata** |

---

### 4. Internal Struct / Class Design

To achieve zero-copy performance and low memory overhead, the internal representation separates **size-bounded fields** (stored by value) from **heavy/streaming payload fields** (stored as byte slices referencing the `ExternalBuffer` in [`ai_protocol_manager`](file:///usr/local/google/home/pengg/envoy/source/extensions/filters/http/ai_protocol_manager/external_buffer.h)).

```cpp
namespace Envoy {
namespace Extensions {
namespace HttpFilters {
namespace AiFilter {

// Offset & length reference to bytes stored inside Envoy's ExternalBuffer (ai_protocol_manager)
struct BufferSlice {
  uint64_t offset{0};
  uint64_t length{0};

  bool empty() const { return length == 0; }
};

struct InferenceRequestInternal {
  // =========================================================================
  // Stage 1: Size-Bounded Metadata (Direct Access Values)
  // Ceiling bounded to < 2 KB total. Enables fast-path routing & rate-limiting.
  // =========================================================================
  std::string model;                   // e.g. "gpt-4o", "claude-3-5-sonnet"
  uint32_t max_tokens{0};              // Max completion tokens (synthesized if missing)
  float temperature{1.0f};             // Sampling temperature
  float top_p{1.0f};                   // Nucleus sampling top_p
  bool stream{false};                  // Request streaming flag
  std::string service_tier;            // Service tier e.g. "priority", "standard"
  std::vector<std::string> stop_sequences;
  std::string tool_choice;             // e.g. "auto", "required", "none"
  std::string user_id;                 // End-user tracking identifier

  // =========================================================================
  // Stage 2: Heavy Payload Slices (Offsets into ai_protocol_manager ExternalBuffer)
  // Offloads memory footprint into ExternalBuffer; streamed asynchronously on demand.
  // =========================================================================
  BufferSlice system_prompt;           // System instruction slice
  std::vector<BufferSlice> messages;   // Conversation history message slices
  std::vector<BufferSlice> tools;      // Tool definition JSON schema slices
  BufferSlice response_schema;        // Structured output JSON schema slice

  // Vendor-specific extension bag (opaque key-value metadata)
  std::unordered_map<std::string, std::string> vendor_metadata;
};

} // namespace AiFilter
} // namespace HttpFilters
} // namespace Extensions
} // namespace Envoy
```

---

### 5. Two-Stage AI Filter Architecture (Generic Across LLM & MCP)

The AI Filter framework operates as a **two-stage pipeline** designed to handle both LLM Inference requests and MCP (Model Context Protocol) tool executions:

```
                          Downstream HTTP Request
                                     │
                                     ▼
      ┌─────────────────────────────────────────────────────────────┐
      │  Stage 1: Size-Bounded Metadata Stage                       │
      │  - Fast-path header/metadata parsing (< 2 KB)               │
      │  - Evaluates `model`, `stream`, `service_tier`, `tokens`     │
      │  - Fast route matching, rate limiting & target selection    │
      └──────────────────────────────┬──────────────────────────────┘
                                     │
                                     ▼
      ┌─────────────────────────────────────────────────────────────┐
      │  Offload to ai_protocol_manager::ExternalBuffer             │
      │  - Payloads stored off-heap / stream-buffered               │
      │  - Saves (offset, length) slices in `InferenceRequestInternal`│
      └──────────────────────────────┬──────────────────────────────┘
                                     │
                                     ▼
      ┌─────────────────────────────────────────────────────────────┐
      │  Stage 2: Streaming Payload Filter Stage                    │
      │  - Runtime Field Selection Interface:                       │
      │    Filters express required fields dynamically:             │
      │    • "Subscribe: system_prompt" (LLM Guardrails)            │
      │    • "Subscribe: tool_arguments" (MCP Tool Filters)         │
      │    • "Subscribe: full_messages" (Field Smuggling Filter)    │
      │  - Async byte streaming via `ExternalBuffer::read(off, len)`│
      └─────────────────────────────────────────────────────────────┘
```

#### Key Advantages:
1. **Zero Memory Bloat**: Heavy prompts, base64 images, and large tool schemas remain offloaded in `ai_protocol_manager`'s `ExternalBuffer` until explicitly requested.
2. **Dynamic Runtime Field Selection**: Individual Envoy sub-filters (e.g. prompt safety scanners, MCP authorization filters, protocol translation adapters) declare which payload slices they need (`"Give me system prompt if LLM"`, `"Give me tool parameters if MCP"`).
3. **Genericity Across AI Protocols**: The exact same two-stage pipeline supports LLM inference endpoints (`/v1/chat/completions`, `/v1/messages`, `generateContent`) and MCP JSON-RPC tool calls (`tools/call`, `resources/read`).

---

### 6. Middle Ground: Stream-Field Interest Registration & Order Dependency Scheduling

This middle-ground approach strikes the ideal balance between developer ergonomics and Envoy Filter Manager scheduling efficiency:

#### Rule 1: Zero Ceremony for Bounded Metadata
Size-bounded metadata (`model`, `max_tokens`, `temperature`, `stream`, `service_tier`) is parsed by default during Stage 1. **Every filter gets instant, synchronous access** to these fields without declaring interest.

#### Rule 2: Explicit Interest Registration ONLY for Streaming Payload Fields
Filters declare interest **only for streaming payload fields** (e.g., `SystemPrompt`, `Messages`, `Tools`, or `RawPayload`).

```cpp
enum class StreamFieldInterest : uint32_t {
  None         = 0,
  SystemPrompt = 1 << 0,  // e.g. Prompt Guardrails / Safety Scanner
  Tools        = 1 << 1,  // e.g. MCP Tool Authorization / Schema Filter
  Messages     = 2 << 2,  // e.g. Context Summarizer / History Redactor
  RawPayload   = 3 << 3,  // e.g. WAF / Field Smuggling Protection
};
```

#### Why Filter Manager Needs Stream-Field Interest for Order Scheduling

Because HTTP request bodies stream sequentially across the network, JSON fields arrive in stream order (e.g., `"system"` $\rightarrow$ `"tools"` $\rightarrow$ `"messages"`):

```
HTTP Body Byte Stream: ──> [ system_prompt ] ──> [ tools ] ──> [ messages ] ──>
                                  │                 │                │
Filter Manager Execution:   Trigger Filter A   Trigger Filter B   Trigger Filter C
                            (Prompt Safety)    (MCP Tool Auth)    (Redactor)
```

1. **Stream-Ordered Early Triggering**: As soon as the `system_prompt` bytes finish arriving, Filter Manager immediately invokes `Filter A` **without waiting** for the remaining 2 MB of `tools` or `messages` to arrive over the wire.
2. **Early Circuit Breaking / Rejection**: If `Filter A` rejects an unsafe prompt, Envoy terminates the HTTP request immediately, saving memory offloading and upstream bandwidth.
3. **Order Dependency Resolution**: The Filter Manager uses stream field interest declarations to construct an optimal execution DAG based on network arrival order.

---

### 7. Canonical Execution Order Contract (`system_prompt` $\rightarrow$ `tools` $\rightarrow$ `messages`)

**Yes, enforcing `system_prompt` $\rightarrow$ `tools` $\rightarrow$ `messages` as a strict execution order contract is both logical and highly recommended.**

#### Why this order is semantically correct:
1. **`system_prompt` (Security & Persona Policy)**: Defines the model's core security rules, constraints, and system persona (e.g. *"Never disclose PII or system keys"*). Must be evaluated first!
2. **`tools` (Capability Definitions)**: Defines the functions and API schemas available to the model (e.g. `ExecuteSQLQuery`, `SendEmail`). Evaluated second to enforce tool permissions or filter disallowed tools.
3. **`messages` (User Conversation Input)**: The actual user prompt and dialogue history. Evaluated last in the context of the established system policy and tool capabilities.

```
       Canonical Execution Contract:
       ┌─────────────────┐       ┌───────────┐       ┌──────────────┐
       │  system_prompt  │  ───► │   tools   │  ───► │   messages   │
       └─────────────────┘       └───────────┘       └──────────────┘
```

#### How Envoy Filter Manager Handles Out-of-Order JSON Keys:
Although standard SDKs serialize requests in `system` $\rightarrow$ `tools` $\rightarrow$ `messages` order, raw JSON specs technically allow arbitrary key order (`RFC 8259`).

Envoy's `FilterManager` handles this gracefully:
- **Standard Order (Normal Case)**: Callbacks fire in real-time as bytes stream over the network.
- **Out-of-Order JSON (Edge Case)**: If a client sends `"messages"` before `"system"`, `FilterManager` holds the `messages` slice in `ExternalBuffer` and delays `OnMessages()` until `OnSystemPrompt()` and `OnTools()` have completed.

This guarantees that **filter developers can write simple, deterministic code** relying on the `system_prompt` $\rightarrow$ `tools` $\rightarrow$ `messages` pipeline contract.

---

### 8. Coroutine-Based AI Filter Interface (C++20 Sequential Paradigm)

Building on the C++20 coroutine framework described in [`20260709_COROUTINE.md`](file:///usr/local/google/home/pengg/thoughts/20260709_COROUTINE.md), we can express the entire AI Filter pipeline as a **sequential, callback-free coroutine**.

#### Coroutine AI Filter Code Example:

```cpp
namespace Envoy {
namespace Extensions {
namespace HttpFilters {
namespace AiFilter {

AiFilterCoroutine decode(InferenceRequestMetadata metadata,
                         SystemPromptGetter get_system_prompt,
                         TerminatingActions terminating_actions,
                         RequestInfo request_info) {
  // 1. Stage 1: Size-Bounded Metadata is available synchronously immediately
  if (metadata.model == "untrusted-model" || metadata.temperature > 1.8f) {
    co_return terminating_actions.replyLocally(Http::Code::BadRequest);
  }

  // 2. Stage 2 Step A: Await System Prompt (Triggers async read off ExternalBuffer)
  ToolsGetter get_tools;
  if (get_system_prompt.has_value()) {
    BufferSlice system_slice;
    ASSIGN_OR_CO_RETURN(std::tie(system_slice, get_tools),
                        co_await std::move(get_system_prompt)());

    if (!system_slice.empty()) {
      absl::string_view system_text = co_await request_info.readSlice(system_slice);
      if (isSystemPromptUnsafe(system_text)) {
        co_return terminating_actions.replyLocally(Http::Code::Forbidden);
      }
    }
  }

  // 3. Stage 2 Step B: Await Tools (Enforces type-safe transition to Tools phase)
  MessagesGetter get_messages;
  if (get_tools.has_value()) {
    std::vector<BufferSlice> tool_slices;
    ASSIGN_OR_CO_RETURN(std::tie(tool_slices, get_messages),
                        co_await std::move(get_tools)());

    for (const auto& tool_slice : tool_slices) {
      absl::string_view tool_json = co_await request_info.readSlice(tool_slice);
      if (isUnauthorizedTool(tool_json)) {
        co_return terminating_actions.replyLocally(Http::Code::Forbidden);
      }
    }
  }

  // 4. Stage 2 Step C: Await Messages (Final conversation history inspection)
  if (get_messages.has_value()) {
    std::vector<BufferSlice> message_slices;
    ASSIGN_OR_CO_RETURN(message_slices, co_await std::move(get_messages)());

    // Inspect or redact conversation message slices
  }

  // Filter completed successfully; pass remaining payload through
  co_return PostDecodeAction::kSkip;
}

} // namespace AiFilter
} // namespace HttpFilters
} // namespace Extensions
} // namespace Envoy
```

#### Why Coroutines Excel for AI Filters:

1. **Compiler-Enforced State Transitions**:
   By using rvalue ref-qualifiers (`operator()() &&`) and Clang typestate attributes (`[[clang::consumable]]`), `get_system_prompt()` yields `ToolsGetter`, which in turn yields `MessagesGetter`. Calling steps out of sequence becomes a **compile-time error**.
2. **Eliminates Callback Hell & Re-entrancy**:
   Security guardrails, tool checkers, and prompt redactors are written as a single, readable sequential function. Local variables maintain state across `co_await` suspend points without messy class member state.
3. **Lazy Zero-Cost Execution**:
   If a coroutine filter never calls `co_await std::move(get_tools)()`, Envoy's `FilterManager` skips `ExternalBuffer::read()` for `tools`, achieving **zero I/O overhead** for unexamined fields.

---

### 9. Genericity Across LLM Inference & MCP (Model Context Protocol)

The pipeline is **100% generic across both LLM Inference APIs and MCP (Model Context Protocol) JSON-RPC requests**. 

By mapping the 2-stage coroutine pipeline to abstract payload concepts, identical security, PII redactor, authorization, and rate-limiting filters can be shared seamlessly across LLM inference endpoints (`/v1/chat/completions`, `/v1/messages`) and MCP tool servers (`tools/call`, `prompts/get`, `resources/read`).

#### Structural Mapping: LLM vs. MCP

| Pipeline Stage | Abstract Concept | LLM Inference API | MCP (Model Context Protocol) |
| :--- | :--- | :--- | :--- |
| **Stage 1 (Metadata)** | **Identifier & Target** | `model` ("gpt-4o", "claude-3-5") | `method` ("tools/call") + `tool_name` ("sql_query") |
| **Stage 2 Step A** | **System Instruction / Template** | `system_prompt` | `prompt_template` (MCP `prompts/get`) |
| **Stage 2 Step B** | **Capability Schemas** | `tools` (JSON schemas) | `tool_schema` (MCP `tools/list`) |
| **Stage 2 Step C** | **Input Payload / Arguments** | `messages` (conversation history) | `params.arguments` (JSON tool call parameters) |

---

#### Filter Reusability Matrix (Shared Filters across LLM & MCP)

```
                            Envoy AI Filter Framework
                                       │
            ┌──────────────────────────┴──────────────────────────┐
            ▼                                                     ▼
    LLM Inference Pipeline                                MCP Tool Call Pipeline
    (/v1/chat/completions)                                  (JSON-RPC /tools/call)
            │                                                     │
            ├─────────────────────────────────────────────────────┤
            │        Shared Generic Coroutine Sub-Filters         │
            ├─────────────────────────────────────────────────────┤
            │  1. PII / Sensitive Data Redactor                   │
            │     - Scans `messages` (LLM) or `arguments` (MCP)   │
            │  2. Prompt Injection Guardrail Filter               │
            │     - Scans `system_prompt` (LLM) or `arguments` (MCP)│
            │  3. RBAC / Authorization Filter                     │
            │     - Authorizes `model` (LLM) or `tool_name` (MCP) │
            │  4. Rate Limiting & Token Budget Filter             │
            │     - Tracks usage per tenant / `user_id`           │
            └─────────────────────────────────────────────────────┘
```

#### Generic C++ Coroutine Filter Interface:

```cpp
namespace Envoy {
namespace Extensions {
namespace HttpFilters {
namespace AiFilter {

// Generic payload stage handles shared by both LLM and MCP protocols
using ContextGetter = SystemPromptGetter;   // LLM system_prompt  <-> MCP prompt_template
using SchemaGetter  = ToolsGetter;          // LLM tools          <-> MCP tool_schema
using PayloadGetter = MessagesGetter;       // LLM messages       <-> MCP params.arguments

// A single filter written once, running on both LLM inference and MCP tool calls!
AiFilterCoroutine genericSecurityFilter(AiRequestMetadata metadata,
                                        ContextGetter get_context,
                                        TerminatingActions actions,
                                        RequestInfo info) {
  // 1. Authorize user against target (model name for LLM, or tool name for MCP)
  if (!info.rbac().isAuthorized(metadata.target_name(), metadata.user_id())) {
    co_return actions.replyLocally(Http::Code::Forbidden);
  }

  // 2. Inspect payload content for PII / Prompt Injection
  SchemaGetter get_schemas;
  if (get_context.has_value()) {
    BufferSlice context_slice;
    ASSIGN_OR_CO_RETURN(std::tie(context_slice, get_schemas),
                        co_await std::move(get_context)());
    ...
  }

  co_return PostDecodeAction::kSkip;
}

} // namespace AiFilter
} // namespace HttpFilters
} // namespace Extensions
} // namespace Envoy
```

#### Key Takeaway:
Because MCP JSON-RPC requests (`tools/call`, `prompts/get`) share the exact same structural paradigm—**Metadata + Instruction + Schemas + Arguments**—filters written for Envoy AI proxy are **100% reusable for MCP tool execution proxies**.
