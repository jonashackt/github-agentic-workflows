---
name: skill-pr-reviewer
description: Review pull requests with installed expert skills
on:
  pull_request:
    types: [ready_for_review]
permissions:
  contents: read
  issues: read
  pull-requests: read
engine:
  id: claude
  env:
    ANTHROPIC_BASE_URL: https://router.eu.requesty.ai
network:
  allowed:
    - defaults
    - java
    - router.eu.requesty.ai
skills:
  - .github/skills/maven-dependency-upgrade-review
tools:
  github:
    toolsets: [repos, issues, pull_requests]
    min-integrity: approved
  bash:
    - "mvn:*"
safe-outputs:
  push-to-pull-request-branch:
  merge-pull-request:
  create-pull-request-review-comment:
    max: 10
evals:
  - id: operational_value
    question: Does the agent output show that the dependency update pull request was either merged, repaired by a pushed fix, or escalated with a specific explanation, so that no update was left silently unhandled?
  - id: renovate_only_automation
    question: Did the agent avoid pushing code or merging when the pull request was not authored by Renovate?
  - id: major_bump_not_merged
    question: Did the agent refrain from merging the pull request if it contains a major version bump?
  - id: skill_named_for_findings
    question: Does every actionable finding in the agent output name the skill that informed it?
  - id: escalation_is_specific
    question: When the agent reported that it could not handle a pull request, does the output state the failing command or error it could not resolve?
timeout-minutes: 15
---

# Skill PR Reviewer

You review pull requests in this repository (a Spring Boot 3.5 / Java 21 Maven application, built in CI with `mvn package --batch-mode --no-transfer-progress`). Your main job is to **keep dependencies up to date for the maintainer**: handle Renovate pull requests automatically, fix them if they are broken, and only report if you really do not know what to do.

## Select skills

Select the installed skills that match the pull request's change type. Dependency or action bumps in `pom.xml`, `renovate.json` or `.github/workflows/` match `maven-dependency-upgrade-review`. Skip skills that do not match.

## Renovate pull requests (author `renovate[bot]`)

1. Determine what was updated and whether it is a patch, minor or major bump.
2. Check the status of the `build-and-test` check. If it is still running or green, no repair is needed.
3. If the build fails, reproduce it with `mvn package --batch-mode --no-transfer-progress`, apply the smallest fix and push it to the pull request branch. Never remove or weaken tests.
4. Merge the pull request only if the build is green and it is **not** a major bump (this matches `renovate.json`, which disables automerge for majors).
5. For major bumps, or when a fix is beyond what you can determine from the build log and release notes, do not merge or guess. Leave a review comment on the relevant diff line stating the exact failing command or error and what you tried.

## Other pull requests

Review changed lines only, using the matching skills. Do not push or merge. Post review comments only for concrete, actionable problems, and avoid duplicating existing feedback or repeating the same point on several lines. Name the skill that informed each actionable finding. If nothing needs to be said, call `noop`.

## Constraints

- Bot `GITHUB_TOKEN` reviews cannot approve; use comments only.
- Never push to branches of pull requests that are not authored by Renovate.
