---
layout: post
title: Entra token protection
comments: true
---

Entra released [Token Protection](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-token-protection) to tackle access-token theft and ensure access tokens are tied to the machine they were requested from. This is nice but creates some [headaches](https://github.com/Azure/azure-cli/issues/31030). In particular, this breaks browser-based log-in flows. For instance, trying to authenticate with [Microsoft Graph](https://learn.microsoft.com/en-us/graph/overview) with this

{% highlight python %}
from azure.identity import InteractiveBrowserCredential

credential=InteractiveBrowserCredential()
access_token=credential.get_token('https://graph.microsoft.com/.default').token
{% endhighlight %}

May now show an error pop-up

> **Sorry, a security policy is preventing access**
>
> An organization security policy requiring token protection is preventing this application from accessing the resource. You may be able to use a different application.

and then throw `Authentication failed: access_denied` (see [this notebook](https://github.com/mmaitre314/brokered-auth/blob/main/auth.ipynb) for more details).

The solution is to replace browser-based auth by [broker-based](https://github.com/Azure/azure-sdk-for-python/tree/main/sdk/identity/azure-identity-broker) auth.

First, install a couple of extra packages in `requirements.txt` (or equivalent):

{% highlight python %}
azure-identity
azure-identity-broker
pywin32 ; sys_platform == "win32"
{% endhighlight %}

Then replace `InteractiveBrowserCredential` by `InteractiveBrowserBrokerCredential` and get a hold of a window handle that will be used to show the log-in pop-up:

{% highlight python %}
from azure.identity.broker import InteractiveBrowserBrokerCredential

if sys.platform == "win32":
    import win32gui
    window_handle=win32gui.GetForegroundWindow()
else:
    import msal
    window_handle=msal.PublicClientApplication.CONSOLE_WINDOW_HANDLE

credential = InteractiveBrowserBrokerCredential(parent_window_handle=window_handle)
access_token=credential.get_token('https://graph.microsoft.com/.default').token
{% endhighlight %}

One issue remaining is that this requires being able to show a window, which is not easily done within Docker containers, including VSCode Dev Containers.