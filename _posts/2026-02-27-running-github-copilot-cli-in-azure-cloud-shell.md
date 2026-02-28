---
layout: post
title: Running GitHub Copilot CLI in Azure Cloud Shell
comments: true
---

When Azure resources do not behave as expected, GitHub Copilot can help find the root cause and propose solutions. For that, it needs access to the Azure resources. An easy way to do just that is to run the GitHub Copilot CLI directly in Azure, using Azure Cloud Shell.

![GitHub Copilot CLI]({{ site.url }}/images/2026-02-27-GHC.png)

# Setup

Start by opening a Bash Azure Cloud Shell https://portal.azure.com/#cloudshell/

Install GHC CLI:
{% highlight bash %}
curl -fsSL https://gh.io/copilot-install | bash
{% endhighlight %}

Log-in with GitHub:
{% highlight bash %}
copilot login
{% endhighlight %}

Start Copilot:
{% highlight bash %}
copilot
{% endhighlight %}

And then ask Copilot to assess the situation.

# Sandboxing

In that setup, GitHub Copilot runs under the user identity. This means it can perform any action the user can perform, including resource deletion, etc. which may not be desired.

Solution: use Privileged Identity Management https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-configure to reduce the user permissions
Access Azure under a low-permission role like Reader https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles
Perform just-in-time elevations to broader roles when needed.

An alternative is to run Copilot in a container with a Managed Identity https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview , using for instance Azure Container Instances https://learn.microsoft.com/en-us/azure/container-instances/container-instances-overview . Copilot's identity will then be separate from the user identity and will start with no access to Azure resources.
