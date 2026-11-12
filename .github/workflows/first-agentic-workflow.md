---
on:
  workflow_dispatch:

permissions:
  contents: read

env:
  GH_AW_MODEL_AGENT_CLAUDE: "vertex/claude-sonnet-5-5@eu[1m]"

engine:
  id: claude
  env:
    ANTHROPIC_BASE_URL: https://router.eu.requesty.ai

network:
  allowed:
    - defaults
    - router.eu.requesty.ai
---

# Repository Analysis

Analyze the repository and identify potential
improvements to its build and CI/CD configuration.

Provide a prioritized summary.