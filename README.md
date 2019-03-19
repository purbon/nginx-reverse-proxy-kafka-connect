# Reverse proxy security for Kafka and others

There are many situations were more specific security measures are not available to protect a component, for example this use to happen in the early days of Elasticsearch, when no security options were available.

A common approach in such situations is to put a reverse proxy in front of such component, this could look something like this:

REVERSE_PROXY_IP:PORT -> INSECURE_WEB_API:PORT

the reserve proxy had the main purpose of:

* Filtering allowed IP ranges.
* Filtering allowed REST resources and/or method. For example, do you wanna filter the OPTIONS request verb? the reverse proxy can help.
* Introduce authentication when required.
* Restricting connections and bandwidth available.

This could be achieved by things like NGINX or Apache HTTP (mod proxy), this repo contains a simple example (build using docker) based on NGINX.

### Running the proxy example

To run the proxy example, execute
```bash
docker-compose up -d
```

from the main directory

this will bring all relevant containers up and running. If you run _docker ps -a_ you should see something like:

```bash
{"version":"2.1.0-cp1","commit":"bda8715f42a1a3db","kafka_cluster_id":"PFxU2FYZRyqbITvyx8ETEQ"}%                                                                                                                       ➜  reverse-proxy docker ps -a
CONTAINER ID        IMAGE                                    COMMAND                  CREATED             STATUS              PORTS                                     NAMES
fc3e4dea6b72        tekn0ir/nginx-stream                     "/opt/nginx/sbin/ngi…"   About an hour ago   Up About an hour    80/tcp, 443/tcp, 0.0.0.0:8080->8080/tcp   nginx
09c03b0d253d        confluentinc/cp-kafka-connect:5.1.1      "/etc/confluent/dock…"   About an hour ago   Up About an hour    8083/tcp, 9092/tcp                        reverse-proxy_kafka-connect-cp_1
655b69783271        confluentinc/cp-schema-registry:5.1.1    "/etc/confluent/dock…"   About an hour ago   Up About an hour    0.0.0.0:8081->8081/tcp                    reverse-proxy_schema-registry_1
4039ec9f6218        confluentinc/cp-enterprise-kafka:5.1.1   "/etc/confluent/dock…"   About an hour ago   Up About an hour    0.0.0.0:9092->9092/tcp                    reverse-proxy_kafka_1
81071164e475        confluentinc/cp-zookeeper:5.1.1          "/etc/confluent/dock…"   About an hour ago   Up About an hour    2181/tcp, 2888/tcp, 3888/tcp              reverse-proxy_zookeeper_1
```

In this example, as you can see we use the NGINX proxy to control the access to the Kafka Connect REST api.

## NGINX reverse proxy configs

An example config is available in [nginx/nginx.conf](nginx.conf) and [nginx/stream.conf](server.conf).  

In the _nginx.conf_ we use the stream directive to control the download and upload rate:

```
proxy_download_rate 100k;
proxy_upload_rate   50k;
```

on the other side, inside the  _nginx_connect_proxy.conf_ file we describe more in details how the reverse proxy will work:

```
limit_conn_zone $binary_remote_addr zone=ip_addr:10m;

server {
    listen 8080;
    listen [::]:8080;

    # Filter access by IP
    # ( uncomment and write down the allowed/denied IP ranges )
    deny   192.168.1.2;
    allow  2001:0db8::/32;
    deny   all;

    # How to limite the number of connection per IP
    limit_conn ip_addr 10;

    server_name example.com;

    add_header Allow "GET, POST, HEAD" always;
    if ( $request_method !~ ^(GET|POST|HEAD)$ ) {
	     return 405;
    }

    if ( $request_method = OPTIONS ) {
      return 405;
    }


    location / {
        limit_rate_after 500k;
        proxy_pass http://kafka-connect-cp:18083/;
        proxy_set_header X-Real-IP $remote_addr;
    }
  }
```

in this example server config we see four important sections.

The first part we encounter in the file is how to filter access to the proxy by ip, this list is static, but in NGINX there are options to do that dynamically, see https://docs.nginx.com/nginx/admin-guide/security-controls/blacklisting-ip-addresses/ for more details.

After that we can see an example usage of the _if statement_, in this config we use it to filter the request methods allowed, calling out anything that does not match _GET_, _POST_ or _HEAD_. We can use this statements to filter and access lots of information of every request.

And at the end, inside the _location_ section we could the proxy specific configuration, more details can be found at https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/.


## Links

* NGINX security controls: https://docs.nginx.com/nginx/admin-guide/security-controls/
* NGINX as load balancer: https://docs.nginx.com/nginx/admin-guide/load-balancer/
