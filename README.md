## Ansible Playbook to automate the setup and configuration of a centralized Logstash server and clients

## Introduction

This playbook installs Logstash from the Logstash apt repository. By default it
uses the embedded Elasticsearch supplied with Logstash.

You can set a host variable to force the use of a standalone Elasticsearch
instance if scaling is likely to be required. In this case, the version of 
Elasticsearch must match the version required by Logstash. We could use 
`elasticsearch_http` as the output, which would decouple the Elasticsearch 
version from the Logstash version, but https://logstash.jira.com/browse/LOGSTASH-903
results in lost messages if the Indexer is running but Elasticsearch is not.

Most clients should use Beaver to ship logs to Logstash, by adding them to the
`[beaver]` group in the inventory file. Beaver is used is preference to Logstash
for shipping because of lower memory requirements. In particular, AWS EC2 t1.micro
instances only have 512MB RAM.

For embedded devices or other clients that cannot run Beaver you can set
separate host variables to enable central rsyslog reception for TCP and/or UDP
on the Logstash host. Note that this uses port 6514 instead of the standard 514
because of https://bugs.launchpad.net/ubuntu/+source/rsyslog/+bug/789174
affecting Ubuntu 12.04.4 LTS

**Platform**: Tested on **Ubuntu 12.04.4LTS**

**Disclaimer**: Do not put a  live production system into the `logstash` group!!
Use a dedicated instance for Logstash and put clients into the `beaver` group
only.

**Prerequisites**: The Logstash server should have at least 1GB and preferably 2GB RAM.

**Logging Logic**: 

Normal Clients run Beaver (managed by `supervisord`) to store logs as json in
a Redis database on the Logstash Server, accessed via an SSH tunnel.

Embedded Clients send remote `syslog` messages to `rsyslog` running on the Logstash
server.

A Logstash indexer process running as part of the default `logstash` service
reads data from redis, and from `/var/log` on the local server, and processes it
and then stores it in Elasticsearch.

A Logstash Web UI process running as part fo the default `logstash-web` service
provides the user interface to Elasticseach

### Preparation

1. Copy `hosts.example` to a new `hosts` inventory file
1. Setup your target Logstash server in `[logstash]` in the `hosts` inventory file
2. Customize Logstash filters (if needed) in roles/logstash/templates/logstash_indexer.conf.j2
3. Set the `logstash_hostname` variable in `[beaver:vars]` in the `hosts` inventory file
4. Setup your clients in `[beaver]` in the `hosts` inventory file
5. Configure any embedded hosts to send `syslog` messages to port 6514 on the
Logstash server and set `rsyslog_receive_tcp` or `rsyslog_receive_udp` in
`[logstash:vars]` in the `hosts` inventory file

### Install

`ansible-playbook site.yml --inventory-file=./hosts`


## Use

Browse **http://{{ logstash_hostname }}:9292** and happy logging!

## Trouble-shooting

### Beaver

If Beaver is not shipping logs, then please test the following:

* The most common error is that the SSH key in `/etc/beaver/id_rsa` on the client
is not authorized to connect to `beaver@{{ logstash_hostname }}`. You can test
this by `ssh`ing into the client and then `ssh`ing from the client to `beaver@logstash`,
but you must make sure that Agent Forwarding is not enabled on the connection
to the client, otherwise you could be connecting using your own credentials
rather than `/etc/beaver/id_rsa`. The best way to do this is to explicitly
disable Agent Forwarding in the initial connection using `ssh -a`:

    ```
CLIENT=client.example.com
LOGSTASH=logstash.example.com
ssh -a $USER@$CLIENT sudo ssh -i /etc/beaver/id_rsa beaver@$LOGSTASH pwd
    ```

* Assuming that the SSH connection appears to be working, then check that `beaver`
is running and that there are processes running for both `beaver` and the `ssh`
tunnel:

    ```
sudo supervisorctl status
sudo ps -ef | grep beaver
    ```

* If `beaver` is running, then the Logstash `redis` instance should be available
for querying (it is OK for the result of the query to be 0, if the Logstash
indexer is working properly then it should be removing messages from the list
as fast as `beaver` is placing them on it:

    ```
redis-cli -p 6380 llen logstash
    ```

* If `beaver` is running and the `redis` instance is accessible, then you should
check that messages placed on the queue are received successfully. The easiest
way to do that is to temporarily disable the Logstash Indexer on the Logstash 
server (e.g. `sudo service logstash stop`). Then new messages should show up 
as new entries on the queue and should be able to be retrieved from the Logstash
`redis` instance:

    ```
# Check the current message count
redis-cli -p 6380 llen logstash
# Create a message
logger "This is a test message for Logstash"
sleep 2
# Check that the current message count has increased
redis-cli -p 6380 llen logstash
# Remove the test message from the queue to make sure the text is correct
redis-cli -p 6380 rpop logstash
    ```

### Indexer

Check if the various services are running

```
sudo service redis-server status
sudo service logstash status
sudo service logstash-web status
```

Check the number of records in the Redis cache:

```
redis-cli llen logstash
```

Check the that the Elasticsearch indexes exist:

```
curl -s http://127.0.0.1:9200/_status?pretty=true | grep logstash
```

Check the number of records in the index:

```
    curl -s http://127.0.0.1:9200/_count?pretty=true
```

Check a query against the index:

```
    curl -gs -XGET http://localhost:9200/logstash-*/_search?pretty=true&q=type=syslog
```

Check that the necessary ports are open:

```
sudo netstat -napt | grep -i LISTEN

tcp        0      0 127.0.0.1:6379          0.0.0.0:*               LISTEN      879/redis-server
tcp6       0      0 :::9292                 :::*                    LISTEN      4053/java       
tcp6       0      0 :::9200                 :::*                    LISTEN      982/java        
tcp6       0      0 :::9300                 :::*                    LISTEN      982/java        
tcp6       0      0 :::9301                 :::*                    LISTEN      3250/java       

### Building Filters

The heart of Logstash is the filtering it does to extract meaningful data
from messages. If you change the configuration and include a bad filter then
Logstash will stop processing logs. You should always test your filters before
releasing them to the production Logstash server. The two best ways to test them
are:

* http://grokdebug.herokuapp.com/
* `java -jar /opt/logstash/logstash.jar agent -e`. For example:

    ```
FILTER='grok {
             match => [
                 "message", "%{NOTSPACE:vhost} %{COMBINEDAPACHELOG}",
                 "message", "%{COMBINEDAPACHELOG}"
             ]
             add_field => [ "program", "%{type}" ]
         }'
java -jar /opt/logstash/logstash.jar agent -e "filter { $FILTER }"
    ```

### TODO

* Add support for Statsd and Librato
* Rsyslog-server role can be extended with TLS support. See http://www.rsyslog.com/doc/rsyslog_tls.html
* Redis password authentication
* Run Beaver as an unprivileged user (with adm and root secondary groups)
* Add Apache or Nginx as a reverse proxy using pam authentication in combination
with `ufw` to block port 9292

### Links

* [Ansible](http://www.ansibleworks.com/)
* [Logstash](http://www.logstash.net/)
* [Elasticsearch](http://www.elasticsearch.org/)
* [Redis](http://redis.io/)
* [Supervisord](http://supervisord.org/)
* [Ansible Fan Community](https://plus.google.com/u/0/communities/108222183653550371543)
