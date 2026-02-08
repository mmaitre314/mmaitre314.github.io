---
layout: post
title: Hosted GitHub Copilot CLI
comments: true
---

[GitHub Copilot](https://github.com/features/copilot) offers powerful agentic orchestration: multi-turn reasoning, built-in tools, file access, context management, subagents, etc. A great tool to develop AI workflows. Until recently though, hosting these workflows in the cloud was not possible: Copilot required both UI and interactive user authentication. The release of the [Copilot CLI](https://github.com/github/copilot-cli) and its [SDK](https://github.com/github/copilot-sdk) changed this. The CLI removed the UI dependency, and the SDK enabled service authentication via [BYOK](https://github.com/github/copilot-sdk/blob/main/docs/auth/byok.md) (Bring Your Own Key). This enabled developing AI workflows locally and deploying them as-is to the cloud.

# Deploy an AI Model in Azure AI Foundry

First, deploy an OpenAI model in [Azure AI Foundry](https://ai.azure.com/). Take the deployment URL (e.g., `https://<account>.openai.azure.com/openai/responses?api-version=2025-04-01-preview`), and extract model parameters for the SDK:

{% highlight json %}
{
    "model": "gpt-5.2-codex",
    "provider": {
        "type": "azure",
        "base_url": "https://<account>.openai.azure.com",
        "wire_api": "responses",
        "azure": {
            "api_version": "2025-04-01-preview"
        }
    }
}
{% endhighlight %}

Grant the [Cognitive Services OpenAI User](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles/ai-machine-learning#cognitive-services-openai-user) RBAC role to both your user account (for local development) and the service [Managed Identity](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview) (for cloud deployment).

# Build the Container

The container needs the Copilot CLI, the Copilot SDK, and during local development the Azure CLI. Azure CLI provides two flavors of interactive user authentication: device code in regular containers and browser pop-up in [VSCode Dev Containers](https://code.visualstudio.com/docs/devcontainers/containers).

{% highlight docker %}
FROM debian:trixie

ARG INSTALL_AZ_CLI=false

RUN DEBIAN_FRONTEND=noninteractive \
    apt-get update && \
    apt-get install -y --no-install-recommends curl python3 python3-pip && \
    if [ "$INSTALL_AZ_CLI" = "true" ]; then curl -fsSL https://aka.ms/InstallAzureCLIDeb | bash; fi && \
    curl -fsSL https://gh.io/copilot-install | bash && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

RUN pip install --no-cache-dir --break-system-packages azure-identity==1.25.1 github-copilot-sdk==0.1.22
{% endhighlight %}

Build and run locally:

{% highlight shell %}
docker build -t copilot-cli --build-arg INSTALL_AZ_CLI=true .
docker run -it copilot-cli
{% endhighlight %}

For VS Code Dev Containers, add `.devcontainer/devcontainer.json`:

{% highlight json %}
{
  "name": "copilot-cli",
  "build": { "dockerfile": "Dockerfile", "args": { "INSTALL_AZ_CLI": "true" } },
  "customizations": {
    "vscode": {
      "extensions": [ "ms-python.python", "github.copilot-chat" ],
      "settings": {
        "python.defaultInterpreterPath": "/usr/bin/python3"
      }
    }
  }
}
{% endhighlight %}

# SDK Wrapper

The SDK's BYOK support accepts bearer tokens, enabling [`DefaultAzureCredential`](https://learn.microsoft.com/en-us/python/api/azure-identity/azure.identity.defaultazurecredential?view=azure-python) to handle auth both locally (Azure CLI) and in the cloud (Managed Identity):

{% highlight python %}
import asyncio
from azure.identity import DefaultAzureCredential
from copilot import CopilotClient

async def main():
    credential = DefaultAzureCredential()

    client = CopilotClient()
    await client.start()
    
    session = await client.create_session({
        "model": "gpt-5.2-codex",
        "provider": {
            "type": "azure",
            "base_url": "https://myendpoint.openai.azure.com",
            "bearer_token": credential.get_token("https://cognitiveservices.azure.com/.default").token,
            "wire_api": "responses",
            "azure": {
                "api_version": "2025-04-01-preview",
            },
        }
    })
    
    response = await session.send_and_wait({"prompt": "Summarize the Iliad in 3 paragraphs."})
    print(f"Response: {response.data.content}")
    
    await session.destroy()
    await client.stop()
    
if __name__ == "__main__":
    asyncio.run(main())
{% endhighlight %}

Caveat: at the time of writing (2/26), the SDK does not support refreshing tokens. So runs are limited to 1h with user tokens and 24h with managed-identity tokens.

Run locally:

{% highlight shell %}
az login
python3 ./main.py
{% endhighlight %}

# Deploy to the Cloud

From here, deploy the container to [Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/overview) (or similar). Assign a Managed Identity to the container app, grant it the Cognitive Services OpenAI User role, and skip `INSTALL_AZ_CLI` in the build args. `DefaultAzureCredential` will automatically pick up the managed identity.
