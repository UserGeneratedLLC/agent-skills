---
name: atlas-push
description: Safely push filesystem changes into Roblox Studio via Atlas MCP with automatic conflict merging. Use when the user says "atlas push", "sync to studio", "push to studio", "push changes", or wants to get their local code changes into a running Studio session without clobbering other people's work.
---

# Atlas Push

Automated workflow for pushing local filesystem changes into Roblox Studio. Handles conflicts by fetching the Studio version, merging intelligently, and using optimistic concurrency guards (`studioHash`) to prevent clobbering concurrent edits.

**Goal:** Get OUR changes into Studio while preserving everyone else's work.

**CRITICAL: NEVER use shell/CLI commands for Atlas operations.** Atlas is an MCP server -- all interactions use the Atlas MCP tools available in your tool list, not terminal commands. There is no `atlas sync` CLI command. Running shell commands like `atlas sync` will fail.

## Prerequisites

- `atlas serve` must be running (the Atlas MCP server must be reachable)
- Roblox Studio must be open with the Atlas plugin installed and connected to the MCP stream
- Atlas tools are available as **direct function calls** in your tool list. Their names end with `-atlas-<toolName>` (e.g. `*-atlas-atlas_sync`, `*-atlas-get_script`). The prefix varies per project -- search your available tools for ones containing `atlas` in the name.

If the MCP tools are not available or return connection errors, tell the user to check that `atlas serve` is running and Studio has the Atlas plugin active.

## Workflow

### Step 1: Dryrun

Always start with a dryrun. Never apply changes without reviewing them first.

Call the **`atlas_sync`** tool with `mode: "dryrun"`.

**Parse the response:** The response text contains a human-readable summary followed by a `<json>...</json>` block. Extract and parse the JSON. It has this shape:

```json
{
  "status": "dryrun",
  "changes": [
    {
      "path": "ServerScriptService/Main",
      "direction": "push" | "pull" | "unresolved",
      "id": "32-char-hex-ref",
      "className": "Script",
      "patchType": "Add" | "Edit" | "Remove",
      "studioHash": "40-char-sha1-hex",
      "defaultSelection": "push" | "pull" | null,
      "fsPath": "src/server/Main.server.luau",
      "properties": { "PropName": { "current": ..., "incoming": ... } }
    }
  ]
}
```

**If status is `"empty"`:** No changes to sync. Tell the user and stop.

### Step 2: Triage

Categorize every change entry:

| `direction` | Meaning | Action |
|---|---|---|
| `"push"` | Only local changed | Auto-push (no conflict) |
| `"pull"` | Only Studio changed | Accept pull (preserve their work) |
| `"unresolved"` | Both sides changed | Needs merging |

Also note the `patchType`:
- `"Edit"` on a script class (`Script`, `LocalScript`, `ModuleScript`) -- mergeable via `get_script`
- `"Edit"` on a non-script -- property conflict, resolve by examining `properties` field
- `"Add"` / `"Remove"` -- structural, usually one-directional; if unresolved, needs judgment

**If there are zero unresolved changes:** Skip to Step 5 (build overrides for all pushes and proceed).

### Step 3: Merge Unresolved Scripts

For each unresolved change where `className` is a script type (`Script`, `LocalScript`, `ModuleScript`) and `patchType` is `"Edit"`:

1. **Fetch Studio source:** Call the **`get_script`** tool with `id: "<id from the change entry>"`.

Response includes `source` (Studio's current code), `studioHash` (SHA1 of git blob format), `fsPath`, and `id`. Save the `studioHash` -- you need it for the override.

2. **Read local file:** Use the `fsPath` from the change entry to read the local filesystem version.

3. **Merge:** Produce a merged version that incorporates both local and Studio changes. Rules:
   - **Our changes take priority** -- the user invoked atlas-push because they want their local edits in Studio.
   - **Preserve Studio-only additions** -- if Studio has new code that doesn't conflict with our changes (new functions, new event handlers, new variables in non-overlapping regions), keep them.
   - **For overlapping edits** (both sides changed the same lines/function): prefer our version, but inspect whether the Studio version added something important that ours removed. Use your code understanding to produce a correct result.

4. **Write merged result** to the local filesystem path (overwrite the file). This is what will be pushed to Studio.

### Step 4: Resolve Non-Script and Structural Conflicts

For unresolved changes that are NOT script edits:

- **Non-script property edits:** Examine the `properties` field (has `current` and `incoming` values). Push our version (direction: `"push"`) unless the property is something we didn't intentionally change.
- **Add (unresolved):** If we added something locally and Studio also added something -- push ours.
- **Remove (unresolved):** If we removed something but Studio still has it -- push the removal if it was intentional. If unclear, ask the user.

### Step 4b: Post-Merge Semantic Review

After producing each merged script, review the result for functional issues the merge may have introduced:

- **Duplicate definitions:** Same variable or function defined twice
- **Broken requires:** `require()` paths that reference moved/renamed modules
- **Signature mismatches:** Function gained a parameter on one side but callers on the other side don't pass it
- **Dangling references:** Variables removed by one side but still used by the other
- **Conflicting logic:** Same config value changed to different things; early-return added that skips new code from the other side

**If you spot a clear issue:** Fix it in the merged file before proceeding.

**If the issue is ambiguous** (unclear which side's intent should win): Ask the user. Show both versions of the conflicting section, explain the issue concisely, and ask what they want. Apply their answer and continue immediately.

### Step 4c: Escalation Rules

Throughout Steps 3-4b, if you encounter a conflict you genuinely cannot resolve:

1. Show the user both versions (local and Studio) of the conflicting section
2. Explain the conflict in one or two sentences
3. Ask the user what to do -- use plain text, NOT multiple-choice, so they can give nuanced instructions (e.g. "keep Studio's version of the damage calc but use my version of the config table")
4. Apply their answer immediately
5. Continue to the next conflict or proceed to sync

**Stay in the loop.** The user should never need to re-invoke this skill. Keep asking and applying until all conflicts are resolved.

### Step 5: Build Overrides

Construct the overrides array for the final sync. For every change you want to push:

```json
{
  "id": "<32-char hex id from dryrun>",
  "direction": "push",
  "studioHash": "<studioHash from get_script response>"
}
```

- **Merged scripts:** Use the `studioHash` you got from `get_script` in Step 3. This is the safety guard -- if someone edited the script between your read and now, the override will be rejected instead of silently clobbering.
- **Clean pushes** (direction was already `"push"` in dryrun): Include them in overrides with `direction: "push"`. The `studioHash` from the dryrun change entry can be used if present.
- **Clean pulls** (direction was `"pull"`): Do NOT include these in overrides. Let them resolve naturally (Studio version wins).

### Step 6: Final Sync

Call the **`atlas_sync`** tool with `mode: "standard"` and your `overrides` array.

### Step 7: Verify and Retry

Parse the response status:

| Status | Meaning | Action |
|---|---|---|
| `"success"` | All changes applied | Done. Tell user sync succeeded. |
| `"empty"` | Nothing to sync | Done. Already up to date. |
| `"rejected"` | User rejected in Studio UI | Tell user it was rejected. |
| `"fastfail_unresolved"` | Unresolved changes remain | Some overrides didn't match or new conflicts appeared. Loop back to Step 1 with a new dryrun. |

**If an override was rejected** (studioHash mismatch = someone edited the script concurrently):
1. Re-fetch the script via `get_script`
2. Re-merge with the new Studio version
3. Write the updated merge to the filesystem
4. Build new overrides with the fresh `studioHash`
5. Retry the sync

**Keep retrying** until sync succeeds or user explicitly says to stop. Each retry is safe because `studioHash` prevents clobbering.

### Step 8: Post-Push Playtest

**Skip this step by default.** Only run a playtest if the user's message explicitly requests it -- look for phrases like "and test", "playtest", "test it", "run test", "with test", or "check it".

After a successful sync and only when requested, run a quick playtest to catch runtime errors introduced by the push.

1. **Record pushed files.** Collect all `fsPath` values from changes where you pushed (direction `"push"` in the final sync response, or any script you merged and pushed). Also note the corresponding instance path segments (e.g. `src/server/GameManager.server.luau` corresponds to `ServerScriptService.GameManager`). You need these for error attribution.

2. **Run the playtest:** Call the **`run_script_in_play_mode`** tool (may appear with a truncated/hashed name like `run_script_i*`) with `code: "task.wait(7)\nprint(\"boot complete\")"`, `mode: "start_play"`, `timeout: 15`.

3. **Parse and filter errors.** From the response `errors` array, keep:
   - All entries with level `"error"`
   - Warnings that indicate real breakage ("Infinite yield possible", security errors)
   - Ignore routine/cosmetic warnings, asset loading warnings, deprecation notices

4. **Attribute errors to pushed files.** Roblox error messages include the script path, e.g.:
   `ServerScriptService.GameManager.PlayerHandler:42: attempt to index nil`

   Match the script name/path segments against your pushed `fsPath` list. Attribution is at the **whole-file level** -- if an error comes from a script we pushed, it's ours even if the erroring line wasn't part of our specific diff. Our change may have broken something else in that file.

5. **Handle errors:**
   - **Our errors** (originate from a pushed file): read the script, diagnose the error using the message and line number, fix the code, write the corrected file, then **loop back to Step 1** (dryrun the fix, sync it, re-test). Keep looping until the playtest is clean.
   - **Others' errors** (no match to any pushed file): ask the user. Show the error, explain it doesn't originate from files we pushed, and recommend they investigate in a planning phase. Let the user decide via text input what to do. Do not auto-fix code you didn't push.
   - **No errors**: report success. Push and playtest complete.

## Safety Invariants

1. **Never sync without a dryrun first** -- always review what will change.
2. **Always use `studioHash` on overrides** -- this is the optimistic concurrency guard. If the hash doesn't match (someone edited between our read and our push), the override is rejected rather than silently overwriting.
3. **Never discard Studio-only changes silently** -- if Studio has changes we didn't make, either merge them in or explicitly acknowledge discarding them.
4. **Merged files are written to the local filesystem, then pushed** -- we never send content directly to Studio. Atlas handles the actual transfer.
5. **Retry on concurrency failures** -- a rejected studioHash means "try again", not "give up".

## Error Handling

| Error | Recovery |
|---|---|
| MCP tools not found | Tell user to check that Atlas MCP server is configured and `atlas serve` is running |
| Plugin not connected | Tell user to open Studio with Atlas plugin |
| `sync_in_progress` | Wait a moment and retry |
| `already_connected` | Live sync is active; changes sync automatically; no manual push needed |
| `get_script` fails | Fall back to pushing local version only; warn user that Studio version couldn't be read |
| Filesystem write fails | Abort and report error; do not proceed with sync |

## Quick Reference: Atlas MCP Tools

All tools are direct function calls in your tool list. Names follow the pattern `*-atlas-<toolName>`.

| Tool | Parameters | Purpose |
|---|---|---|
| `atlas_sync` | `mode` ("dryrun", "standard", "fastfail", "manual"), `overrides` (optional array) | Sync changes between filesystem and Studio |
| `get_script` | `id` (hex ref) or `fsPath` (relative path), `fromDraft` (optional bool) | Read a script's source from Studio |
| `run_script_in_play_mode` | `code` (Luau string), `mode` ("start_play" or "run_server"), `timeout` (optional seconds) | Run code in a play session then auto-stop |
| `start_stop_play` | `mode` ("start_play", "run_server", "stop") | Manual play mode control |
| `syncback` | (none) | Pull entire Studio state to filesystem |
