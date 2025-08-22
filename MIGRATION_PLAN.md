# Migration Plan: Claudia to OpenAI Codex

## Overview
This document outlines a strategy for migrating the Claudia desktop application from Anthropic's **Claude Code** to OpenAI's **Codex** model. The goal is to retain Claudia's features—project and session management, custom agents, analytics, and more—while replacing all Claude‑specific functionality with Codex equivalents.

## Current State Summary
- Claudia is tightly coupled to the `claude` CLI. Rust commands spawn a separate process and stream its output into the GUI.
- The frontend calls Tauri commands such as `execute_claude_code`, `continue_claude_code`, and `resume_claude_code` to manage interactive sessions.
- A `claude_binary` module searches the system for the installed Claude executable and handles environment configuration.
- Session history, settings, and checkpoints live inside `~/.claude` directories. Analytics events and UI copy all reference Claude Code terminology.

## Migration Goals
1. Replace reliance on the `claude` binary with direct calls to OpenAI's Codex API.
2. Preserve existing UX for starting, continuing, and resuming coding sessions.
3. Maintain session history, checkpoints, and analytics using a Codex‑compatible storage layout.
4. Provide a clean abstraction so additional providers can be added in the future.

## Detailed Steps
### 1. Introduce Provider Abstraction
- Create a provider interface encapsulating "start/continue/resume session" and stream handling.
- Implement `ClaudeProvider` as a thin wrapper around the current process‑based approach (for backward compatibility) and a new `CodexProvider` that uses OpenAI APIs.
- Refactor frontend API utilities to dispatch through this provider layer rather than directly invoking Claude commands.

### 2. Backend: Replace Process Execution with HTTP Requests
- Remove `spawn_claude_process` and related child‑process management. Instead, create async functions that call the Codex completion API (`async-openai` crate recommended) and stream tokens to the frontend via Tauri events.
- Delete `claude_binary.rs` and any path‑finding logic. Substitute configuration for `OPENAI_API_KEY` and model selection.
- Adapt process registry/state to track in‑memory sessions instead of OS processes; reuse existing database schema where possible.

### 3. Session Storage & File Layout
- Replace `~/.claude` directories with a new provider‑agnostic path such as `~/.codexia`.
- Migrate existing session files and checkpoints to the new structure (write a one‑time migration script).
- Update checkpoints, `CLAUDE.md` management, and hooks to read/write from the new root.

### 4. Frontend Updates
- Rename public UI elements from “Claude” to “Codex” or a neutral term like “AI Assistant.”
- Update `src/lib/api.ts` to expose provider‑neutral methods (`executeSession`, `continueSession`, `resumeSession`). Internally, call the active provider.
- Adjust components that listen for events like `claude-output` to use provider‑agnostic channels.

### 5. Agent & MCP Features
- Ensure custom agents use the Codex provider for execution. Add configuration options for model, temperature, and tool usage.
- Review MCP server management to confirm compatibility with Codex or provide alternative tooling if required.

### 6. Analytics & Telemetry
- Extend analytics to track Codex usage (tokens, latency, cost). Update existing events to distinguish between providers.
- Provide a migration script to convert historical Claude usage data or clearly separate the two datasets.

### 7. Security & Configuration
- Support API key management via encrypted local storage. Provide UI for entering and testing `OPENAI_API_KEY`.
- Review and tighten scopes for network/file permissions given the removal of external processes.

### 8. Testing & QA
- Add unit tests for the provider abstraction and Codex streaming implementation.
- Update existing integration tests to run against a mocked Codex endpoint.
- Perform manual QA on all major features: session management, agents, checkpoints, and analytics.

### 9. Documentation & Release
- Rename repository references from Claudia to a Codex‑friendly name as desired.
- Update README, screenshots, and marketing copy to describe Codex integration.
- Provide clear upgrade instructions for existing users, including how to migrate their `~/.claude` data.

## Estimated Timeline
| Phase | Tasks | Duration |
| --- | --- | --- |
| **1. Abstraction Layer** | Provider interface, basic Codex client, feature toggles | 1–2 weeks |
| **2. Core Migration** | Replace CLI logic, session storage, API key management | 2–3 weeks |
| **3. Feature Parity** | Agents, analytics, MCP, testing | 2 weeks |
| **4. Docs & Release** | Documentation, migration scripts, packaging | 1 week |

## Risks & Mitigations
- **API Limits / Costs** – Implement rate limiting and cost tracking to avoid surprises.
- **Model Differences** – Codex may behave differently than Claude Code. Provide tuning options and document known gaps.
- **Backward Compatibility** – Keep a Claude provider during transition or offer data migration tooling.

## Conclusion
Migrating from the Claude CLI to the Codex API requires replacing process‑based execution with HTTP streaming, introducing a provider abstraction, and updating storage, analytics, and UI terminology. With the steps above, Claudia can evolve into a Codex‑powered development assistant while retaining its rich feature set.
