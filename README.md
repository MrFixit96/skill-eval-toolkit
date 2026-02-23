# Skill Eval Toolkit

A Copilot CLI plugin that codifies the **diagnose → research → fix → validate** skill improvement cycle.

## What's Included

| Primitive | Name | Purpose |
|-----------|------|---------|
| **Command** | `/improve-skill` | Run the full improvement cycle on a named skill |
| **Command** | `/eval-skills` | Run Tier 1 validation (lint + score + triggers) locally |
| **Agent** | `skill-improver` | Reads eval failures, researches docs, makes surgical fixes |
| **Agent** | `skill-diagnostician` | Analyzes eval results and produces actionable diagnoses |
| **Skill** | `skill-eval` | Reference knowledge about the eval framework, tiers, and scoring |

## Quick Start

```
/eval-skills                     # Run Tier 1: lint + score + trigger coverage
/eval-skills tier2               # Also run semantic trigger eval (needs MODELS_PAT)
/improve-skill golang            # Full cycle: diagnose → research → fix → validate
```

## The Improvement Cycle

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Diagnose │───→│ Research  │───→│   Fix    │───→│ Validate │
│          │    │          │    │          │    │          │
│ Parse    │    │ Fetch    │    │ Surgical │    │ Lint +   │
│ judge    │    │ official │    │ edits to │    │ score +  │
│ reasoning│    │ docs for │    │ SKILL.md │    │ trigger  │
│ from eval│    │ accuracy │    │ only     │    │ coverage │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
      ↑                                               │
      └───────────────────────────────────────────────┘
                    Re-evaluate if needed
```
