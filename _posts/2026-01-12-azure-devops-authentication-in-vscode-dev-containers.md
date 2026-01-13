---
layout: post
title: Azure DevOps authentication in VSCode Dev Containers
comments: true
---

[VSCode Dev Containers](https://code.visualstudio.com/docs/devcontainers/containers) allow sandboxing [GitHub coding agents](https://code.visualstudio.com/docs/copilot/agents/overview) and providing agents with the dependencies and tools they need to be successful. In containers, authentication with private Azure DevOps (ADO) artifact feeds can be challenging though. This blog goes over how to streamline ADO authentication using [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/what-is-azure-cli) when restoring packages from Nuget, Maven, or NPM.

TLDR: the [trick](https://github.com/jongio/azure-cli-awesome/blob/main/create-devops-pat.azcli) is to use Azure CLI to authenticate with ADO and generate Personal Access Tokens (PAT), which can then be used in package-manager configs.

{% highlight shell %}
az login
az rest --method post --uri "https://vssps.dev.azure.com/{org}/_apis/Tokens/Pats?api-version=6.1-preview" --resource "https://management.core.windows.net/" --body '{"displayName": "DevContainer", "scope": "vso.packaging"}' --headers Content-Type=application/json --query patToken.token --output tsv
{% endhighlight %}

> In the code snippets, replace `{org}` with your ADO organization and `{feed}` with your ADO artifact feed.

# Nuget

For [Nuget](https://www.nuget.org/), start by creating a Dev Container config, e.g. `/.devcontainer/devcontainer.json`, that defines a custom `Dockerfile` plus the extensions, settings, etc. of your choice.

{% highlight json %}
{
  "name": "DevContainer",
  "build": { "dockerfile": "Dockerfile" },
  "customizations": {
    "vscode": {
      "extensions": ["ms-dotnettools.csdevkit"],
      "settings": {
        "chat.tools.autoApprove": true,
        "chat.tools.terminal.autoApprove": {"/.*/": true},
        "chat.tools.terminal.ignoreDefaultAutoApproveRules": true
      }
    }
  }
}
{% endhighlight %}

In `/.devcontainer/Dockerfile`, start with the official Dev Container image, install the Azure CLI, and pre-populate a user Nuget config.

{% highlight docker %}
FROM mcr.microsoft.com/devcontainers/dotnet:8.0

RUN curl -sL https://aka.ms/InstallAzureCLIDeb | bash

COPY <<EOF /root/.nuget/NuGet/NuGet.Config
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <add key="{feed}" value="placeholder" />
  </packageSources>
  <packageSourceCredentials>
    <{feed}>
      <add key="Username" value="DevContainer" />
      <add key="ClearTextPassword" value="placeholder" />
    </{feed}>
  </packageSourceCredentials>
</configuration>
EOF
{% endhighlight %}

Finally, create a Bash file at `/.devcontainer/ado-login.sh` that logs-in using Azure CLI, generates a PAT, and sets the PAT in the user Nuget config.

{% highlight shell %}
#!/bin/bash
set -euo pipefail

echo "Logging into Azure..."
az config set core.login_experience_v2=off
az login --allow-no-subscriptions -o tsv

echo "Generating Azure DevOps PAT token for NuGet..."
token=$(az rest --method post --uri "https://vssps.dev.azure.com/{org}/_apis/Tokens/Pats?api-version=6.1-preview" --resource "https://management.core.windows.net/" --body '{"displayName": "DevContainer", "scope": "vso.packaging"}' --headers Content-Type=application/json --query patToken.token --output tsv)
sed -i -E "s|<add key=\"ClearTextPassword\".+?/>|<add key=\"ClearTextPassword\" value=\"$token\" />|" /root/.nuget/NuGet/NuGet.Config
{% endhighlight %}

# Maven

For [Maven](https://maven.apache.org/), follow a similar setup with the user config at `~/.m2/settings.xml`.

{% highlight xml %}
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                              https://maven.apache.org/xsd/settings-1.0.0.xsd">
  <servers>
    <server>
      <id>{feed}</id>
      <username>{org}</username>
      <password>placeholder</password>
    </server>
  </servers>
</settings>
{% endhighlight %}

# NPM

For [NPM](https://www.npmjs.com/), do the same with the user config at `~/.npmrc`, after applying an extra base64-encoding to the PAT by piping it through `| base64 -w0`.

{% highlight c %}
; begin auth token
//{org}.pkgs.visualstudio.com/_packaging/{feed}/npm/registry/:username={org}
//{org}.pkgs.visualstudio.com/_packaging/{feed}/npm/registry/:_password=placeholder
//{org}.pkgs.visualstudio.com/_packaging/{feed}/npm/registry/:email=not used
//{org}.pkgs.visualstudio.com/_packaging/{feed}/npm/:username={org}
//{org}.pkgs.visualstudio.com/_packaging/{feed}/npm/:_password=placeholder
//{org}.pkgs.visualstudio.com/_packaging/{feed}/npm/:email=not used
; end auth token
{% endhighlight %}
