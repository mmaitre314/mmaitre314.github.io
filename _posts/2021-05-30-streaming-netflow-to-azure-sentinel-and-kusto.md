---
layout: post
title: Streaming Netflow to Azure Sentinel and Kusto
comments: true
---

Network telemetry provides security analysts with visibility into the actors present on a network, allowing better tracking of adversaries as they enter and move through the network.
To be effective, the telemetry needs to collected from edge devices into data stores where it can be correlated at scale with other signals.
This blog shows how to leverage [Logstash](https://www.elastic.co/logstash) running on [Linux Ubuntu](https://ubuntu.com/) to stream [Netflow/IPFIX](https://en.wikipedia.org/wiki/NetFlow) telemetry to Azure Sentinel for [SIEM+SOAR](https://docs.microsoft.com/en-us/azure/sentinel/overview) and Azure Data Explorer (Kusto) for [long-term storage](https://docs.microsoft.com/en-us/azure/sentinel/store-logs-in-azure-data-explorer).

# Install Logstash

Start by installing Logstash:

{% highlight Bash %}
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
sudo apt-get update
sudo apt-get install logstash
sudo chmod -R 777 /var/log/logstash
sudo chmod -R 777 /var/lib/logstash
cd /usr/share/logstash
sudo bin/logstash-keystore --path.settings /etc/logstash create
{% endhighlight %}

Verify the installation by writing a few log lines to the console:

{% highlight Bash %}
bin/logstash --path.settings /etc/logstash -e 'input { generator { count => 10 } } output { stdout {} }'
{% endhighlight %}

# Send logs to Azure Sentinel

Install the Azure Log Analytics plugin:

{% highlight Bash %}
sudo bin/logstash-plugin install  microsoft-logstash-output-azure-loganalytics
{% endhighlight %}

Store the Log Analytics workspace key in the Logstash key store. The workspace key can be found in Azure Portal under `Azure Sentinel > Settings > Workspace settings > Agents management > Primary key`. While there, also write down the Workspace ID (`workspace_id` below).

{% highlight Bash %}
sudo bin/logstash-keystore --path.settings /etc/logstash add LogAnalyticsKey
{% endhighlight %}

The command prompts for the key.

Create the configuration file `/etc/logstash/generator-to-sentinel.conf`:

{% highlight text %}
input {
    stdin {}
    generator { count => 10 }
}
output {
    stdout {}
    microsoft-logstash-output-azure-loganalytics {
        workspace_id => "<workspace_id>"
        workspace_key => "${LogAnalyticsKey}"
        custom_log_table_name => "TestLogstash"
    }
}
{% endhighlight %}

This will create a table called `TestLogstash_CL` in Azure Sentinel.

Run the pipeline:

{% highlight Bash %}
bin/logstash --debug --path.settings /etc/logstash -f /etc/logstash/generator-to-sentinel.conf
{% endhighlight %}

The pipeline starts by generating 10 rows and then waits for user inputs to send as extra rows.

# Send logs to Azure Data Explorer (Kusto)

Install the Azure Data Explorer plugin:

{% highlight Bash %}
sudo bin/logstash-plugin install  logstash-output-kusto
{% endhighlight %}

Create an AAD application in Azure Portal under `Azure Active Directory > App registrations > New registration`. Write down the Application ID (`app_id` below) and Tenant ID (`app_tenant` below).
Create a new key for that app under `Certificates & secrets > Client secrets > New client secret` and store it in the Logstash key store:

{% highlight Bash %}
sudo bin/logstash-keystore --path.settings /etc/logstash add AadAppKey
{% endhighlight %}

The command prompts for the key.

In Kusto, grant the AAD app ingestor access and initialize the table:

{% highlight text %}
.add database <database> ingestors ('aadapp=<app_id>;<app_tenant>')
.create table TestLogstash (timestamp:datetime, message:string, sequence:long)
.create table TestLogstash ingestion json mapping 'v1' '[{"column":"timestamp","path":"$.@timestamp"},{"column":"message","path":"$.message"},{"column":"sequence","path":"$.sequence"}]'
{% endhighlight %}

Create the configuration file `/etc/logstash/generator-to-kusto.conf`:

{% highlight text %}
input {
    stdin {}
    generator { count => 10 }
}
output {
    stdout {}
    kusto {
        ingest_url => "https://ingest-<cluster>.kusto.windows.net/"
        database => "<database>"
        table => "TestLogstash"
        json_mapping => "v1"
        app_tenant => "<app_tenant>"
        app_id => "<app_id>"
        app_key => "${AadAppKey}"
        path => "/tmp/kusto/%{+YYYY-MM-dd-HH-mm-ss}.txt"
    }
}
{% endhighlight %}

Run the pipeline:

{% highlight Bash %}
bin/logstash --debug --path.settings /etc/logstash -f /etc/logstash/generator-to-kusto.conf
{% endhighlight %}

# Receive logs from Netflow

Netflow logs are collected by [Filebeat](https://www.elastic.co/beats/filebeat) which forwards them to Logstash. For simplicity, below, the output of Logstash is written to the console. This can be replaced by the Log Analytics and Kusto outputs as needed.

## Setup Filebeat

Install Filebeat:

{% highlight Bash %}
sudo apt-get install filebeat
sudo chmod 644 /etc/filebeat/filebeat.yml
sudo mkdir /var/lib/filebeat
sudo mkdir /var/log/filebeat
sudo chmod -R 777 /var/log/filebeat
sudo chmod -R 777 /var/lib/filebeat
cd /usr/share/filebeat
{% endhighlight %}

Have Filebeat listen for NetFlow UDP traffic on `localhost:2055`:

{% highlight Bash %}
sudo filebeat modules enable netflow
{% endhighlight %}

Redirect the output of Filebeat from ElasticSearch to Logstash. In `/etc/filebeat/filebeat.yml`:
- Comment out the section `output.elasticsearch`
- Uncomment the section `output.logstash`

## Run Logstach

Create the configuration file `/etc/logstash/filebeat-to-stdout.conf`:

{% highlight text %}
input {
    beats {
        port => 5044
    }
}
output {
    stdout {}
}
{% endhighlight %}

Run Logstash:

{% highlight Bash %}
bin/logstash --debug --path.settings /etc/logstash -f /etc/logstash/filebeat-to-stdout.conf
{% endhighlight %}

## Run Filebeat

In another terminal, run Filebeat:

{% highlight Bash %}
filebeat run -e
{% endhighlight %}

## Generate mock NetFlow traffic

For quick testing, [nflow-generator](https://github.com/nerdalert/nflow-generator) can be used to generate local NetFlow traffic.

In a third terminal, install and run nflow-generator:

{% highlight Bash %}
wget https://github.com/nerdalert/nflow-generator/raw/master/binaries/nflow-generator-x86_64-linux
chmod 777 nflow-generator-x86_64-linux
./nflow-generator-x86_64-linux -t localhost -p 2055
{% endhighlight %}

# Beyond Netflow

Logstash and its companion Filebeat are not limited to forwarding Netflow/IPFIX: they also support a wide variety of other inputs related to network security, including Zeek, Suricata, Snort, etc. See [Filebeat modules](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-modules.html) for details on how to configure them.

# References

## Versions

Tested using:
- Ubuntu 18.04.3 LTS
- Logstash 7.13.0
- jruby 9.2.16.0 (2.5.7)
- OpenJDK 64-Bit Server VM 11.0.10+9

## To probe further

- [Installing Logstash](https://www.elastic.co/guide/en/logstash/7.13/installing-logstash.html#_apt)
- [Azure Sentinel Logstash connector](https://docs.microsoft.com/en-us/azure/sentinel/connect-logstash)
- [Azure Data Explorer Logstash connector](https://docs.microsoft.com/en-us/azure/data-explorer/ingest-data-logstash)
- [Filebeat NetFlow module](https://www.elastic.co/guide/en/beats/filebeat/7.13/filebeat-module-netflow.html)
- [nflow-generator](https://github.com/nerdalert/nflow-generator)
