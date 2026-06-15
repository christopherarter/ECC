# ECC for Reasonix

> **ECC-managed file.** Installed by Everything Claude Code via
> `./install.sh --target reasonix`. Edits here are overwritten on reinstall.
> Keep your own project memory in a separate `REASONIX.md` if you run
> `reasonix /init`.

This file provides Reasonix with the baseline ECC workflow, review standards, and security checks for repositories that install the Reasonix target.

## Overview

Everything Claude Code (ECC) is a cross-harness coding system of specialized agents, skills, and commands. The Reasonix target installs ECC's commands, agents, skills, and flattened rules into `./.reasonix/` so they are available inside DeepSeek Reasonix sessions.

## Core Workflow

1. Plan before editing large features.
2. Prefer test-first changes for bug fixes and new functionality.
3. Review for security before shipping.
4. Keep changes self-contained, readable, and easy to revert.

## Coding Standards

- Prefer immutable updates over in-place mutation.
- Keep functions small and files focused.
- Validate user input at boundaries.
- Never hardcode secrets.
- Fail loudly with clear error messages instead of silently swallowing problems.

## Security Checklist

Before any commit:

- No hardcoded API keys, passwords, or tokens
- All external input validated
- Parameterized queries for database writes
- Sanitized HTML output where applicable
- Authz/authn checked for sensitive paths
- Error messages scrubbed of sensitive internals

## Delivery Standards

- Use conventional commits: `feat`, `fix`, `refactor`, `docs`, `test`, `chore`, `perf`, `ci`
- Run targeted verification for touched areas before shipping
- Prefer contained local implementations over adding new third-party runtime dependencies

## ECC Areas To Reuse

- `.reasonix/rules/` for repo-wide operating rules
- `.reasonix/skills/` for deep workflow guidance
- `.reasonix/commands/` for slash-command patterns worth adapting into prompts/macros
- `.reasonix/agents/` for specialized subagent definitions
