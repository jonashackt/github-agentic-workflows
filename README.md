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



# Setup your preferred AI engine: Using Agentic Workflows with Claude on Google Vertex (EU) through requesty.ai

There are [build-in AI engines](https://github.github.com/gh-aw/reference/engines/) available for GitHub Agentic Workflows like GitHub Copilot (which is obviously the default), Claude Code & OpenAI Codex. And there is the possibility to use [custom AI engines](https://github.github.com/gh-aw/reference/third-party-agent/) including OpenCode, Cursor, DeepSeek Harness among others.

As a Claude user, you may simply use `ANTHROPIC_API_KEY` [as documented in the build-in docs](https://github.github.com/gh-aw/engines/claude/) - and you're good to go.

But if you're using a AI gateway/LLM router (featuring multiple models) like [requesty.ai](https://www.requesty.ai/) things become a bit more complicated. Digging into the topic I was reminded (again), that knowing the architecture behind the scenes is always better than to stab in the dark. So using requesty with a EU-based setup together with Google Vertex AI as the Managed Inference Platform our architecture looks like this:

```shell
+------------------------------------------------------+
| GitHub Actions Runner                                |
|                                                      |
|  +------------------------------------------------+  |
|  | Agent Workflow Firewall (AWF)                  |  |
|  |                                                |  |
|  |  +----------------------+                      |  |
|  |  | Agent Container      |                      |  |
|  |  | Claude Code          |                      |  |
|  |  | No API credentials   |                      |  |
|  |  +----------+-----------+                      |  |
|  |             |                                  |  |
|  |             | http://api-proxy:10001           |  |
|  |             v                                  |  |
|  |  +----------------------+                      |  |
|  |  | API Proxy Sidecar    |                      |  |
|  |  | - API credentials    |                      |  |
|  |  | - Auth injection     |                      |  |
|  |  | - Request routing    |                      |  |
|  |  +----------+-----------+                      |  |
|  |             |                                  |  |
|  |             v                                  |  |
|  |  +----------------------+                      |  |
|  |  | Squid Proxy          |                      |  |
|  |  | Network egress       |                      |  |
|  |  +----------+-----------+                      |  |
|  +-------------|----------------------------------+  |
+----------------|-------------------------------------+
                 |
                 | HTTPS
                 v
+------------------------------------------------------+
| Requesty EU                                          |
| router.eu.requesty.ai                                |
| AI Gateway / Model Router                            |
+----------------+-------------------------------------+
                 |
                 v
+------------------------------------------------------+
| Google Cloud Vertex AI (EU)                          |
| Managed Inference Platform                           |
|                                                      |
| Model: Anthropic Claude Sonnet 5.5                   |
+------------------------------------------------------+
```

GitHub Agentic Workflows runs Claude Code in an isolated agent container within a GitHub Actions runner. The Agent Workflow Firewall (AWF) provides a dedicated API proxy sidecar that securely injects API credentials and forwards model requests, keeping secrets inaccessible to the agent. Outbound network traffic is controlled through a separate Squid proxy.

In the Requesty setup I use, the model requests are routed through the Requesty EU AI Gateway to Google Cloud Vertex AI, which provides managed inference for Anthropic models in the EU.



## Configuring requesty.ai in Agentic Workflows

So in order to use requesty.ai with Agentic Workflows, we need to be able to configure a different `ANTHROPIC_BASE_URL` also. This isn't documented in [the same area of the AI engine docs](https://github.github.com/gh-aw/reference/engines/). But luckily in the reference docs at the [GitHub Actions secrets for AI engines](https://github.github.com/gh-aw/reference/auth/?#github-actions-secrets-for-ai-engines) paragraph there's the `Custom endpoints` section [for the ANTHROPIC_API_KEY](https://github.github.com/gh-aw/reference/auth/?#anthropic_api_key):

> To route Claude through a custom Anthropic-compatible endpoint (e.g., an internal proxy or Azure-hosted model), set ANTHROPIC_BASE_URL in engine.env and store any additional credentials as secrets.

So using requesty.ai needs the following `engine.env` variables in our workflow file [.github/workflows/first-agentic-workflow.md](.github/workflows/first-agentic-workflow.md) (if you aren't using the requesty EU endpoint, you need to leave out the `.eu` from the urls):

```yaml
engine:
  id: claude
  env:
    ANTHROPIC_BASE_URL: https://router.eu.requesty.ai

network:
  allowed:
    - defaults
    - router.eu.requesty.ai #TODO: is this needed for requesty?
```

Additionally we need to configure `router.eu.requesty.ai` [as an allowed network via the `network.allowed` keyword](https://github.github.com/gh-aw/reference/engines/?#custom-api-endpoints-via-environment-variables). This is needed, since the Agent Workflow Firewall (AWF) would otherwise block the request to requesty.





## Getting the requesty model declaration through the GitHub Agentic Workflows compiler

You should also choose an approved model from the requesty model-library to configure in the `ANTHROPIC_MODEL` variable. You may find those via the direct link https://app.requesty.ai/model-library. Looking for the exact name, you have to search for it in your requesty.ai console:

![](docs/choose-requesty-approved-models.png)


Leaving out the model definition completely, would use a default Anthropic model name like `anthropic/claude-sonnet-5-5` and will mostly run into GitHub Actions errors like:

```shell
"content":[{"type":"text","text":"Failed to authenticate. API Error: 403 The requested model is not approved for this API key. An org or group admin can approve it in the Model Library (https://app.requesty.ai/model-library) or add it to the access list attached to this key or group"}]
```

The default GitHub Agentic Workflow [model aliases are documented here](https://github.github.com/gh-aw/reference/model-tables/).



### Trying engine.model to set to vertex/claude-sonnet-5-5@eu

The most straighforward approach for me was to use [the `engine.model` variable](https://github.github.com/gh-aw/reference/engines/#default-model-routing) to configure my requesty approved model name:

```yaml
engine:
  id: claude
  model: vertex/claude-sonnet-5-5@eu
```

But here I ran into an compilation error running `gh aw compile`:

```shell
gh aw compile
✗ Compiled 1 workflow: 0 succeeded, 1 failed, 1 error, 0 warnings

✗ Failed workflows:
✗ first-agentic-workflow.md

✗ first-agentic-workflow.md (1 error(s)):
  • 🟡 MEDIUM PRIORITY:
  • 1. .github/workflows/first-agentic-workflow.md:1:1: error: engine.model: model identifier: character '@' is not allowed in model token "claude-sonnet-5-5@eu[1m]" (segment type: model)
i → Fix this configuration issue and re-run `gh aw compile`.
✗ compilation failed
```

I turned out that the [GitHub Model Alias Format Specification](https://github.github.com/gh-aw/specs/model-alias-specification/) doesn't support the special characters `@` and `[]`.



### Trying engine.env.ANTHROPIC_MODEL to set to vertex/claude-sonnet-5-5@eu

So in my case I wanted to configure the requesty-approved model `vertex/claude-sonnet-5-5@eu`. My first attempt was to use the `engine.env.ANTHROPIC_MODEL` variable like this (which [is documented in the Claude Code Docs](https://code.claude.com/docs/en/settings#a-flag-or-environment-variable-sets-the-same-thing)):

```shell 
engine:
  id: claude
  env:
    ANTHROPIC_MODEL: vertex/claude-sonnet-5-5@eu
```

The Actions error remained:

```shell
"content":[{"type":"text","text":"Failed to authenticate. API Error: 403 The requested model is not approved for this API key. An org or group admin can approve it in the Model Library (https://app.requesty.ai/model-library) or add it to the access list attached to this key or group"}]
```

Digging a bit deeper, I found that the GitHub Agentic Workflow engine just ignored or dropped the `engine.env.ANTHROPIC_MODEL` variable. I saw in the Actions logs, in the second job under `Execute Claude Code CLI`:

```shell
 [claude-harness] 2026-11-12T08:24:57.401Z model resolution: engine=claude role=main configured=claude-sonnet-5 resolved=claude-sonnet-5 provider=anthropic status=exact
```

So I needed to try something different.



### What about GH_AW_MODEL_AGENT_CLAUDE?

I stumbled upon the [`GH_AW_MODEL_AGENT_CLAUDE` environment variable](https://github.github.com/gh-aw/reference/environment-variables/#model-override-variables), which I found in the model override environment variables section of the GitHub Agentic Workflows docs.

Ideally I wanted a solution, where the variable would be set inside the workflow file also, so I tried to directly define it there like this:

```yaml
env:
  GH_AW_MODEL_AGENT_CLAUDE: "vertex/claude-sonnet-5-5@eu[1m]"
```

Sadly, this also didn't make it through the GitHub Agentic Workflow engine:

```shell
 [claude-harness] 2026-11-12T08:24:57.401Z model resolution: engine=claude role=main configured=claude-sonnet-5 resolved=claude-sonnet-5 provider=anthropic status=exact
```

But there was a solution: Defining the `GH_AW_MODEL_AGENT_CLAUDE` variable on the GitHub repository level. Setting the variable as repository variable worked finally:

```shell
gh variable set GH_AW_MODEL_AGENT_CLAUDE --body 'vertex/claude-sonnet-5-5@eu[1m]'
gh variable get GH_AW_MODEL_AGENT_CLAUDE
```

This definition doesn't live in the workflow yaml file sadly, but I made a note in the back of my head to create an issue for that one later...


## Where to put the Requesty API token: `ANTHROPIC_AUTH_TOKEN` vs. `ANTHROPIC_API_KEY`

We somehow need to configure our Requesty API token to work with GitHub Agentic Workflows. It's quite counterintuitive I have learned!

As [the Requesty documentation states](https://www.requesty.ai/blog/claude-code-environment-variables-anthropic-base-url-auth-token), we should use `ANTHROPIC_AUTH_TOKEN` instead of `ANTHROPIC_API_KEY` when configuring a different LLM gateway than Anthropic.

So I was tempted to use `engine.env.ANTHROPIC_AUTH_TOKEN: ${{ secrets.REQUESTY_API_KEY }}` as a configuration here also. Later on this lead me to an error:

```shell
gh aw compile
✗ Compiled 1 workflow: 0 succeeded, 1 failed, 1 error, 0 warnings

✗ Failed workflows:
✗ first-agentic-workflow.md

✗ first-agentic-workflow.md (1 error(s)):
  • 🟡 MEDIUM PRIORITY:
  • 1. .github/workflows/first-agentic-workflow.md:8:1: error: strict mode: secrets detected in 'engine.env' section are excluded from the agent sandbox via awf --exclude-env (requires AWF v0.25.3+) and are not accessible to the agent when that version is in use. Found: ${{ secrets.REQUESTY_API_KEY }}. Use engine-specific secret configuration instead. See: https://github.github.com/gh-aw/reference/engines/
 5 | permissions:
 6 |   contents: read
 7 |
 8 | engine:
     ^^^^^^
 9 |   id: claude
10 |   env:
11 |     ANTHROPIC_BASE_URL: https://router.eu.requesty.ai
i → Adjust the permissions block to match the workflow's required scopes.
✗ compilation failed
```

The reason is that the Agent Workflow Firewall (AWF) [blocks the usage of such secrets](https://github.github.com/gh-aw/reference/frontmatter/#environment-variables-env) directly in the `env` section in Actions workflow:

> CAUTION: Do not use ${{ secrets.* }} expressions in the workflow-level env: section. Environment variables defined here are passed directly to the agent container, which means secret values would be visible to the AI model. In strict mode, this is a compilation error. In non-strict mode, it emits a warning.

> Use engine-specific secret configuration instead of the env: section to pass secrets securely.

This prevents secrets from beeing injected into workflows - and thus from potential leaks. Which might also be the good reason to have the scrict mode turned on in `gh aw compile`.

So just the opposide as [the Requesty documentation states](https://www.requesty.ai/blog/claude-code-environment-variables-anthropic-base-url-auth-token), with GitHub Agentic Workflows I tried to use the `ANTHROPIC_API_KEY` GitHub Actions secret instead. And: it worked!

You can create the Secret using the `gh aw` CLI like this:

```shell
gh aw secrets set ANTHROPIC_API_KEY --value "REQUESTY_API_KEY"
```

or using the GitHub UI:

![](docs/github-actions-secrets-for-agentic-workflows.png)

Since this is a Agent Workflows build-in engine key, we don't need to define it inside our workflow anywhere.



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
```

But be aware that __this is only a description of the Actions workflow, we want the GitHub Agentic AI engine to generate for us__. [The GitHub Agentic Workflows docs intro](https://github.github.com/gh-aw/introduction/overview/#coding-agents-running-with-tools-in-github-actions) has a good explanation ready for us:  

> Instead of writing intricate scripts to handle issue triage, code reviews, or release management, you describe what you want to happen. The AI agent uses the repository context, tools, and permissions that the workflow allows to interpret the situation and request appropriate actions.

So with GitHub Agentic Workflows we have to execute the `gh aw compile` command BEFORE beeing able to run our pipeline as we're accustomed by "normal" GitHub Actions. This command will trigger the GitHub AI-based generation of the actual GitHub Agentic Workflow:

```shell
gh aw compile
```

This will generate the "real" GitHub Actions based Agentic Workflow for us, residing in a `.lock.yml` file inside the `.github/workflows` dir. In our example this is [.github/workflows/first-agentic-workflow.lock.yml](.github/workflows/first-agentic-workflow.lock.yml).


The `gh aw compile` should work, but also outputs the following warning:

```shell
gh aw compile
.github/workflows/first-agentic-workflow.md: warning: safe update mode detected unapproved changes

New restricted secret(s):
  - ANTHROPIC_API_KEY

Remediation options:
  1. Use the --approve flag to allow the changes.
  2. Revert the unapproved changes.
  3. Use an interactive coding agent to review and approve the changes.

✓ .github/workflows/first-agentic-workflow.md (110.2 KB)
⚠ ⚠️  SECURITY REVIEW REQUIRED — safe update mode detected unapproved changes:

safe update mode detected unapproved changes

New restricted secret(s):
  - ANTHROPIC_API_KEY

Remediation options:
  1. Use the --approve flag to allow the changes.
  2. Revert the unapproved changes.
  3. Use an interactive coding agent to review and approve the changes.

Action required for the calling agent:
  1. Review every new secret and action listed above for potential malicious use
     (e.g. credential exfiltration, supply-chain attacks, unexpected network access).
  2. Add a security review note to the pull request description that:
     - Lists any new or removed secrets, actions, and redirect changes from this compilation.
     - Confirms you have reviewed each one and explains why it is safe to add.
     - Flags any item you cannot fully verify so a human reviewer can inspect it.

⚠ Compiled 1 workflow: 1 succeeded, 1 warning
```


Since we can use the `--approve` flag here, since we were the one that created the new `ANTHROPIC_API_KEY` secret:

```shell
gh aw compile first-agentic-workflow --approve
```

That one should run without warnings.

It's also worth to note that using `--approve` [won't deactivate the strict mode generally](https://github.github.com/gh-aw/specs/safe-update-specification/), so it can be used savely if you were the one changing secrets.

So now our generated workflow in [.github/workflows/first-agentic-workflow.lock.yml](.github/workflows/first-agentic-workflow.lock.yml) should be ready to be committed!



# Trigger your first Agentic Workflow!

As we use the `on:workflow_dispatch` keyword in our formatter workflow description, we can manually trigger our first Agentic Workflow e.g. via the GitHub Actions UI. The name is defined via the markdown heading under the formatter description - in our case: `# Repository Analysis`. So we need to explicitely click on the workflow named `Repository Analysis` and on `Run workflow` (you can leave the `aw_context` empty):

![](docs/manually-trigger-agentic-workflow-execution.png)

This should start our first Agentic Workflow:

![](docs/first-agentic-workflow-run.png)



# Add some code to be analysed

I started with a completely empty repository just to see, what I would need to configure in order to use GitHub Agentic Workflows.

But as the setup is working now, it's time to add some code that our Agentic Workflow instruction can work on:

> Analyze the repository and identify potential improvements to its build and CI/CD configuration.

> Provide a prioritized summary.

So I copied the code from https://github.com/jonashackt/microservice-api-spring-boot - let's see what our Agentic Workflow does with that one! It should be an ideal project since I didn't head an eye on this for some time now (lot's of renovate PRs open, Spring Boot 3.5 - while 4.2 is the latest release etc.).





# Links

https://docs.github.com/en/copilot/concepts/agent-capabilities/agent-powered-workflows-and-operations/about-github-agentic-workflows

https://github.blog/ai-and-ml/automate-repository-tasks-with-github-agentic-workflows/

https://agenticpipeline.dev/resources/ci-cd-secrets/