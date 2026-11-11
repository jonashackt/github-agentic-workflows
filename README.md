# github-agentic-workflows
Example project showing how to use GitHub Agentic Workflows to manage your repos automagically


# Why Agentic Workflows?

When I started my career I devoted a lot of time to creating public GitHub repositories - mostly as a start for writing about certain technologies and learning things myself. This got me into a situation, where my account held more then 100 repos - and taking care became impossible.

Sure I had my toolbelt setup: GitHub Actions workflows are a must - incl. a running Renovate to keep things up-to-date for me. But also with that keeping up with breaking changes in all sorts of tools, frameworks, and even GitHub Actions pipelines itself got out of hand. And saw myself focussing on only the few repositories using technologies I was currently interested in - and leaving the rest for good. 

With the advent of AI agents one question was stuck in the back of my head: Wouldn't it be nice to have AI manage my Renovate PRs, check for changes and generally maintain my repositories. Ideally also proposing changes for new framework-versions and overall optimization of my repositories?

Beeing busy with so many other things it seemed that I needed my time until I finally looked into the topic. And the little "Agents" tab already suggested a solution for some time now: There are [GitHub Agentic Workflows](https://github.github.com/gh-aw/) availalbe for exaclty my use case, which [just got generally available in Oct 2026](https://github.blog/changelog/2026-10-28-github-agentic-workflows-generally-available/). Time to have a look!



# GitHub integrated Agents into their Actions CI/CD workflows!

What's really nice about Agentic workflows: Their are simply integrated into GitHub Actions, no new tool invented here! As I love GitHub Actions [for many reasons](https://www.codecentric.de/en/knowledge-hub/blog/github-actions-nextgen-cicd) that was great news for me. 

Actions were simply enhanced using a ["frontmatter" syntax](https://github.github.com/gh-aw/reference/frontmatter/) which simply means using a configuration section between `---` markers directly your Actions workflow YAML files.

There are many ways to create your first GitHub Agentic Workflow: e.g. through [the GitHub web interface](https://github.github.com/gh-aw/setup/creating-workflows/#create-a-new-workflow-with-the-github-web-interface), using [your coding agent of choice](https://github.github.com/gh-aw/setup/creating-workflows/#create-a-new-workflow-with-a-coding-agent-or-vs-code) - or even start [setting things up manually](https://github.github.com/gh-aw/setup/creating-workflows/#create-a-new-workflow-manually), which I prefer to use for this post, since I wanted to have a more in-depth look about whats going on.



# Prepare GitHub workflow dir & install gh CLI aw extension

If you already worked with GitHub Actions, you're already used to creating workflows. Nothing new here:

```shell
# create .github/workflows/ directory
mkdir -p .github/workflows

# create a GitHub Actions workflow file
touch .github/workflows/first-agentic-workflow.md
```

Be sure to the [GitHub CLI installed](https://cli.github.com/) and install the Agentic Workflow extension:

```shell
gh extension install github/gh-aw
```

Now we should [setup our preferred AI engine](https://github.github.com/gh-aw/reference/engines/) in our workflow. 



# Setup your preferred AI engine: Claude through requesty.ai

There are [build-in AI engines](https://github.github.com/gh-aw/reference/engines/) available for GitHub Agentic Workflows like GitHub Copilot (which is obviously the default), Claude Code & OpenAI Codex. And there is the possibility to use [custom AI engines](https://github.github.com/gh-aw/reference/third-party-agent/) including OpenCode, Cursor, DeepSeek Harness among others.

As a regular Claude user, you may simply use `ANTHROPIC_API_KEY` [as documented in the build-in docs](https://github.github.com/gh-aw/engines/claude/) - and you're good to go.

But if you're using a unified AI gateway/LLM router (featuring multiple models) like [requesty.ai](https://www.requesty.ai/), you need to be able to configure a different `ANTHROPIC_BASE_URL` also, which isn't documented in the same area of the AI engine docs. But luckily in the reference docs at the [GitHub Actions secrets for AI engines](https://github.github.com/gh-aw/reference/auth/?#github-actions-secrets-for-ai-engines) paragraph there's the `Custom endpoints` section [for the ANTHROPIC_API_KEY](https://github.github.com/gh-aw/reference/auth/?#anthropic_api_key):

> To route Claude through a custom Anthropic-compatible endpoint (e.g., an internal proxy or Azure-hosted model), set ANTHROPIC_BASE_URL in engine.env and store any additional credentials as secrets.

So using requesty.ai will be as simple as using the following `engine.env` variables in our workflow file [.github/workflows/first-agentic-workflow.md](.github/workflows/first-agentic-workflow.md):

```yaml
engine:
  id: claude
  env:
    ANTHROPIC_BASE_URL: https://router.requesty.ai
    ANTHROPIC_AUTH_TOKEN: ${{ secrets.REQUESTY_API_KEY }}
    ANTHROPIC_MODEL: anthropic/claude-sonnet-5-5

network:
  allowed:
    - defaults
    - router.requesty.ai #TODO: is this needed for requesty?
```

This will route all requests through requesty, using the authentication as `ANTHROPIC_AUTH_TOKEN` defined in a common GitHub Actions Secret `REQUESTY_API_KEY`. You can create the Secret using the `gh aw` CLI like this:

```shell
gh aw secrets set REQUESTY_API_KEY --value "YOUR_REQUESTY_API_KEY"
```

or using the GitHub UI:

![](docs/github-actions-secrets-for-agentic-workflows.png)

As you may also spotted, we need to configure `router.requesty.ai` [as an allowed network via the `network.allowed` keyword](https://github.github.com/gh-aw/reference/engines/?#custom-api-endpoints-via-environment-variables). This is needed, since the Agent Workflow Firewall (AWF) would otherwise block the request to requesty.



# Create & compile your first GitHub Agentic Workflow file

With that knowledge at hand let's craft our first Agentic-boosted GitHub Actions workflow - featuring Claude as our AI engine, which the GitHub Agentic Workflows engine consumes through requesty.ai. Our [.github/workflows/first-agentic-workflow.md](.github/workflows/first-agentic-workflow.md) may look like this:

```yaml

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
```

With GitHub Agentic Workflows we have to execute another thing BEFORE beeing able to run our pipeline as we're accustomed by normal GitHub Actions: We need to compile your YAML file via the `gh aw compile` command:

```shell
gh aw compile
```






# Links

https://docs.github.com/en/copilot/concepts/agent-capabilities/agent-powered-workflows-and-operations/about-github-agentic-workflows

https://github.blog/ai-and-ml/automate-repository-tasks-with-github-agentic-workflows/

https://agenticpipeline.dev/resources/ci-cd-secrets/