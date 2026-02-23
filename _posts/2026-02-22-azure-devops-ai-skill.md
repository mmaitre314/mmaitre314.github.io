---
layout: post
title: Azure DevOps AI Skill
comments: true
---

AI agents have become very effective at writing their own tools. Adding a touch of markdown, i.e. making them [AI Skills](https://code.visualstudio.com/docs/copilot/customization/agent-skills), helps agents reuse the tools in future sessions and provides an alternative to [MCP tools](https://code.visualstudio.com/docs/copilot/customization/mcp-servers).

This blogs goes through creating an AI Skill for [Azure DevOps](https://learn.microsoft.com/en-us/devops/what-is-devops) (ADO), as an alternative to using the [ADO MCP server](https://github.com/microsoft/azure-devops-mcp). Associated GH repo with the AI skill: https://github.com/mmaitre314/azure-devops-skill

AI Skills have a couple of advantages in that scenario:
- Reduced context pollution. Ex: Azure DevOps provides a large number of tools https://github.com/microsoft/azure-devops-mcp/blob/main/docs/TOOLSET.md which impacts context usage https://github.com/microsoft/azure-devops-mcp/issues/755 while not being complete https://github.com/microsoft/azure-devops-mcp/issues/338#issuecomment-3110831303 . The tool response ends up in the LLM context, creating challenges to enable code reviews for instance https://github.com/microsoft/azure-devops-mcp/issues/635#issuecomment-3513091091 . AI skills enable writing response to disc instead of context, which is can be further processed before being loaded in context.
- Faster iterations. When the agent struggles because of an ill-adapted tool or tool bug, the agent can be asked to reflect one it and improve the skill on the spot. This enables self-improvement, where agents learn from their action and use skills as a form of memory to optimize future sessions. This avoids having to post a feature request and wait for it to be implemented.

To create the AI skill, start with a mini-spec giving an overview of what the skill should do, add a few API-doc pointers, personal preferences, and cover some gotchas (authentication for instance):
[start-quote]
Create an AI skill to connect to Azure Dev Ops (ADO). Enable things like downloading files from repos, querying diffs from Pull Requests, reading wikis, reading work items, etc. Limit to read-only operations. Do not allow making changes to ADO.

Use the ADO documentation at https://learn.microsoft.com/en-us/rest/api/azure/devops/?view=azure-devops-rest-7.2 to get a list of available operations and their syntax. Also review the list of MCP tools for ADO at https://github.com/microsoft/azure-devops-mcp/blob/main/docs/TOOLSET.md for an (incomplete) list of operations. 

The list of operations is fairly long, so organize the skill in a way that optimizes future AI agent performance and does not unnecessarily pollute agent context.

Use Python as coding language.

Use Azure CLI for authentication. For instance Az CLI allows calling ADO like this:
```bash
az login
az rest --method post --uri "https://vssps.dev.azure.com/{org}/_apis/Tokens/Pats?api-version=6.1-preview" --resource "https://management.core.windows.net/" --body '{"displayName": "DevContainer", "scope": "vso.packaging"}' --headers Content-Type=application/json --query patToken.token --output tsv
```

When writing Python, consider using the `azure-identity` package (`DefaultAzureCredential`, `AzureCliCredential`) instead of running Az CLI directly. Note that it does not cache the access tokens from Az CLI, so consider adding a cache where useful to speed things up.

Authenticate with ADO using Entra/OAuth. Do not use PATs.

Also consider using the `azure-devops-python-api` Python package if helpful. Its documentation is at https://github.com/Microsoft/azure-devops-python-api
[end-quote]

After that, ask the agent to do some work:
[start-quote]
Code review https://myorg.visualstudio.com/myproject/_git/myrepo/pullrequest/12345 . Pay special attention to regressions.
[end-quote]

When done, ask the agent to reflect and improve:
[start-quote]
Reflect on the challenges hit during this code review and improve the Azure DevOps skill.
[end-quote]

Then rinse and repeat...

When creating the ADO skill at https://github.com/mmaitre314/azure-devops-skill, this resulted on the agent making a series of improvements to the skill:
1. With the mini-spec, the agent wrote the bulk of the CLI and its markdown
2. After the first code review, it added support for bulk file download and documented the workflow
3. After the second code review, it automated the workflow

As in previous blogs, Dev Containers can help sandbox the agent and provide dependencies (here Python and Azure CLI). For the ADO skill, the following `.devcontainer/devcontainer.json` was used:
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
