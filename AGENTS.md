# shard-provider Repository Guidance

## Purpose

`shard-provider` is Shard's host-free provider transport boundary. It owns
provider-facing wire contracts and low-level HTTP behavior while accepting all
host decisions and configuration explicitly.

Sibling repositories:

- [`shard-v2`](https://github.com/oupadhyay/shard-v2) — desktop host and UI.
- [`shard-tool-api`](https://github.com/oupadhyay/shard-tool-api) — canonical,
  provider-neutral tool contracts.
- [`shard-external-tools`](https://github.com/oupadhyay/shard-external-tools) —
  host-free external tool implementations.

## Ownership

This repository owns:

- Gemini Files upload/delete protocol, DTOs, explicit transport configuration,
  status handling, and returned file identifiers;
- Gemini text/multimodal embedding request and response transport;
- Gemini chat/generateContent and Interactions DTOs, message/tool request
  shaping, schema normalization, SSE decoding, and provider stream events;
- reusable OpenAI-compatible vision request shaping, Bearer authentication,
  HTTP transport, response extraction, and bounded error previews.

It does **not** own:

- endpoint or API-key discovery, provider/model choice, retries, fallback, or
  prompt/workflow composition;
- image lifecycle, chat-history file-URI ownership, or cleanup policy;
- embedding chunking/invalidation, vector databases/schema, when to embed, or
  memory retrieval/search policy;
- the decision to invoke vision fallback or vision model priority;
- Tauri commands/events, persistence/SQLite, sessions, UI state, or external
  tool execution.

Transport code may apply credentials and endpoints supplied by its caller; it
must not discover them from host configuration, keychains, or global state.

## Dependency Rules

The one-way graph is:

```text
shard-v2 ──> shard-tool-api
    ├──────> shard-external-tools ──> shard-tool-api
    └──────> shard-provider ─────────> shard-tool-api
```

- This crate may depend on `shard-tool-api` at one immutable Git revision.
- It must never depend on `shard-external-tools`.
- Do not add Tauri, database/persistence crates, OS keychain/config lookup, UI
  emitters, host model selection, retry/fallback orchestration, or workflow
  policy.
- Keep trust boundaries explicit: validate paired/multimodal inputs, preserve
  stable wire shapes, bound error output, and distinguish partial protocol
  events from complete tool calls.

## Build and Validation

Run from the repository root:

```bash
cargo fmt --all -- --check
cargo check --all-targets
cargo test --all-targets
cargo clippy --all-targets -- -D warnings
RUSTDOCFLAGS="-D warnings" cargo doc --no-deps
cargo tree -e normal
cargo tree -d
cargo tree -i shard-tool-api
```

Every protocol change needs focused wire-shape and transport tests, including
headers, query parameters, status/error handling, and streaming event order as
applicable. Keep `Cargo.lock` synchronized with the reviewed dependency graph.

## Updating Pinned Revisions

`Cargo.toml` pins `shard-tool-api` by immutable `rev`. To update it:

1. merge and validate the tool-contract change first;
2. replace `rev` with the resulting commit SHA—never a branch name;
3. run `cargo update -p shard-tool-api` to update `Cargo.lock`;
4. run all checks above and confirm `cargo tree -i shard-tool-api` shows one
   expected Git source;
5. coordinate the same tool-API revision with `shard-external-tools` and
   `shard-v2` before host cutover.

After this crate changes, merge and validate it standalone, then update
`shard-v2` to the exact resulting Git revision. Do not point consumers at an
unreviewed moving branch.

## Host GUI Regression Matrix

This crate has no GUI. Validate affected behavior through the real `shard-v2`
Tauri application after provider changes:

- Gemini Files: image upload, use in a chat turn/history, and cleanup;
- embeddings: memory creation or image/text embedding plus retrieval/search;
- Gemini chat: normal streaming, reasoning/thought signatures, tool calls, host
  UI events, persistence, and cancellation;
- OpenAI-compatible vision: image analysis and host-selected fallback while
  preserving host model priority and prompt composition;
- shared DTO changes: one provider tool call and one external-tool dispatch to
  catch duplicate or incompatible `shard-tool-api` types.

Use deterministic transport fixtures when live credentials are unavailable and
state that limitation explicitly. A screenshot illustrates the result; an
executed interaction plus request/event evidence verifies it.
