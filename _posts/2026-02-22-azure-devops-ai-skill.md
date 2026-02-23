---
layout: post
title: Azure DevOps AI Skill
comments: true
---

AI agents have become very effective at writing their own tools. Adding a touch of markdown, i.e. making them [AI Skills](https://code.visualstudio.com/docs/copilot/customization/agent-skills), helps agents reuse those tools in future sessions and provides an alternative to [MCP tools](https://code.visualstudio.com/docs/copilot/customization/mcp-servers).

This post walks through creating an AI Skill for [Azure DevOps](https://learn.microsoft.com/en-us/devops/what-is-devops) (ADO), as an alternative to the [ADO MCP server](https://github.com/microsoft/azure-devops-mcp). AI Skills have a couple of advantages in that scenario:
- **Reduced context pollution.** The ADO MCP server exposes a [large number of tools](https://github.com/microsoft/azure-devops-mcp/blob/main/docs/TOOLSET.md) which [impacts context usage](https://github.com/microsoft/azure-devops-mcp/issues/755), while still [not being complete](https://github.com/microsoft/azure-devops-mcp/issues/338#issuecomment-3110831303). Tool responses end up in the LLM context, creating challenges for use cases like [code review](https://github.com/microsoft/azure-devops-mcp/issues/635#issuecomment-3513091091). AI Skills can write responses to disk instead of into context, enabling further processing before loading into context.
- **Faster iterations.** When the agent struggles because of an ill-adapted tool or a tool bug, it can be asked to reflect and improve the skill on the spot. This enables self-improvement, where agents learn from their actions and use skills as a form of long-term memory to optimize future sessions. No need to file a feature request and wait for it to be implemented.

The create the skill, start with a mini-spec, ask the agent to do some work, ask it to reflect and improve, and repeat.

# Bootstrapping the Skill

To create the skill, start with a mini-spec giving an overview of what the skill should do, a few API-doc pointers, personal preferences, and some gotchas (authentication for instance):

> Create an AI skill to connect to Azure Dev Ops (ADO). Enable things like downloading files from repos, querying diffs from Pull Requests, reading wikis, reading work items, etc. Limit to read-only operations. Do not allow making changes to ADO.
>
> Use the [ADO REST API documentation](https://learn.microsoft.com/en-us/rest/api/azure/devops/?view=azure-devops-rest-7.2) to get a list of available operations and their syntax. Also review the [list of MCP tools for ADO](https://github.com/microsoft/azure-devops-mcp/blob/main/docs/TOOLSET.md) for an (incomplete) list of operations.
>
> The list of operations is fairly long, so organize the skill in a way that optimizes future AI agent performance and does not unnecessarily pollute agent context.
>
> Use Python as coding language.
>
> Use Azure CLI for authentication. When writing Python, consider using the `azure-identity` package (`DefaultAzureCredential`, `AzureCliCredential`) instead of running Az CLI directly. Note that it does not cache the access tokens from Az CLI, so consider adding a cache where useful to speed things up.
>
> Authenticate with ADO using Entra/OAuth. Do not use PATs.
>
> Also consider using the [`azure-devops-python-api`](https://github.com/Microsoft/azure-devops-python-api) package if helpful.

# Iterating

After bootstrapping, ask the agent to do some work:

> Code review `https://myorg.visualstudio.com/myproject/_git/myrepo/pullrequest/12345`. Pay special attention to regressions.

When done, ask the agent to reflect and improve:

> Reflect on the challenges hit during this code review and improve the Azure DevOps skill.

Then rinse and repeat.

When creating the [ADO skill repo](https://github.com/mmaitre314/azure-devops-skill), this resulted in the agent making a series of improvements:

1. With the mini-spec, the agent wrote the bulk of the CLI and its markdown.
2. After the first code review, it added support for bulk file download and documented the workflow.
3. After the second code review, it automated the workflow.

# Dev Container

As in [previous posts](/2026/02/14/authoring-pyspark-notebooks-with-github-copilot/), [VSCode Dev Containers](https://code.visualstudio.com/docs/devcontainers/containers) help sandbox the agent and provide dependencies (here Python and [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/what-is-azure-cli)).

{% highlight json %}
{
  "name": "Python",
  "image": "mcr.microsoft.com/devcontainers/python:3.13",
  "postCreateCommand": "curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash",
  "customizations": {
    "vscode": {
      "extensions": [ "ms-python.python", "github.copilot-chat" ],
      "settings": {
        "python.defaultInterpreterPath": "/usr/local/bin/python",
        "chat.agent.autoApprove": true,
        "chat.agent.maxRequests": 100,
        "chat.tools.edits.autoApprove": { "**/*": true, "**/.git/**": false },
        "chat.tools.terminal.autoApprove": { "/.*/": true, "**/.git/**": false },
        "chat.tools.terminal.ignoreDefaultAutoApproveRules": true
      }
    }
  }
}
{% endhighlight %}

# GitHub Repo

The companion repo with the full AI skill is at [github.com/mmaitre314/azure-devops-skill](https://github.com/mmaitre314/azure-devops-skill).
