---
layout: post
title: AzureCliCredential access-token caching
comments: true
---

Azure's [`DefaultAzureCredential`](https://learn.microsoft.com/en-us/azure/developer/python/sdk/authentication/overview) is a convenient way to authenticate with Azure REST APIs. It allows the same code to run locally and hosted, abstracting away differences and complexities. The lack of caching on some of the access-token providers which run behind `DefaultAzureCredential`, like `AzureCliCredential` for instance, slows down local development though. At an extra ~3s per HTTP request, this quickly becomes [an](https://github.com/Azure/azure-sdk-for-python/issues/40636)-[noy](https://github.com/Azure/azure-sdk-for-net/issues/32579)-[ing](https://github.com/Azure/azure-sdk-for-go/issues/23533). There are some reasons for the lack of caching: handling multiple users on the same machine with ability to log-off, etc. can be difficult-to-impossible and opens up security concerns. That being said, more often than not development happens on a machine with a single user who does not log off, which makes those non-issues.

In Python, a quick way to cache tokens is to monkey patch the `get_token()` and `get_token_info()` methods like this:

{% highlight Python %}
import json
import time
import azure.identity

def wrap_with_token_cache(object, property_name):
    token_cache = {}
    func = getattr(object, property_name)
    def wrapper(self, *args, **kwargs):
        key = json.dumps(args, sort_keys=True) + json.dumps(kwargs, sort_keys=True)
        token = token_cache.get(key, None)
        if token is None or int(time.time()) >= token.expires_on - 300:
            token = func(self, *args, **kwargs)
            token_cache[key] = token
        return token
    setattr(object, property_name, wrapper)

wrap_with_token_cache(azure.identity.AzureCliCredential, 'get_token')
wrap_with_token_cache(azure.identity.AzureCliCredential, 'get_token_info')
{% endhighlight %}

After this, the rest of the authentication code remains as-is and just starts to run faster:

{% highlight Python %}
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()
access_token = credential.get_token("https://management.azure.com/.default").token
{% endhighlight %}

For benchmarking results and a more complete code sample, see the [token_caching.ipynb](https://github.com/mmaitre314/mmaitre314.github.io/blob/master/download/token_caching.ipynb) notebook.
