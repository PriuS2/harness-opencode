# Opencode Port — Changes from Original

## Overview
This repository is a port of [revfactory/harness](https://github.com/revfactory/harness) for Opencode.

## Modified Commits
- [9e037bf](https://github.com/PriuS2/harness-opencode/commit/9e037bf820d313969a15a6b7503e151ba07cc2a7) — Initial Opencode port
- [5538ba7](https://github.com/PriuS2/harness-opencode/commit/5538ba7a80075446ddb464bf8f634f21b81f5186) — Architecture modification
- [f032ae6](https://github.com/PriuS2/harness-opencode/commit/f032ae612f848d08f9ef88019cc872a7c88a6bd9) — Final adaptation

## Key Changes

| Area | Original (Claude Code) | Ported (Opencode) |
|------|------------------------|-------------------|
| Execution | Agent Teams (TeamCreate, SendMessage) | **Parallel Tasks** |
| Compatibility | Claude Code only | **Opencode only** (not compatible with Claude Code) |

## Compatibility Note
⚠️ This version uses Opencode's Task-based parallelism and is **NOT compatible** with Claude Code's Agent Team system.
