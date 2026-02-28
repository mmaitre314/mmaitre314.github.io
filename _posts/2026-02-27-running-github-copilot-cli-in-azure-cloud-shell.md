---
layout: post
title: Running GitHub Copilot CLI in Azure Cloud Shell
comments: true
---

When Azure resources misbehave, [GitHub Copilot](https://github.com/features/copilot) can help find the root cause and propose fixes, provided it has access to those resources. An easy way to achieve this is to run the [Copilot CLI](https://github.com/github/copilot-cli) directly in [Azure Cloud Shell](https://learn.microsoft.com/en-us/azure/cloud-shell/overview), where the Azure CLI and user credentials are already available.

![GitHub Copilot CLI]({{ site.url }}/images/2026-02-27-GHC.png)

This post covers the setup steps and a couple of approaches to manage Copilot's Azure permissions.

# Setup

Open a Bash session in [Azure Cloud Shell](https://portal.azure.com/#cloudshell/).

Install the Copilot CLI:

{% highlight bash %}
curl -fsSL https://gh.io/copilot-install | bash
{% endhighlight %}

Log in with GitHub:

{% highlight bash %}
copilot login
{% endhighlight %}

Start an interactive Copilot session:

{% highlight bash %}
copilot
{% endhighlight %}

From there, ask Copilot to assess the situation. It can invoke Azure CLI commands, query resource states, and suggest remediation steps.

# Sandboxing

In this setup, Copilot runs under the user's identity. It can perform any action the user can perform, including resource deletion, which may not be desired.

One approach is to use [Privileged Identity Management](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-configure) (PIM) to limit the blast radius. Access Azure under a low-permission role like [Reader](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#reader) for the Copilot session, and perform just-in-time elevations to broader roles only when needed.

An alternative is to run Copilot in a container with its own [Managed Identity](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview), using for instance [Azure Container Instances](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-overview). Copilot's identity is then separate from the user's identity and starts with no access to Azure resources. Permissions are granted explicitly via RBAC, keeping full control over what Copilot can and cannot do.
