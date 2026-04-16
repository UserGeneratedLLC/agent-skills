---
name: audit-plan
description: >
  Audit the current plan from first principles -- verify every API, library,
  framework, and pattern referenced is current, correctly used, and follows
  modern best practices. Uses Context7 MCP (`/context7-mcp` skill) and firecrawl-search
  in parallel. Use when the user says "audit plan", "audit the plan",
  "verify the plan", "check the plan", "/audit-plan", or asks to validate
  APIs and practices in a plan.
---

# Audit Plan

Audit the active plan from first principles. Every API, library, pattern, and claim is verified against live documentation. Nothing is trusted from training data.

## Posture: Zero Trust

- Assume every API reference, method call, parameter, pattern, version, and claim in the plan is **wrong** until verified against live documentation.
- Training data is stale by definition. Do NOT rely on it for any claim.
- Do NOT skip items that "look correct" -- verify externally or it didn't happen.
- If you cannot verify an item, flag it as **UNVERIFIED** with the reason. Never mark anything as "probably fine."

## Phase 1: Fractal Extraction

Read the active `.cursor/plans/*.plan.md` file line by line. For every line, extract every verifiable claim into a **numbered manifest**. Categories:

| Category | Examples |
|----------|----------|
| API/service names and versions | `reqwest 0.12`, `Gemini v1beta` |
| Method/function signatures | name, params, param types, optionality, order, defaults |
| Return types and response shapes | `-> Result<Vec<Part>>`, JSON `{ "candidates": [...] }` |
| Error types, error codes, failure modes | `404 Not Found`, `RateLimitError` |
| Import paths and module structure | `use crate::gemini::types::Part` |
| URL shapes, endpoints, HTTP methods, headers | `POST /v1/models/{model}:generateContent` |
| Data formats, serialization, encoding | JSON, protobuf, base64 |
| Enum values, constants, magic strings | `"gemini-2.0-flash"`, `Enum.Material.Plastic` |
| Configuration keys and valid values | `temperature: 0.0..2.0`, `topK: int` |
| Library/framework/SDK names and versions | `tokio 1.x`, `React 19` |
| Design patterns and architectural choices | "use middleware X for Y", "singleton pattern" |
| Implicit assumptions | "runs on the server" implies runtime, "async" implies executor |
| Inter-step data flows | output of step N feeding into step N+1 |

**Second pass**: Re-read the entire plan. Compare against the manifest. Add anything missed.

**The manifest IS the checklist. Nothing gets verified that isn't on it. Nothing on it gets skipped.**

## Phase 2: Parallel Verification

For each manifest item, verify **all** of the following -- no shortcuts:

**(a) Existence** -- It exists in the current version of the API/library.
**(b) Signature match** -- Parameter names, types, optionality, order all match.
**(c) Return type match** -- Return type and shape match what the plan assumes.
**(d) Deprecation check** -- Not deprecated. If it is, identify the replacement.
**(e) Modern alternative** -- No newer/better/simpler way to do the same thing.
**(f) Error contract** -- Error handling matches the documented failure modes.
**(g) Behavioral correctness** -- The plan's description of what the API *does* matches reality. Subtle semantic misuse is the hardest bug class.

**Fractal recursion**: When verifying an item reveals sub-items (e.g., a method takes an options object with 8 fields), add those sub-items to the manifest and verify them too. Continue recursing until every leaf is verified.

### Subagent Strategy

**You are an orchestrator, not a worker.** Delegate verification to subagents. Spend most of your turns launching and collecting subagent results, not doing lookups yourself.

- **`generalPurpose` (Context7)**: Spawn in parallel batches of 10+. One per library/framework/SDK. Each subagent follows the `/context7-mcp` skill: call `resolve-library-id` to find the library, then `query-docs` to fetch documentation. If there are 30 items across 8 libraries, that's 8+ concurrent subagents, not 8 sequential ones. Do NOT serialize lookups.
- **`explore`**: Spawn to read and cross-reference local docs in `.cursor/docs/` and codebase files in parallel while Context7 lookups run.
- **`generalPurpose`**: Spawn for complex verification requiring multi-step reasoning -- tracing a data flow across multiple files, verifying an architectural pattern against multiple sources.
- **`shell`**: Spawn to run `firecrawl search "<api> <method> parameters documentation" --scrape` for external APIs, breaking changes, deprecations. Multiple searches in parallel.

**Escalation order**: `.cursor/docs/` local docs first → Context7 MCP (`/context7-mcp` skill) → firecrawl-search. Only escalate when the previous source is insufficient.

When any subagent returns results, **read them fully**. Do not skim summaries.

## Phase 3: Execution Path Tracing

After individual items are verified, trace the plan's execution paths end-to-end:

- For each step that produces output consumed by a later step, verify the **output shape matches the expected input shape**.
- Flag type mismatches, missing fields, implicit conversions, and assumed defaults.
- Check that error paths from early steps are handled by later steps (or explicitly noted as unhandled).
- Verify ordering constraints -- does step N truly need to complete before step N+1, or is there a false dependency?

## Phase 4: Adversarial Review

After all items are verified and paths traced, perform a hostile review:

- Are there APIs or patterns that **should** be used but **aren't** mentioned?
- Are there known footguns, gotchas, or breaking changes in the versions being used?
- Are there simpler or more idiomatic alternatives to multi-step sequences?
- Does the plan reinvent anything that a well-known library already solves?
- Are there security, performance, or reliability concerns the plan ignores?
- Would an expert in this stack do it differently? How?
- Does the plan produce modular, composable systems or tightly-coupled slop? If the same pattern appears in two or more places, it should be a shared abstraction.
- Are there opportunities to refactor existing code touched by the plan to be more maintainable -- cleaner interfaces, better separation of concerns, reduced coupling? Flag these as part of the audit.

## Phase 5: Report and Amend

Present findings grouped by severity:

| Severity | Meaning | Action |
|----------|---------|--------|
| **Wrong** | API doesn't exist, signature mismatch, deprecated with no replacement | Must fix |
| **Outdated** | Works but there's a newer/better way | Should fix |
| **Risky** | Subtle semantic misuse, missing error handling, implicit assumptions | Should fix |
| **Suggestion** | Style, idiom, simplification | Optional |

For each finding include: manifest item number, what the plan says, what the docs say (with source URL or file path), and the specific correction.

Use `AskQuestion` to present critical findings before editing the plan. Amend the plan file directly with all approved corrections.

## Rules

These are non-negotiable:

- Do NOT batch-summarize or skip-ahead. Process every manifest item individually.
- A single unverified item is a failed audit.
- If a verification tool returns ambiguous results, escalate to a different tool.
- If you cannot verify an item with available tools, flag it as **UNVERIFIED** with the reason.
- Do not self-impose low parallelism limits. 10+ concurrent subagents is normal.
