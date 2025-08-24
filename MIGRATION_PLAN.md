# Multi-Provider AI Assistant: OpenAI Codex Integration Plan

## Overview
This document outlines a strategy for extending the Claudia desktop application to support OpenAI's **Codex** alongside Anthropic's **Claude Code**. Rather than a replacement, this is an expansion that leverages Claudia's existing MCP (Model Context Protocol) infrastructure to add Codex as an additional AI provider. The goal is to create a multi-provider architecture while preserving all existing Claude Code functionality and user workflows.

## Current State Analysis
Claudia has significant advantages for this integration:

- **Robust MCP Infrastructure**: 727 lines of server management code in `mcp.rs` with full stdio/SSE transport support
- **Established Patterns**: Existing process management, session storage, and UI patterns are provider-agnostic at the core level
- **MCP Server Management**: JSON-based configuration with command, args, and environment variable support
- **Session Management**: Mature project and session handling in `~/.claude` directories with checkpoints and analytics

## Key Discovery: MCP-Based Integration
OpenAI Codex CLI supports MCP server mode via `codex mcp serve`, enabling integration through Claudia's existing MCP infrastructure. Community projects like `openai-codex-mcp` provide MCP server wrappers that can be configured as stdio-based MCP servers.

## Integration Goals
1. Add OpenAI Codex as an additional AI provider using the existing MCP infrastructure
2. Preserve all existing Claude Code functionality and user workflows  
3. Enable users to choose between Claude Code and Codex on a per-session basis
4. Leverage existing session management, checkpoints, and analytics with provider awareness
5. Create a foundation for future AI provider integrations

## Technical Implementation Plan

### Phase 1: MCP Integration Foundation (3-4 days)
**Setup Codex MCP Server**
- Configure OpenAI Codex CLI with MCP server capabilities using `codex mcp serve`
- Extend existing `mcp_add()` function to support Codex-specific server configuration:
  ```rust
  // Example configuration
  mcp_add(
      app,
      "codex-assistant".to_string(),
      "stdio".to_string(),
      Some("codex".to_string()),
      vec!["mcp", "serve", "--stdio"],
      env_vars, // Include OPENAI_API_KEY
      None,
      "user".to_string()
  )
  ```
- Add OpenAI API key management to existing encrypted settings infrastructure
- Test connectivity using existing `mcp_test_connection()` function

### Phase 2: Provider Abstraction (1 week)
**Create Provider Interface**
```rust
pub trait CodeExecutionProvider {
    async fn start_session(&self, prompt: &str, project_path: &str) -> Result<SessionHandle, String>;
    async fn continue_session(&self, session_id: &str, input: &str) -> Result<String, String>;
    async fn get_session_history(&self, session_id: &str) -> Result<Vec<Message>, String>;
    fn get_provider_type(&self) -> ProviderType;
    fn get_provider_name(&self) -> String;
}

pub enum ProviderType {
    ClaudeCode,
    CodexMCP,
}
```

**Implement Providers**
- `ClaudeCodeProvider`: Wraps existing Claude Code process management
- `CodexMCPProvider`: Communicates with Codex via MCP stdio protocol
- Update `commands/claude.rs` to route through provider abstraction
- Maintain existing event streaming patterns with provider context

### Phase 3: Frontend Integration (1 week)
**Provider Selection UI**
- Add provider dropdown to session creation dialog
- Update session tabs to show provider indicator (Claude/Codex icons)
- Modify `src/lib/api.ts` to include provider parameter:
  ```typescript
  export interface SessionStartRequest {
    prompt: string;
    projectPath: string;
    provider: 'claude' | 'codex';
  }
  ```

**Session Management Updates**
- Extend session metadata to include provider information
- Update session restoration logic to use appropriate provider
- Add provider-specific settings panels in preferences

### Phase 4: Feature Extensions (3-4 days)
**Agent Integration**
- Extend custom agents to support provider selection
- Add provider-specific configuration (model, temperature, max tokens)
- Update agent execution to use selected provider

**Analytics & Cost Tracking**
- Extend existing usage analytics to differentiate between providers
- Add OpenAI-specific cost calculation and token counting
- Update analytics dashboard with provider-aware visualizations
- Implement usage monitoring and optional spending limits

**Testing & Documentation**
- Comprehensive testing of both providers
- Update user documentation for provider selection
- Create OpenAI API key setup guide

## Session Storage Schema Updates

Extend existing session format to include provider information:

```json
{
  "type": "session_start",
  "provider": "codex",
  "provider_config": {
    "model": "gpt-4",
    "temperature": 0.7
  },
  "message": {
    "role": "user", 
    "content": "..."
  },
  "timestamp": "2024-01-15T10:30:00Z"
}
```

## MCP Server Configuration

Example Codex MCP server configuration:
```json
{
  "mcpServers": {
    "codex-assistant": {
      "command": "codex",
      "args": ["mcp", "serve", "--stdio"],
      "env": {
        "OPENAI_API_KEY": "${OPENAI_API_KEY}"
      }
    }
  }
}
```

## Implementation Timeline

| Phase | Tasks | Duration |
| --- | --- | --- |
| **1. MCP Integration** | Codex MCP server setup, API key management, connectivity testing | 3-4 days |
| **2. Provider Abstraction** | Provider interface, Claude/Codex implementations, backend routing | 1 week |
| **3. Frontend Integration** | Provider selection UI, session management updates, visual indicators | 1 week |
| **4. Feature Extensions** | Agent support, analytics, cost tracking, testing, documentation | 3-4 days |

**Total Duration: 2-3 weeks** (vs 6-8 weeks for the original API replacement approach)

## Risk Assessment & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| **Codex MCP Server Issues** | Medium | Medium | Test thoroughly, contribute fixes to community projects |
| **OpenAI API Costs** | High | High | Implement robust cost tracking, usage limits, spending alerts |
| **Feature Parity Gaps** | Medium | Medium | Document differences, provide fallback options |
| **User Experience Confusion** | Low | Medium | Clear UI indicators, comprehensive onboarding |
| **Session Compatibility** | Low | Low | Provider labeling, appropriate error handling |

## Success Metrics

1. **Seamless Integration**: Existing Claude Code workflows remain unchanged
2. **User Adoption**: Easy provider switching with clear value proposition
3. **Cost Transparency**: Users understand and control OpenAI usage costs
4. **Extensibility**: Architecture easily supports additional providers
5. **Performance**: No degradation to existing Claude Code performance

## Future Extensibility

This architecture enables easy addition of other AI providers:
- Anthropic Claude API (direct)
- Google Gemini
- Local models via Ollama
- Custom enterprise models

Each new provider requires only:
1. MCP server configuration or direct API implementation
2. Provider interface implementation  
3. UI configuration options

## Conclusion

This MCP-based integration approach leverages Claudia's existing strengths rather than replacing them. By using the established MCP infrastructure, we can add OpenAI Codex support in 2-3 weeks with minimal risk to existing functionality. Users gain provider choice while maintaining access to all Claude Code features, creating a more flexible and future-proof AI assistant platform.

## Key Success Factors

1. **Leverage Existing Architecture** – Build on proven MCP infrastructure rather than rebuilding
2. **Preserve User Experience** – Maintain all existing workflows while adding new capabilities  
3. **Incremental Deployment** – Add Codex as an option, not a replacement
4. **Provider Transparency** – Clear indication of which provider is being used for each session
5. **Future Extensibility** – Architecture that easily supports additional AI providers

This approach transforms a risky 6-8 week migration into a manageable 2-3 week enhancement that strengthens rather than replaces Claudia's core capabilities.