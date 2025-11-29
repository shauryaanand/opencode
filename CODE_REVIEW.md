# OpenCode Code Review

This document contains a comprehensive code review of the OpenCode repository.

## Executive Summary

OpenCode is a well-structured TypeScript/Bun monorepo implementing an AI coding agent with a terminal-first user experience. The codebase follows modern patterns and demonstrates good separation of concerns. However, there are several areas that could benefit from improvements.

## Architecture Overview

The repository is organized as a monorepo with the following key packages:
- `packages/opencode` - Core TypeScript server and CLI
- `packages/tui` - Go-based Terminal UI
- `packages/sdk` - Client SDK (JavaScript and Go)
- `packages/console` - Web console applications
- `packages/web` - Web frontend

## Code Quality Assessment

### Strengths ✅

1. **Type Safety**: Good use of Zod schemas for runtime validation and type inference
2. **Modular Architecture**: Clear separation between server, providers, tools, and session management
3. **Error Handling**: Custom `NamedError` class provides structured error handling with schema validation
4. **API Design**: Well-documented OpenAPI-compliant endpoints with proper validation
5. **Configuration**: Flexible configuration system supporting JSON, JSONC, and markdown frontmatter

### Areas for Improvement 🔧

#### 1. Code Duplication in Config Migration

**File**: `packages/opencode/src/config/config.ts` (lines 90-115)

The autoshare migration code is duplicated:

```typescript
// Handle migration from autoshare to share field
if (result.autoshare === true && !result.share) {
  result.share = "auto"
}
if (result.keybinds?.messages_revert && !result.keybinds.messages_undo) {
  result.keybinds.messages_undo = result.keybinds.messages_revert
}

// Handle migration from autoshare to share field (DUPLICATE)
if (result.autoshare === true && !result.share) {
  result.share = "auto"
}
```

**Recommendation**: Remove the duplicate migration block (lines 97-103).

#### 2. Potential Memory Leak in Storage Migrations

**File**: `packages/opencode/src/storage/storage.ts`

The migration system loads all messages and parts into memory when migrating projects. For large projects, this could cause memory issues.

**Recommendation**: Consider implementing streaming/batched migrations.

#### 3. Error Swallowing

**File**: `packages/opencode/src/session/index.ts` (line 310)

```typescript
} catch (e) {
  log.error(e)
}
```

Silent error handling in the `remove` function could hide important failures.

**Recommendation**: Consider re-throwing or using a more descriptive error handling pattern.

#### 4. Type Safety in Agent Configuration

**File**: `packages/opencode/src/agent/agent.ts` (lines 178-204)

The `mergeAgentPermissions` function uses `any` types:

```typescript
function mergeAgentPermissions(basePermission: any, overridePermission: any): Agent.Info["permission"]
```

**Recommendation**: Use proper types for better type safety.

#### 5. Shell Command Injection Risk

**File**: `packages/opencode/src/tool/bash.ts` (line 148)

While there's permission checking, the command is passed directly to shell spawn:

```typescript
const process = spawn(params.command, {
  shell: true,
  ...
})
```

The tree-sitter parsing provides some protection, but commands outside the permitted paths restriction aren't fully sanitized.

**Recommendation**: Consider additional command sanitization or sandboxing for enhanced security.

#### 6. Commented Code

**File**: `packages/opencode/src/util/error.ts` (lines 2-4)

```typescript
// import { Log } from "./log"

// const log = Log.create()
```

**Recommendation**: Remove commented-out code.

#### 7. Console Logging Concerns

The repository conventions state "Don't use console" but there are patterns that could be improved:
- Error handling in some places uses `log.error(e)` without proper error serialization

#### 8. Magic Numbers

**File**: `packages/opencode/src/tool/bash.ts`

```typescript
const MAX_OUTPUT_LENGTH = 30_000
const DEFAULT_TIMEOUT = 1 * 60 * 1000
const MAX_TIMEOUT = 10 * 60 * 1000
```

While these are constants, they could benefit from being configurable or documented in a central configuration.

### Security Considerations 🔒

1. **Path Traversal Protection**: The bash tool includes path containment checks using `Filesystem.contains()` - good practice.

2. **API Key Handling**: Authentication data is properly segregated in the `Auth` namespace.

3. **Permission System**: The granular permission system (`ask`, `allow`, `deny`) provides good access control.

4. **Potential Improvement**: The `{env:...}` and `{file:...}` interpolation in config could be documented with security implications.

### Testing Coverage

The test directory shows focused unit tests for:
- Bun registry configuration
- Tool execution (bash, patch)
- File operations
- Snapshots

**Recommendation**: Consider adding integration tests for:
- Session management workflows
- Provider initialization
- MCP server connections

### Code Style Consistency

The codebase follows the project conventions well:
- No semicolons (prettier config: `semi: false`)
- Print width of 120 characters
- Namespace pattern for module organization
- Zod for schema validation

### Documentation

- Good inline JSDoc comments in schema definitions
- AGENTS.md provides clear project context
- README.md has comprehensive setup instructions

## Specific File Reviews

### `packages/opencode/src/server/server.ts`

**Rating**: Good ⭐⭐⭐⭐

- Well-structured Hono routes with OpenAPI documentation
- Proper error handling with NamedError
- Good use of middleware for logging and instance management

**Minor issues**:
- Large monolithic file (~1500 lines) could benefit from route modularization
- Some response schemas use `z.any()` which reduces type safety

### `packages/opencode/src/session/index.ts`

**Rating**: Good ⭐⭐⭐⭐

- Clean session management with proper event publishing
- Good use of Storage abstraction
- Proper handling of sharing functionality

**Minor issues**:
- Error handling in `remove` function could be improved

### `packages/opencode/src/provider/provider.ts`

**Rating**: Good ⭐⭐⭐⭐

- Flexible provider loading system
- Good support for multiple AI providers
- Dynamic SDK loading with proper error handling

**Minor issues**:
- Complex initialization logic could benefit from more modularization

## Recommendations Summary

### High Priority
1. Remove duplicate migration code in config.ts
2. Improve type safety in `mergeAgentPermissions`
3. Remove commented code

### Medium Priority
4. Consider modularizing the large server.ts file
5. Improve error handling consistency
6. Add integration tests for core workflows

### Low Priority
7. Make bash tool timeout/output limits configurable
8. Document security implications of config interpolation

## Conclusion

The OpenCode codebase demonstrates good engineering practices with strong type safety, modular architecture, and comprehensive API documentation. The main areas for improvement are code duplication cleanup, enhanced type safety in a few areas, and potential test coverage expansion. The security model is well thought out with proper permission handling and path containment checks.
