# Provider-Agnostic Single-File Chat UX Specification

## 1. Purpose

This document specifies a reusable, ChatGPT-like browser UX derived from the
llama.cpp WebUI while separating presentation and conversation management from
LLM inference. The distributable application is one self-contained HTML file
that any project can serve and connect to its own chat system.

The first non-llama integration target is
[`japer-technology/ubuntu-zombie`](https://github.com/japer-technology/ubuntu-zombie).
The design also incorporates lessons from
[`NandaIda/standalone_llamacpp_webui`](https://github.com/NandaIda/standalone_llamacpp_webui),
which independently extracted the llama.cpp WebUI into a static client for
OpenAI-compatible providers.

Normative terms **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY**
are used as defined by RFC 2119.

## 2. Goals

The application MUST:

1. Ship as a single HTML file containing all required HTML, CSS, JavaScript,
   icons, fonts, and workers.
2. Run as a static browser application without a Node.js, Python, llama.cpp, or
   application-specific runtime in the browser.
3. Keep the chat UX independent of endpoint paths, request bodies,
   authentication, response schemas, streaming protocols, model parameters,
   tool execution, and inference providers.
4. Support both request/response and streaming chat systems.
5. Preserve the useful llama.cpp UX: conversation history, responsive layout,
   message editing and branching, Markdown, code blocks, copy actions,
   attachments, cancellation, themes, and accessible controls.
6. Allow a host to select only the capabilities its backend supports.
7. Work offline after the HTML file has loaded, except for calls intentionally
   made to the configured chat backend.
8. Support same-origin, cookie-authenticated systems such as Ubuntu Zombie
   without requiring browser-stored API credentials.

The application SHOULD:

- use a stable, documented host API rather than requiring a fork;
- retain conversations across reloads when local persistence is enabled;
- remain useful when JavaScript APIs such as `EventSource`, IndexedDB, or the
  Clipboard API are unavailable;
- support right-to-left and mixed-direction content;
- provide a production build small enough to embed in a native server binary.

## 3. Non-goals

The reusable UX is not responsible for:

- loading or running an LLM;
- applying model chat templates;
- tokenization or context-window policy;
- selecting sampling defaults for a provider;
- deciding whether tools are safe to execute;
- executing privileged actions;
- storing server-side conversation or audit history;
- translating every provider protocol in the presentation layer;
- exposing provider secrets to a static public deployment.

Provider-specific settings, llama.cpp server properties, token-per-second
statistics, Python execution, MCP, and tool approval are optional capabilities,
not assumptions in the core UX.

## 4. Source Assessment

### 4.1 Current llama.cpp WebUI

The reusable behavior is presently spread across
`tools/server/webui/src/`:

| Concern | Current source | Extraction implication |
| --- | --- | --- |
| Layout and routing | `App.tsx` | Keep the layout; make routing independent of server paths. |
| Turn orchestration | `utils/app.context.tsx` | Split UI state from llama.cpp fetch and SSE handling. |
| Conversation model | `utils/types.ts` | Retain the message tree, but remove llama-specific API types. |
| Persistence | `utils/storage.ts` | Put behind a storage interface; IndexedDB remains the default local implementation. |
| Transcript and composer | `components/ChatScreen.tsx` | Retain as presentation driven by controller state and declared capabilities. |
| Message actions | `components/ChatMessage.tsx` | Retain copy, edit, regenerate, and branch navigation. |
| Rich text | `components/MarkdownDisplay.tsx` | Retain GFM, code highlighting, math, and copy; enforce sanitization. |
| Attachments | `components/useChatExtraContext.tsx` | Retain file preparation separately from provider serialization. |
| Settings | `Config.ts`, `components/SettingDialog.tsx` | Replace llama sampling fields with host-provided setting definitions. |
| Build | `tools/server/webui/vite.config.ts` | Retain single-file bundling, deterministic gzip, and a size limit. |

The principal coupling is in `utils/app.context.tsx`: it builds llama.cpp
sampling parameters, posts to `/v1/chat/completions`, assumes OpenAI-compatible
SSE chunks, fetches `/props`, and interprets llama.cpp timing fields. These
operations belong in adapters.

The source already proves that a rich application can be bundled into one HTML
artifact using `vite-plugin-singlefile`. The generated artifact is build
output; maintainable source MUST remain modular.

### 4.2 `standalone_llamacpp_webui` precedent

`standalone_llamacpp_webui` demonstrates that the llama.cpp UX can become a
static, browser-only client with configurable API base URL, API key, model,
multiple OpenAI-compatible services, IndexedDB history, attachments, Android
packaging, MCP, and agentic tool loops.

The specification adopts these lessons:

- backend address and model identity must be runtime configuration;
- browser-local settings and history can work without a backend database;
- static deployment and mobile wrapping are compatible with the UX;
- tool and provider features should be capability-driven.

It does not treat that project as the final abstraction. Its public purpose is
still OpenAI-compatible API access, its source requires a Svelte build and many
packages, and advanced inference/tool behavior remains inside the client. This
specification instead places all wire-protocol knowledge behind a transport
adapter and requires one portable HTML artifact.

### 4.3 Ubuntu Zombie integration

Ubuntu Zombie already serves a dependency-free single-file template at
`payload/agent/templates/index.html`. Its backend is not a normal OpenAI chat
endpoint:

- `POST /api/message` starts a turn;
- a returned opaque `turn_id` is consumed through
  `GET /api/stream/{turn_id}`;
- the stream emits `phase`, `token`, `tool_start`, `tool_end`,
  `pending_approval`, `turn_done`, and `turn_error`;
- cookie-authenticated `EventSource` is the preferred transport;
- synchronous JSON and conversation reload are fallback paths;
- closing the stream does not cancel server-side work;
- tool approval, lifecycle/TTL, commands, queueing, audit visibility, and
  authoritative server-side history are product requirements.

These semantics MUST be implemented by an Ubuntu Zombie adapter. They MUST NOT
be encoded into generic transcript components.

## 5. Deliverables

An implementation conforming to this specification MUST produce:

1. A maintainable modular source tree.
2. `chat-ux.html`, the uncompressed standalone application.
3. `chat-ux.html.gz`, a deterministic gzip artifact when native embedding is
   required.
4. A transport adapter API and type definitions.
5. A generic fetch/stream adapter.
6. A llama.cpp OpenAI-compatible adapter.
7. An Ubuntu Zombie adapter.
8. An example host configuration for each adapter.
9. Automated contract, component, accessibility, security, and browser tests.
10. A license and attribution notice covering reused llama.cpp material and
    bundled third-party assets.

The source filename and output location MAY be customized by a consuming
project, but the artifact MUST remain a single HTML file.

## 6. Architecture

### 6.1 Required layers

The implementation MUST enforce these dependency directions:

1. **View layer** — renders state and emits semantic user intents.
2. **Chat controller** — owns the UX state machine and coordinates services.
3. **Conversation domain** — provider-neutral conversations, branches,
   messages, attachments, and status events.
4. **Transport interface** — starts, observes, resumes, and optionally cancels
   turns.
5. **Transport adapters** — translate one backend protocol into the transport
   interface.
6. **Storage interface** — optional local or host-owned persistence.
7. **Host configuration** — branding, capabilities, settings, policy, and
   adapter selection.

The view and controller MUST NOT call `fetch`, `EventSource`, WebSocket, or
provider SDKs directly. They MUST NOT inspect provider response JSON.

### 6.2 Mounting and configuration

The artifact MUST support:

- automatic full-page mounting into a documented root element;
- configuration supplied before startup through a documented global object;
- programmatic mounting for embedding into an existing page;
- a host-supplied transport object;
- a declarative built-in adapter configuration for deployments that cannot
  inject JavaScript functions.

Configuration MUST distinguish:

| Group | Examples |
| --- | --- |
| Branding | title, logo data URI, assistant label, accent colors |
| Capabilities | attachments, branching, regeneration, cancellation, tools, approvals |
| Transport | adapter ID, same-origin base URL, endpoint map, stream mode |
| Authentication | cookie, bearer, custom headers, host callback |
| Persistence | none, local, or host adapter |
| Presentation | theme, transcript width, welcome content, composer placeholder |
| Backend settings | host-declared fields and validation rules |

Unknown configuration keys SHOULD produce a diagnostic warning and MUST NOT
silently change behavior.

### 6.3 Capability negotiation

The transport MUST expose capabilities before interactive controls are shown.
Capabilities include:

- streaming;
- cancellation and whether cancellation stops server-side work;
- server-authoritative conversations;
- local persistence;
- branching, editing, regeneration, and deletion;
- attachment MIME types, count, and size limits;
- tool activity and approval;
- reasoning display;
- usage statistics;
- model selection;
- custom commands;
- resumable turns.

Unsupported controls MUST be hidden or disabled with an explanation. The UX
MUST NOT infer support from a provider name.

Hosts SHOULD declare static baseline capabilities in configuration. If dynamic
initialization fails or times out, the application MUST still render an
offline/degraded shell using those baseline capabilities, retain drafts, and
offer reconnection. Controls that depend on unconfirmed backend capabilities
MUST remain disabled until negotiation succeeds.

## 7. Provider-Neutral Domain Model

### 7.1 Conversation

A conversation MUST have:

- stable string `id`;
- display `title`;
- creation and modification timestamps;
- current branch leaf ID;
- optional backend metadata treated as opaque;
- optional persistence ownership (`local` or `server`).

### 7.2 Message

A message MUST have:

- stable string `id`;
- conversation ID;
- role: `system`, `user`, `assistant`, or `tool`;
- content represented as ordered content parts;
- creation timestamp;
- status: `queued`, `sending`, `streaming`, `complete`, `cancelled`, or `error`;
- optional parent and child IDs for branching;
- optional attachments;
- optional tool activity, reasoning, usage, and opaque backend metadata.

IDs MUST NOT be generated solely from `Date.now()`. The implementation MUST use
collision-resistant IDs or server-provided IDs.

### 7.3 Content parts

The core model MUST support:

- plain text;
- Markdown text;
- image references;
- audio references;
- attached text/file references;
- tool call and tool result summaries;
- status/progress information.

Adapters MAY carry additional opaque parts, but the core renderer MUST use a
safe fallback for unknown types.

### 7.4 Attachments

An attachment MUST describe name, media type, size, source, and preview
metadata independently of provider serialization. A source MAY be a browser
`File`, object URL, data URL, opaque server ID, or remote URL permitted by host
policy.

Base64 conversion, PDF extraction, and image conversion are preparation
services. Converting an attachment into OpenAI `image_url`,
`input_audio`, or text content is adapter behavior.

## 8. Transport Contract

### 8.1 Operations

Every transport MUST implement:

| Operation | Requirement |
| --- | --- |
| `initialize` | Validate configuration and return capabilities. |
| `startTurn` | Accept a provider-neutral turn request and return a turn handle. |
| `observeTurn` | Deliver normalized events until terminal state. |
| `reattachTurn` | Optional; resume observation of a backend turn by opaque turn ID. |
| `cancelTurn` | Optional; return whether local observation or backend work was cancelled. |
| `respondToApproval` | Optional; submit allow/deny for a specific approval request. |
| `loadConversation` | Required when the server owns authoritative history. |
| `listConversations` | Optional for server-owned history. |
| `mutateConversation` | Optional rename/delete/edit/branch operations. |
| `dispose` | Close streams, release listeners, and clear transient secrets. |

Each turn MUST have an opaque stable ID and an `AbortSignal` for client-side
resource cleanup. An approval event MUST carry an opaque decision ID, safe
summary, allowed decisions, and expiry/status metadata. The shared approval UI
MUST call `respondToApproval`; classification and execution policy remain host
responsibilities.

### 8.2 Normalized events

Adapters MUST normalize backend activity into this event vocabulary:

| Event | Meaning |
| --- | --- |
| `accepted` | Backend accepted the turn and may supply canonical IDs. |
| `phase` | Human-readable coarse status without transcript content. |
| `content_delta` | Append-only assistant content. |
| `content_replace` | Replace provisional content with authoritative content. |
| `reasoning_delta` | Optional separately rendered reasoning content. |
| `tool_start` | A named tool/action began. |
| `tool_end` | A tool/action completed, failed, or was denied. |
| `approval_required` | The host requires an explicit user decision. |
| `usage` | Provider-neutral or namespaced usage metrics. |
| `completed` | Terminal success with authoritative messages or reload hint. |
| `cancelled` | Terminal cancellation. |
| `error` | Recoverable or terminal failure with safe user-facing text. |

Events MUST include monotonic sequence numbers when the backend supplies them.
Duplicate events MUST be safe to ignore. Adapters SHOULD support replay or an
authoritative reload after disconnect.

### 8.3 Generic OpenAI-compatible adapter

The built-in OpenAI-compatible adapter MUST:

- make base URL, endpoint, model, headers, and request options configurable;
- serialize provider-neutral content parts explicitly;
- support non-streaming JSON and `data:` SSE responses;
- handle `[DONE]`, fragmented UTF-8, fragmented lines, empty deltas, usage-only
  chunks, non-2xx responses, and malformed events;
- propagate cancellation through `AbortController`;
- avoid llama.cpp-only fields unless the host explicitly configures them;
- namespace non-standard fields rather than adding them to the core model.

Direct browser access to third-party services MUST be opt-in because it exposes
credentials to the browser and depends on provider CORS policy.

### 8.4 Ubuntu Zombie adapter

The Ubuntu Zombie adapter MUST:

1. Use same-origin session cookies and never request or persist the provider API
   key.
2. Start streaming turns through `POST /api/message`.
3. Open `EventSource` on the returned turn ID.
4. Map Ubuntu Zombie events to the normalized event vocabulary.
5. Render tool activity and approvals as first-class status, not assistant
   Markdown.
6. Keep approval decisions on Ubuntu Zombie’s existing authenticated APIs.
7. Treat the final server payload and reloaded conversation as authoritative.
8. Reattach to a live turn by opaque turn ID when supported, otherwise recover
   from dropped streams by reloading history.
9. Clearly state when closing a stream only stops local observation and does
   not cancel server-side work.
10. Preserve the existing one-message queue, slash-command discovery, TTL
    tombstone/login states, rebranding, transcript-width preference, and
    provider/model status.

The generic UX MUST expose extension slots for these controls without knowing
their endpoint names or policy semantics.

## 9. UX Requirements

### 9.1 Application shell

Desktop layout SHOULD provide a conversation sidebar, header, transcript, and
sticky composer. On narrow screens, the sidebar MUST become a dismissible
drawer and the transcript MUST retain the full usable width.

The host MUST be able to replace llama.cpp branding without rebuilding.

### 9.2 Conversation management

When supported, users MUST be able to:

- create, select, rename, delete, and export conversations;
- see conversations ordered and grouped by recency;
- edit user messages and create a branch;
- regenerate assistant responses;
- navigate sibling branches without losing prior versions.

Local and server-authoritative history MUST not be merged implicitly. The
selected persistence mode MUST identify one authority and define reconciliation
behavior.

### 9.3 Composer

The composer MUST:

- submit on Enter and insert a newline on Shift+Enter;
- respect IME composition;
- provide an accessible send button;
- replace send with stop/cancel only when cancellation is meaningful;
- preserve an unsent draft after recoverable failure;
- show attachment limits before reading large files;
- support paste and drag/drop where enabled;
- display queued state independently from active generation;
- allow host-provided command completion.

### 9.4 Transcript

The transcript MUST:

- visually distinguish user, assistant, system, tool, queued, and error states;
- display provisional streaming text without repeatedly resetting selection or
  scroll position;
- auto-scroll only while the user remains near the bottom;
- preserve the user’s scroll position when reading older content;
- provide copy controls for messages and code blocks;
- show clear retry/reconnect actions for recoverable errors;
- represent tool progress and approval separately from model prose.

### 9.5 Markdown and code

The renderer SHOULD support CommonMark, GitHub Flavored Markdown, fenced code,
syntax highlighting, tables, soft line breaks, and optional math.

Raw HTML from messages MUST be disabled or sanitized with an allowlist. Links
MUST use safe protocols; external links opened in a new context MUST use
`rel="noopener noreferrer"`. The renderer MUST NOT execute scripts, event
handlers, `javascript:` URLs, SVG scripts, or active content from messages.

Code execution MUST NOT be part of the default renderer. A host that enables it
MUST provide a separately permissioned sandbox.

### 9.6 Reasoning

Reasoning content MAY be shown in a collapsed region when the adapter supplies
it as a separate normalized field. The generic renderer MUST NOT depend on
`<think>` tags. A compatibility parser MAY be provided by an adapter.

### 9.7 Settings

Core settings SHOULD be limited to presentation, persistence, and privacy.
Backend settings MUST be declared by the host as typed fields with defaults,
validation, help text, sensitivity, and persistence policy.

API keys and other secrets MUST be marked sensitive. Cookie-authenticated
applications MUST NOT display irrelevant API-key fields.

## 10. Persistence

The default local storage adapter SHOULD use:

- IndexedDB for conversations, messages, branches, and attachment metadata;
- localStorage only for small, non-sensitive preferences;
- an explicit schema version and transactional migrations.

The application MUST remain functional with persistence disabled. It MUST
handle denied storage, quota exhaustion, corrupt records, and private browsing
without losing the active in-memory turn.

Secrets SHOULD remain in memory. If a deployment deliberately persists a
browser-side API key, the UI MUST explain that any user or script with access
to that browser origin can retrieve it.

Conversation export MUST include a format version. Import MUST validate size,
shape, IDs, parent relationships, and content types before writing data.

## 11. Security and Privacy

The implementation MUST:

- render all untrusted text escape-first;
- sanitize rich output;
- validate adapter configuration and backend payloads at runtime;
- enforce attachment count and byte limits before conversion;
- avoid logging prompts, responses, credentials, attachment content, or
  authorization headers by default;
- redact sensitive values from diagnostics;
- prevent prototype-pollution keys in imported/configuration objects;
- use `textContent` for status and tool output unless sanitized Markdown is
  explicitly intended;
- revoke object URLs when no longer needed;
- abort readers and detach stream listeners on navigation/disposal;
- avoid third-party CDNs, analytics, fonts, and telemetry in the artifact;
- document the origin, CORS, cookie, and CSRF assumptions of each adapter.

For same-origin cookie authentication, state-changing endpoints MUST retain the
host application’s CSRF protections. `EventSource` cannot set arbitrary
authorization headers, so bearer-authenticated SSE requires a secure
host-specific mechanism and MUST NOT place long-lived secrets in URLs.

A single-file build necessarily contains inline script and style. Deployments
with a strict Content Security Policy MUST use an artifact hash, server nonce
injection, or an explicitly documented CSP policy. The implementation MUST NOT
weaken the host CSP silently.

## 12. Accessibility and Internationalization

The application MUST target WCAG 2.2 AA and provide:

- semantic landmarks and heading order;
- visible keyboard focus;
- keyboard access to every action;
- correctly associated names, descriptions, labels, and dialog controls;
- focus trapping and restoration for modal dialogs and mobile drawers;
- `aria-live` announcements for status changes without announcing every token;
- minimum contrast ratios in all bundled themes;
- reduced-motion support;
- screen-reader text for icon-only controls;
- direction-aware message content;
- locale-aware dates and conversation grouping;
- no reliance on color, hover, or pointer input alone.

## 13. Build and Artifact Requirements

The source MAY use a framework, but the production artifact MUST:

- contain no external runtime imports or asset requests;
- work when served from an arbitrary path and filename;
- avoid absolute application URLs;
- use hash routing or host-controlled routing where routing is needed;
- be reproducible from the lockfile and documented toolchain;
- produce deterministic gzip output;
- include a generated-file warning and source reference;
- define and enforce compressed and uncompressed size budgets;
- include all worker code without creating undeclared network dependencies.

The current llama.cpp compressed limit of 2 MiB is a useful upper bound.
Implementations SHOULD define a smaller core budget and make heavyweight PDF,
math, syntax-language, MCP, or code-execution features optional at build time.

## 14. Error Handling and Recovery

The controller MUST model at least these states:

`idle`, `submitting`, `queued`, `connecting`, `streaming`,
`awaiting_approval`, `reconnecting`, `completed`, `cancelled`, and `failed`.

It MUST:

- prevent duplicate submission caused by repeated input events;
- distinguish validation, authentication, rate-limit, network, protocol,
  backend, and cancellation failures;
- retain partial output after interruption;
- avoid persisting provisional assistant content as final;
- allow adapters to replace provisional output with authoritative output;
- retry only idempotent operations automatically;
- use bounded reconnect attempts with visible status;
- restore the composer draft when a submission was not accepted;
- reload server-authoritative state when completion status is uncertain.

## 15. Testing and Acceptance Criteria

### 15.1 Contract tests

Every adapter MUST pass the same transport suite covering:

- successful synchronous and streaming turns;
- fragmented multibyte text and stream frames;
- empty and malformed events;
- duplicate and out-of-order events;
- non-2xx and structured backend errors;
- cancellation before acceptance and during streaming;
- disconnect, reconnect, replay, and authoritative reload;
- approval allow/deny, duplicate decisions, expiry, and authorization failure;
- reattachment to live turns and explicit reload-only fallback;
- unsupported capability behavior;
- disposal without leaked requests or listeners.

### 15.2 UX tests

Automated browser tests MUST verify:

- new and existing conversation flows;
- send, queue, stop, retry, edit, regenerate, and branch navigation;
- draft restoration;
- near-bottom auto-scroll behavior;
- Markdown and code rendering;
- safe handling of hostile Markdown and URLs;
- attachment limits and previews;
- local persistence failure and migration;
- keyboard-only operation;
- mobile drawer and responsive composer;
- theme contrast and reduced motion.

### 15.3 Integration tests

At minimum, release validation MUST cover:

1. llama.cpp `/v1/chat/completions`, streamed and non-streamed;
2. a mock non-OpenAI custom transport;
3. Ubuntu Zombie turn creation, SSE phases/tokens/tools, approval, completion,
   dropped-stream recovery, queueing, authentication, and TTL expiry;
4. a direct static deployment with no backend-specific asset server;
5. Chromium and Firefox, with Safari/WebKit included where practical.

### 15.4 Definition of done

The work is complete when:

- changing backend protocols requires only an adapter;
- the view and controller contain no llama.cpp or Ubuntu Zombie endpoint names;
- the same HTML artifact runs against both reference adapters through runtime
  configuration;
- disabling optional capabilities removes unusable controls;
- no runtime request is made for application assets;
- compressed and uncompressed size budgets are enforced;
- repeated builds produce byte-identical HTML and gzip artifacts;
- documented strict-CSP hash or nonce deployment succeeds without weakening
  unrelated host policy;
- security and accessibility tests pass;
- license notices are present;
- documentation explains how a third project implements and registers an
  adapter without modifying the core.

## 16. Migration Sequence

1. Freeze existing llama.cpp UX behavior with component and browser tests.
2. Extract provider-neutral domain types from `utils/types.ts`.
3. Move fetch, SSE parsing, `/props`, sampling, and OpenAI serialization out of
   `utils/app.context.tsx` into a llama.cpp adapter.
4. Introduce the chat controller and normalized event state machine.
5. Put IndexedDB and preferences behind the storage interface.
6. Convert settings and visible controls to capability-driven definitions.
7. Add the generic OpenAI-compatible adapter.
8. Add the Ubuntu Zombie adapter against its existing APIs without moving
   policy or secrets into the browser.
9. Produce and test the self-contained HTML artifact.
10. Compare periodically with llama.cpp and `standalone_llamacpp_webui` for UX
    fixes, porting focused changes rather than merging backend assumptions.

## 17. Design Decision Summary

- **Single file is a distribution format, not a single-source-file
  requirement.**
- **The core abstraction is a normalized turn/event transport, not an
  OpenAI-compatible endpoint.**
- **Conversation UX and persistence are reusable; inference and policy remain
  host responsibilities.**
- **Ubuntu Zombie is a first-class reference because it exercises streaming,
  tools, approval, queueing, authentication, and server-authoritative history.**
- **`standalone_llamacpp_webui` is evidence that extraction is viable and a
  source of lessons, but OpenAI compatibility alone is not sufficient for a
  universal chat UX.**
