---
layout: post
title: SharePoint Sharing-URL AI Skill
comments: true
---

AI agent download docs from Sharepoint (convert to markdown)
helping AI agents download documents from SharePoint. To take advantage of all the TSGs, SOPs, etc. already written for humans to follow.

TODO: download_sharing_url.py to test code

Sharing URL:

{% highlight python %}
url = https://contoso.sharepoint.com/:w:/r/teams/engineering/Shared%20Documents/Report.docx?d=w1234&csf=1
{% endhighlight %}

// Requirements.txt

{% highlight text %}
azure-identity
azure-identity-broker
pywin32 ; sys_platform == "win32"
markitdown[docx]
{% endhighlight %}

// Sharing URL encoding

{% highlight python %}
from base64 import urlsafe_b64encode

sharing_token = "u!" + urlsafe_b64encode(url.encode("utf-8")).decode("ascii").rstrip("=")
{% endhighlight %}

Doc: https://learn.microsoft.com/en-us/graph/api/shares-get

// Authentication

Use auth broker in case Token Protection is enabled on MS Graph access

Client: Microsoft Office (d3590ed6-52b3-4102-aeff-aad2292ab01c) to have sufficent permissions

{% highlight python %}
from sys import platform
from azure.identity.broker import InteractiveBrowserBrokerCredential

if platform == "win32":
    import win32gui
    window_handle = win32gui.GetForegroundWindow()
else:
    import msal
    window_handle = msal.PublicClientApplication.CONSOLE_WINDOW_HANDLE

credential = InteractiveBrowserBrokerCredential(
    client_id="d3590ed6-52b3-4102-aeff-aad2292ab01c",
    parent_window_handle=window_handle,
    use_default_broker_account=True,
)
access_token = credential.get_token("https://graph.microsoft.com/.default").token
{% endhighlight %}

// Download file

{% highlight python %}
import requests

r = requests.get(
    f"https://graph.microsoft.com/v1.0/shares/{sharing_token}/driveItem/content",
    headers={"Authorization": f"Bearer {access_token}"},
)
with open(path, "wb") as f:
    f.write(r.content)
{% endhighlight %}

Note: behind the scene, `requests` automatically handles a `302 Found` redirect to a preauthenticated download URL (i.e a URL which does not require an `Authorization` header to access) for the file in the `Location` header.

// Convert to Markdown

{% highlight bash %}
markitdown file.docx > file.md
{% endhighlight %}

# GitHub Repo

The companion repo with the full code sample is at [github.com/mmaitre314/m365-skill](https://github.com/mmaitre314/m365-skill).
