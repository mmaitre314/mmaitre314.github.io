---
layout: post
title: Entra token protection
comments: true
---

[Microsoft Entra](https://learn.microsoft.com/en-us/entra/fundamentals/what-is-entra) released [Token Protection](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-token-protection), a conditional-access policy that binds access tokens to the device they were issued on to harden against token theft. This is a welcome security improvement which brings some [headaches](https://github.com/Azure/azure-cli/issues/31030). In particular, it breaks browser-based interactive login flows. The fix is to switch to [broker-based authentication](https://github.com/Azure/azure-sdk-for-python/tree/main/sdk/identity/azure-identity-broker), which delegates login to the OS authentication broker (Web Account Manager on Windows, Company Portal on macOS, Microsoft Identity Broker on Linux). This post covers the error and the fix.

# The Error

Trying to authenticate with [Microsoft Graph](https://learn.microsoft.com/en-us/graph/overview) using `InteractiveBrowserCredential` from the [Azure Identity](https://pypi.org/project/azure-identity/) library:

{% highlight python %}
from azure.identity import InteractiveBrowserCredential

credential = InteractiveBrowserCredential()
access_token = credential.get_token('https://graph.microsoft.com/.default').token
{% endhighlight %}

now shows an error pop-up when Token Protection is enabled:

> **Sorry, a security policy is preventing access**
>
> An organization security policy requiring token protection is preventing this application from accessing the resource. You may be able to use a different application.

followed by an `Authentication failed: access_denied` exception.

# Broker-Based Authentication

The [azure-identity-broker](https://github.com/Azure/azure-sdk-for-python/tree/main/sdk/identity/azure-identity-broker) package delegates authentication to the OS broker, which produces device-bound tokens that satisfy Token Protection. Add it along with [pywin32](https://pypi.org/project/pywin32/) (needed on Windows to retrieve a window handle) in `requirements.txt` (or equivalent):

{% highlight text %}
azure-identity
azure-identity-broker
pywin32 ; sys_platform == "win32"
{% endhighlight %}

Then replace `InteractiveBrowserCredential` with `InteractiveBrowserBrokerCredential`, passing it a window handle for the login popup:

{% highlight python %}
import sys
from azure.identity.broker import InteractiveBrowserBrokerCredential

if sys.platform == "win32":
    import win32gui
    window_handle = win32gui.GetForegroundWindow()
else:
    import msal
    window_handle = msal.PublicClientApplication.CONSOLE_WINDOW_HANDLE

credential = InteractiveBrowserBrokerCredential(parent_window_handle=window_handle)
access_token = credential.get_token('https://graph.microsoft.com/.default').token
{% endhighlight %}

One open issue: broker-based authentication requires a windowing system, which is not available in Docker containers, including [VSCode Dev Containers](https://code.visualstudio.com/docs/devcontainers/containers). If someone has a solution, potentially using VSCode's built-in authentication proxy, I am interested...

# GitHub Repo

The companion notebook with the full code sample is at [github.com/mmaitre314/brokered-auth](https://github.com/mmaitre314/brokered-auth).