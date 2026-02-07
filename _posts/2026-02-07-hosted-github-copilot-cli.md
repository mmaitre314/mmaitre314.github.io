---
layout: post
title: Hosted GitHub Copilot CLI
comments: true
---

TODO: intro

GitHub Copilot offers powerful agentic orchestration with multi-turn reasoning, built-in tools, file access, context management, etc. A great tool to develop AI solutions.

Until recently though, it was not possible to take such solutions and host them in the cloud to offer them as a service. The release of Copilot CLI and its associated SDK unblocked this. The former removed the UI and the latter enabled service authentication. With a touch of containers, this allows developing AI workflows locally and then publishing them as-in to the cloud.

TODO: deploy AI model in Azure AI Foundry

Deploy an OpenAI model in Azure AI Foundry [URL chunk, remove name]
Deploy a model on AI foundry. Deployment URL: https://onetiai-swec.openai.azure.com/openai/responses?api-version=2025-04-01-preview -> url -> chop it to make it SDK friendly

{% highlight json %}
{
    "model": "gpt-5.2-codex",
    "provider": {
        "type": "azure",
        "base_url": "https://onetiai-swec.openai.azure.com",
        "wire_api": "responses",
        "azure": {
            "api_version": "2025-04-01-preview",
        },
    }
}
{% endhighlight %}

Ensure user and service (managed identity) have ARM RBAC role Cognitive Services OpenAI User.

TODO: create container

Auth:
		Locally: auth with Az CLI (integrated with VSCode Dev Container)
		Deployed: auth with managed identity

		doc: https://github.com/github/copilot-sdk/blob/main/docs/getting-started.md

Azure identity + copilot CLI + copilot SDK
Az CLI for local auth. In regular container, using device code. In VS Code Dev Container, browser pop-up auth.

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


{% highlight shell %}
docker build -t copilot-cli --build-arg INSTALL_AZ_CLI=true .
docker run -it copilot-cli
{% endhighlight %}

Dev container (browser auth in VSCode)

{% highlight json %}
{
  "name": "copilot-cli",
	"build": { "dockerfile": "Dockerfile", "args": { "INSTALL_AZ_CLI": "true" } },
  "customizations": {
    "vscode": {
      "extensions": [ "ms-python.python", "github.copilot-chat" ],
      "settings": {
        "python.defaultInterpreterPath": "/usr/bin/python3",
        "python.selectInterpreter": "/usr/bin/python3"
      }
    }
  }
}
{% endhighlight %}

> SDK wrapper

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
            "base_url": "https://onetiai-swec.openai.azure.com",
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

Copilot SDK removes the Copilot limitation of user auth
Auth relies on Bring Your Own Key (BYOK), where 'key' should be understood loosely as it also includes bearer tokens.
		doc: https://github.com/github/copilot-sdk/blob/main/docs/auth/byok.md


{% highlight shell %}
az login
python3 ./copilot_cli.py
{% endhighlight %}

From there, deploy container to the cloud services like Azure Container App.









