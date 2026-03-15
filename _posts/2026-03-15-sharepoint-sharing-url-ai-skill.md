---
layout: post
title: SharePoint Sharing-URL AI Skill
comments: true
---

Organizations accumulate large volumes of knowledge in [SharePoint](https://www.microsoft.com/en-us/microsoft-365/sharepoint/collaboration) as Word, Excel, PowerPoint, and other Office documents. Troubleshouting Guides (TSGs), Standard Operating Procedures (SOPs), specs, checklists, etc. are invaluable for AI agents to perform their tasks, as long as they are able to download them and convert them to a format they understand. This post goes over the key steps needed to create a skill that takes a SharePoint sharing URL, downloads the file via [Microsoft Graph](https://learn.microsoft.com/en-us/graph/overview), and converts it to Markdown using [MarkItDown](https://github.com/microsoft/markitdown).

# Dependencies

Start by adding the following Python packages to `requirements.txt` (or equivalent):

{% highlight text %}
azure-identity
azure-identity-broker
pywin32 ; sys_platform == "win32"
markitdown[docx]
requests
{% endhighlight %}

[azure-identity-broker](https://github.com/Azure/azure-sdk-for-python/tree/main/sdk/identity/azure-identity-broker) handles [broker-based authentication]({% post_url 2026-03-08-entra-token-protection %}) needed when [Entra Token Protection](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-token-protection) is enabled on Microsoft 365.

# Sharing-URL Encoding

Clicking the 'Share' button in Word/Excel/etc. produces a Sharing URL such as this when the documents are stored in SharePoint:

{% highlight text %}
https://contoso.sharepoint.com/:w:/r/teams/engineering/Shared%20Documents/Report.docx?d=w1234&csf=1
{% endhighlight %}

The [Shares API](https://learn.microsoft.com/en-us/graph/api/shares-get) in Microsoft Graph accepts these URLs after encoding them into a sharing token using Base64:

{% highlight python %}
from base64 import urlsafe_b64encode

sharing_token = "u!" + urlsafe_b64encode(url.encode("utf-8")).decode("ascii").rstrip("=")
{% endhighlight %}

# Authentication

Authenticating using an auth broker and the Microsoft Office client ID (`d3590ed6-52b3-4102-aeff-aad2292ab01c`) ensures sufficient permissions on SharePoint files:

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

# Download

Download the file content using the sharing token and the MS Graph `GET /shares/driveItem/content` endpoint:

{% highlight python %}
import requests

r = requests.get(
    f"https://graph.microsoft.com/v1.0/shares/{sharing_token}/driveItem/content",
    headers={"Authorization": f"Bearer {access_token}"},
)
with open(path, "wb") as f:
    f.write(r.content)
{% endhighlight %}

Behind the scenes, `requests` automatically follows a `302 Found` redirect to a pre-authenticated download URL in the `Location` header—no extra authorization handling needed.

# Convert to Markdown

MarkItDown converts the downloaded file to Markdown, ready for consumption by AI agents:

{% highlight bash %}
markitdown Report.docx > Report.md
{% endhighlight %}

# GitHub Repo

The companion repo with the full code sample is at [github.com/mmaitre314/m365-skill](https://github.com/mmaitre314/m365-skill).
