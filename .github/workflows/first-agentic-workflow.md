---
on:
  workflow_dispatch:

permissions:
  contents: read

engine:
  id: claude
  env:
    ANTHROPIC_BASE_URL: https://router.requesty.ai
    ANTHROPIC_AUTH_TOKEN: ${{ secrets.REQUESTY_API_KEY }}
    ANTHROPIC_MODEL: anthropic/claude-sonnet-5-5

network:
  allowed:
    - defaults
    - router.requesty.ai
---

# Repository Analysis

Analyze the repository and identify potential
improvements to its build and CI/CD configuration.

Provide a prioritized summary.