---
layout: post
title: Entra Token Protection in VSCode Dev Containers
comments: true
---

A previous post on [Entra Token Protection]({% post_url 2026-03-08-entra-token-protection %}) noted that broker-based authentication requires a windowing system, which Docker containers lack. This post presents a workaround: run the authentication broker on the host inside a local HTTP server that mimics the [Azure App Service managed-identity endpoint](https://learn.microsoft.com/en-us/azure/app-service/overview-managed-identity), and have [Azure Identity](https://pypi.org/project/azure-identity/)'s `ManagedIdentityCredential` in the container call that endpoint via Docker's `host.docker.internal` hostname. The post covers the HTTP server and the [VSCode Dev Container](https://code.visualstudio.com/docs/devcontainers/containers) configuration.

# Local HTTP Server

The host-side server wraps `InteractiveBrowserBrokerCredential` from [azure-identity-broker](https://github.com/Azure/azure-sdk-for-python/tree/main/sdk/identity/azure-identity-broker) in Python's built-in `HTTPServer`. It listens on `http://127.0.0.1:40342` and responds to `GET /oauth2/token` requests with a JSON body containing `access_token` and `expires_on`, matching the response format of the [App Service managed-identity token endpoint](https://learn.microsoft.com/en-us/azure/app-service/overview-managed-identity#rest-endpoint-reference). The `resource` query parameter and `X-IDENTITY-HEADER` request header are forwarded from the container request to select the token scope and validate the caller.

The full server implementation is at [auth_broker_server.py](https://github.com/mmaitre314/auth-broker-server/blob/main/auth_broker_server.py).

# Dev Container

The Dev Container configuration installs `azure-identity` and sets the `IDENTITY_ENDPOINT` and `IDENTITY_HEADER` environment variables that `ManagedIdentityCredential` reads at runtime. `IDENTITY_ENDPOINT` uses `host.docker.internal` to route token requests from the container to the host-side HTTP server.

{% highlight json %}
{
    "name": "AuthBrokerServerDemo",
    "image": "mcr.microsoft.com/devcontainers/python:3",
    "remoteEnv": {
        "IDENTITY_ENDPOINT": "http://host.docker.internal:40342/oauth2/token",
        "IDENTITY_HEADER": "AuthBrokerServer"
    },
    "postCreateCommand": "pip install azure-identity",
    "customizations": {
        "vscode": {
            "extensions": [ "ms-python.python" ]
        }
    }
}
{% endhighlight %}

Code running in the container then acquires device-bound tokens without any broker or windowing dependency:

{% highlight python %}
from azure.identity import ManagedIdentityCredential

credential = ManagedIdentityCredential()
token = credential.get_token("https://graph.microsoft.com/.default").token
{% endhighlight %}

# GitHub Repo

The companion repo with the full code sample is at [github.com/mmaitre314/auth-broker-server](https://github.com/mmaitre314/auth-broker-server).
