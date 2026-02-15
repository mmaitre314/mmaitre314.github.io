---
layout: post
title: Authoring PySpark Notebooks with GitHub Copilot
comments: true
---

[Apache Spark](https://spark.apache.org/) is great for data processing at scale. [GitHub Copilot](https://github.com/features/copilot) agents are great at writing data-processing code. Getting the two to work together can be challenging though. Agents work best when they can validate the code they write, running it locally, executing tests, and iterating on failures. At the same time, setting up a local Spark cluster can be complex and time-consuming. [VSCode Dev Containers](https://code.visualstudio.com/docs/devcontainers/containers) bridge the gap: they streamline local Spark setup so agents can author PySpark notebooks that are then deployed to larger clusters.

Two things are needed: a Dev Container config to provide the Spark runtime, and agent instructions to guide code generation and testing.

# Dev Container

Dev Containers provide both a sandbox to prevent agents from damaging the host machine and the tools agents need to be successful (local Spark cluster, pytest, Jupyter notebooks, etc.).

Create `.devcontainer/Dockerfile`. Most of the Spark setup is handled by Spark's [official Docker image](https://hub.docker.com/_/spark). On top of it, add git, set Python environment variables, and install packages for testing, data-processing, and notebooks.

{% highlight docker %}
FROM spark:4.0.1-scala2.13-java21-python3-ubuntu

USER root

RUN apt-get update && apt-get install -y git
RUN pip3 install pytest==9.0.2 ipykernel==7.1.0 chispa==0.11.1 delta-spark==4.0.1

ENV PYTHONPATH=$SPARK_HOME/python:$SPARK_HOME/python/lib/py4j-0.10.9.9-src.zip
ENV PYSPARK_PYTHON=/usr/bin/python3
ENV PYSPARK_DRIVER_PYTHON=/usr/bin/python3
{% endhighlight %}

Then add `.devcontainer/devcontainer.json` to reference the Dockerfile, configure VSCode extensions for Python, Copilot, and notebooks, and enable auto-approval of agent actions within the container sandbox (sandboxed YOLO).

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

# Agent Instructions

Agents perform best when they can validate their code and correct errors. Providing guidance via an [AGENTS.md](https://agents.md/) file or a spec helps, especially for patterns models may not have encountered often during training.

For instance, starting a local Spark session with [Delta Lake](https://delta.io/) support is not completely trivial. A short code snippet in the instructions goes a long way:

{% highlight python %}
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

Agents are good at writing [pytest](https://docs.pytest.org/) tests but less familiar with running notebooks from within them. It helps to remind agents that `.ipynb` files are JSON: load them, loop through the code strings at JSONPath `$.cells[*].source`, and execute each one via `exec()`.

It is also useful to instruct agents to group notebook parameters (input/output paths, Spark session, etc.) into the first cell. This makes them easy to override in tests, allowing input/output data paths to be replaced with temporary paths and a shared Spark session to be reused across tests for faster execution.

For a complete working example, see the companion repo at [github.com/mmaitre314/pyspark-ai](https://github.com/mmaitre314/pyspark-ai).
