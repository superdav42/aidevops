---
name: ai-hallucination-defense
description: Reduce brand hallucination risk by auditing factual consistency, claim support, and ambiguity across site content
mode: subagent
tools:
  read: true
  write: false
  edit: false
  bash: true
  glob: true
  grep: true
  webfetch: true
  task: true
---

# AI Hallucination Defense

Prevent misinformation by fixing contradictions and weakly-supported claims in source content.

## Quick Reference

- Purpose: reduce retrieval ambiguity that leads to wrong AI answers about a brand
- Inputs: high-risk fact list, claim inventory, key conversion pages
- Outputs: consistency audit, claim-evidence matrix, remediation priorities

## Workflow

### 1) Define critical fact inventory

- Track canonical values for pricing, timelines, credentials, policies, service areas, support terms
- Assign one owner source for each value
- Mark values that must stay synchronized across all pages

### 2) Run contradiction scan

- Find all mentions for each critical fact
- Label mismatches by severity: critical, moderate, low
- Prioritize fixes on pages most likely to be retrieved

### 3) Validate claim support

- Extract explicit marketing claims from primary pages
- Attach direct evidence links and confidence ratings
- Replace vague superlatives with measurable, verifiable statements

### 4) Remove ambiguity

- Rewrite unclear references (pronouns, implied subjects, outdated qualifiers)
- Keep policy and offer language explicit
- Normalize terminology across product, support, and legal pages

### 5) Re-test representative prompts

- Ask question sets likely to trigger past confusion
- Confirm generated answers align with canonical facts
- Record any residual drift for follow-up

## Risk Signals

- Conflicting numeric values between key pages
- Unsupported "best"/"leading"/"award-winning" claims
- Old versions of offers still indexable
- Different names for same feature or policy

## Related Subagents

- `geo-strategy.md` for criteria and retrieval planning
- `sro-grounding.md` for snippet survivability
- `ai-agent-discovery.md` for autonomous discoverability checks
