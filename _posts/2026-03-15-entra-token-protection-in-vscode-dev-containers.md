---
layout: post
title: Entra Token Protection in VSCode Dev Containers
comments: true
---

This blog is a follow up on [Entra token protection]({% post_url 2026-03-08-entra-token-protection %}) with a solution to get device-bound tokens inside VSCode Dev Containers. The challenge with Entra token protection is Docker containers are typically console-based and do not have a a windowing system while token protection relies on authentication brokers which require WebKitGTK and X11. One solution is to run the authentication broker on the host, wrap it in an HTTP server, make the HTTP server behave like the Azure App Service managed-identity endpoint, and have Azure Identity code running in the container call it that endpoint.

// Local HTTP Server

Wrap Azure Identity's `InteractiveBrowserBrokerCredential` in Python' `HTTPServer` so it responds to `GET http://127.0.0.1:40342/oauth2/token` requests with a JSON with two properties `access_token` and `expires_on`.

For details, see https://github.com/mmaitre314/auth-broker-server/blob/main/auth_broker_server.py

// Dev Container

The define a VSCode Dev Container config. Install `azure-identity` to get access to `ManagedIdentityCredential`. Set the `IDENTITY_ENDPOINT` and `IDENTITY_HEADER` environment variables of Azure App Service managed identity, and in `IDENTITY_ENDPOINT` leverage Docker's `host.docker.internal` hostname to send requests from the container to the host.

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

# GitHub Repo

A companion repo with the full code sample is available at [github.com/mmaitre314/auth-broker-server](https://github.com/mmaitre314/auth-broker-server).
