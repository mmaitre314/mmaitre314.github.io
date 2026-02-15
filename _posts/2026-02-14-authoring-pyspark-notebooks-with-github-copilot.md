---
layout: post
title: Authoring PySpark Notebooks with GitHub Copilot
comments: true
---

Apache Spark and AI Agents are both powerfull technologies in their own right. Spark is great for data processing at scale. AI Agents are great at writing data-processing code. Getting the two to work together can be a bit challenging though. AI agents work best when they can validate the code they write, running it locally, executing tests, and iterating on failures. At the same time, setting up a Spark cluster to run locally can be both complex and time consumming. This is where VSCode Dev Containers can help bridge the gap, streamlining local Spark setup for agents to author data-processing Python notebooks that can then be deployed to larger Spark clusters.

Start by setting up the Dev Container and then create instructions to guide agents (instructions we may become unnecessary if models get trained on this blog...).

# Dev Container

Dev Containers provide both a sandbox to prevent agents from damaging the host machine and the set of tools that agents need to be successful (local Spark cluster, pytest tests, Jupyter notebooks, etc.).

Start by creating `.devcontainer/Dockerfile`. Most of the Spark setup is handled by Spark's [official Docker image](https://hub.docker.com/_/spark). We only need to add some tools like git, set a few environment variables for Python, and add a few Python packages for testing, data-processing, and notebooks.

{% highlight docker %}
FROM spark:4.0.1-scala2.13-java21-python3-ubuntu

USER root

RUN apt-get update && apt-get install -y git
RUN pip3 install pytest==9.0.2 ipykernel==7.1.0 chispa==0.11.1 delta-spark==4.0.1

ENV PYTHONPATH=$SPARK_HOME/python:$SPARK_HOME/python/lib/py4j-0.10.9.9-src.zip
ENV PYSPARK_PYTHON=/usr/bin/python3
ENV PYSPARK_DRIVER_PYTHON=/usr/bin/python3
{% endhighlight %}

Then add `.devcontainer/devcontainer.json` to reference the Dockerfile, set a few VSCode settings for Python and Copilot (making AI agent YOLO within the boundary of the container sandbox), and add a few VSCode extensions for Python, Copilot, and notebooks.

{% highlight json %}
{
  "name": "pyspark-ai",
  "build": { "dockerfile": "Dockerfile" },
  "remoteUser": "root",
  "customizations": {
    "vscode": {
      "extensions": [ "github.copilot-chat", "ms-python.python", "ms-toolsai.jupyter" ],
      "settings": {
        "chat.tools.autoApprove": true,
	    "chat.tools.terminal.autoApprove": { "/.*/": true },
        "chat.tools.terminal.ignoreDefaultAutoApproveRules": true,
        "chat.tools.edits.autoApprove": { "**/*": true, "**/.git/**": false },
        "chat.agent.maxRequests": 100,
        "python.defaultInterpreterPath": "/usr/bin/python3",
        "python.selectInterpreter": "/usr/bin/python3"
      }
    }
  }
}
{% endhighlight %}

# Agent instructions

Agents perform best when they can validate their code and correct errors.

guide agents through instructions, be it a spec, an AGENTS.md, etc. Give them hints when they need to rely on patterns they might not have seen very often during their training.

For instance, starting a local Spark session with support for Delta tables is not completely trivial, so giving a short code snippet helps: 

{% highlight json %}
from pyspark.sql import SparkSession

spark = (
    SparkSession.builder
    .master('local[*]')
    .config('spark.jars.packages', 'io.delta:delta-spark_2.13:4.0.1')
    .config('spark.sql.extensions', 'io.delta.sql.DeltaSparkSessionExtension')
    .config('spark.sql.catalog.spark_catalog', 'org.apache.spark.sql.delta.catalog.DeltaCatalog')
    .getOrCreate()
)
{% endhighlight %}

For testing, agents are very good at writting Pytest tests but less used to run notebooks from within such tests. It helps to remind agents that .ipynb notebook file are first and foremost JSON files, which can be loaded as JSON and executed by looping through JSON code strings at `$.cells[*].source` and executing them one by one.

It is also helpful to suggest agents group the notebook parameters into the first notebook cell, so that they are easy to replace when running tests. This allows replacing input/output data paths with temporary paths, or reusing a Spark session across tests to speed things up.

For more details, see the companion GitHub repo at https://github.com/mmaitre314/pyspark-ai .
