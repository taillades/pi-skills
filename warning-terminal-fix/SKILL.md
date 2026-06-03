---
name: warning-terminal-fix
description: Analyze warnings shown in terminal output, explain what they mean, determine severity, and recommend one or more concrete fixes. Use when the user pastes terminal warnings, build warnings, runtime warnings, deprecation notices, package manager warnings, compiler warnings, linter warnings, or test warnings.
---

# Warning Terminal Fix

Use this skill when the user wants help understanding or fixing terminal warnings.

## Goals

- Identify the source of each warning.
- Explain what is happening in plain but technical language.
- Classify severity:
  - harmless/noise
  - should fix eventually
  - likely bug
  - blocking/failing soon
  - security/reliability risk
- Recommend one or more fixes.
- If working in a repo, inspect the relevant files before editing.
- Prefer the smallest safe fix.
- Do not silence warnings unless that is clearly the right fix.

## Workflow

1. Ask for missing context if needed:
   - exact command that produced the warning
   - full warning output
   - language/framework/tool version
   - whether the user wants an explanation only or code changes too

2. Parse the warning:
   - tool or subsystem producing it
   - file/line/module/package involved
   - warning category, e.g. deprecation, type, config, dependency, resource, security, performance

3. Explain:
   - what triggered it
   - whether it affects correctness now
   - what may break later
   - whether it is from user code, dependency code, environment, or configuration

4. Recommend fixes, ordered by preference:
   - best long-term fix
   - minimal safe fix
   - workaround/suppression only if appropriate
   - no-op if truly harmless

5. If the user asks to fix it:
   - inspect the relevant files
   - make precise edits
   - run the command again or the narrowest validation command
   - report remaining warnings, if any

## Response Format

For each warning, respond with:

````markdown
### Warning: <short name>

**Source:** <tool/package/file if known>  
**Severity:** <classification>  
**What it means:** <brief explanation>  
**Why it happens:** <cause>  

**Recommended fix:** <best fix>

**Other options:**
1. <alternative>
2. <alternative>

**Validation:**
```bash
<command to confirm>
```
````

If there are many repeated warnings, group duplicates and say how many occurrences were seen.

## Fixing Rules

- Prefer fixing root causes over suppressing output.
- Do not upgrade major dependency versions without calling out risk.
- Do not remove warnings by redirecting stderr or disabling checks unless the user explicitly wants that.
- If a warning comes from a third-party dependency, first check whether a newer compatible version fixes it.
- If unsure whether a warning is safe, say so and explain what evidence is missing.
- If the warning indicates deprecated API usage, replace with the documented supported API when possible.
- If multiple fixes exist, explain tradeoffs clearly.

## Common Warning Types

### Deprecation warnings

Explain:
- which API/config is deprecated
- replacement API/config
- timeline/risk if ignored

Preferred fixes:
1. migrate to replacement
2. pin compatible version temporarily
3. suppress only for known third-party noise

### Compiler/build warnings

Explain:
- whether they affect generated artifacts
- whether they can become errors under stricter flags

Preferred fixes:
1. change code/config
2. adjust build flags only if warning is intentional

### Package manager warnings

Explain:
- peer dependency mismatch
- deprecated package
- lockfile/version issue
- engine/version mismatch

Preferred fixes:
1. install compatible versions
2. upgrade or replace deprecated packages
3. document why warning is accepted

### Runtime warnings

Explain:
- whether behavior is already affected
- likely future failure mode

Preferred fixes:
1. update call site/config
2. add explicit handling
3. validate with a focused reproduction

### Security warnings

Explain:
- whether it is direct or transitive
- exploitability if known
- impact of upgrade

Preferred fixes:
1. upgrade patched package
2. replace vulnerable package
3. apply mitigation only if upgrade is not possible
