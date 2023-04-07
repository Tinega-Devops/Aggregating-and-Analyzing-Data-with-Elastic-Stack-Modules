# Aggregating-and-Analyzing-Data-with-Elastic-Stack-Modules

## ABOUT THIS LAB

The Elastic Stack provides a plethora of Beat clients to collect and ship all kinds of data. Furthermore, each Beat client also utilizes modules that come pre-packaged with all the configurations, Elasticsearch index templates, ingest pipelines, and Kibana dashboards. Using these modules allows anyone to quickly get up and running with the Elastic Stack. In this hands-on lab, you will deploy and configure a three-node Elasticsearch cluster; generate and deploy Elasticsearch node certificates; encrypt the Elasticsearch transport cluster; enable user authentication and set built-in user passwords; deploy and configure Kibana to connect to Elasticsearch; deploy and configure Filebeat; enable and use the system module in Filebeat to collect, ship, parse, and visualize system log files; deploy and configure Metricbeat; use the system module in Metricbeat to collect, ship, and visualize system telemetry data in Kibana; and explore the Kibana user interface and analyze your system log and telemetry data.

## LEARNING OBJECTIVES

* Install Elasticsearch on Each Node

* Configure Each Node to Form a Three-Node Cluster per the Instructions

* Generate and Deploy the Development Certificate to Each Node

* Encrypt the Elasticsearch Transport Network on Each Node

* Use the `elasticsearch-setup-passwords` Tool to Set the Password for Each Built-In User on the `master-1` Node

* Deploy Kibana on the `master-1` Node

* Configure Kibana to Bind to the Site-Local Address, Listen on Port 8080, and Connect to Elasticsearch

* Deploy Metricbeat on Each Node

* Configure Metricbeat on Each Node to Use the System Module to Ingest System Telemetry to Elasticsearch and Visualize It in Kibana

* Deploy Filebeat on Each Node

* Configure Filebeat on Each Node to Use the System Module to Ingest System Logs to Elasticsearch and Visualize Them in Kibana

* Use Kibana to Explore Your System Logs and Telemetry Data

## Solution
Log in to each node using the credentials provided:

```
ssh cloud_user@<PUBLIC_IP_ADDRESS>

```
Then, become the `root` user in each one:

``` 
sudo su - 

```
IMPORTANT NOTE: Unless otherwise noted, all commands should be run on all nodes.

Hint: When copying and pasting code into Vim from the lab guide, first enter :set paste (and then i to enter insert mode) to avoid adding unnecessary spaces and hashes.

### Install Elasticsearch on Each Node

1. Import the Elastic GPG key:
```
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```
2. Download the Elasticsearch 7.6 RPM:
```
curl -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.6.0-x86_64.rpm

```
3. Install Elasticsearch:
```
rpm --install elasticsearch-7.6.0-x86_64.rpm

```
4. Configure Elasticsearch to start on system boot:
```
systemctl enable elasticsearch

```
### Configure Each Node to Form a Three-Node Cluster per the Instructions

1. Open the `/etc/elasticsearch/elasticsearch.yml` file:

```
vim /etc/elasticsearch/elasticsearch.yml

```
2. On each node, change the following line:

```
#cluster.name: my-application
```
   to

```
cluster.name: development

```
3. On the `master-1` node, change the following line:
```
#node.name: node-1
```
   to

```
node.name: master-1
```
4. On the `data-1` node, change the following line:

#node.name: node-1
   to
```
node.name: data-1
```
5. On the `data-2` node, change the following line:
```
#node.name: node-1
```
   to
```
node.name: data-2
```
6. On each node, change the following line:
```
#network.host: 192.168.0.1
```
   to
```
network.host: [_local_, _site_]
```
7. On each node, change the following line:
```
#discovery.seed_hosts: ["host1", "host2"]
```
   to
```
discovery.seed_hosts: ["10.0.1.101"]
```
8. On each node, change the following line:
```
#cluster.initial_master_nodes: ["node-1", "node-2"]
```
   to
```
cluster.initial_master_nodes: ["master-1"]
```
9. On the `master-1` node, add the following lines:
```
node.master: true
node.data: false
node.ingest: true
node.ml: false
```
10. On both the `data-1` and `data-2` nodes, add the following lines:
```
node.master: false
node.data: true
node.ingest: false
node.ml: false
```
11. On each node, save and quit the file by pressing Escape followed by :wq!.

12. Start Elasticsearch:
```
systemctl start elasticsearch
```
13. On the `master-1` node, check your configuration using the `_cat/nodes` API:
```
curl localhost:9200/_cat/nodes?v
```

### Generate and Deploy the Development Certificate to Each Node

1. Create the `/etc/elasticsearch/certs` directory:
```
mkdir /etc/elasticsearch/certs
```
2. On the `master-1` node, generate the `development` PKCS#12 certificate:
```
/usr/share/elasticsearch/bin/elasticsearch-certutil cert --name development --out /etc/elasticsearch/certs/development
```
3. At the password prompt, press Enter since we don't need to give it a password.

4. On the `master-1` node, allow group read access to the `development` certificate:
```
chmod 640 /etc/elasticsearch/certs/development
```
5. On the `master-1` node, copy the `development` certificate to the `data-1` and `data-2` nodes:
```
scp /etc/elasticsearch/certs/development 10.0.1.102:/etc/elasticsearch/certs/
scp /etc/elasticsearch/certs/development 10.0.1.103:/etc/elasticsearch/certs/
```
6. On the `data-1` node, verify the certificate was copied:
```
ll /etc/elasticsearch/certs/
```
We should see that it's there.

### Encrypt the Elasticsearch Transport Network

1. Open the `/etc/elasticsearch/elasticsearch.yml` file:
```
vim /etc/elasticsearch/elasticsearch.yml
```
2. Add the following lines to the file:
```
#
# ---------------------------------- Security ----------------------------------
#
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: certs/development
xpack.security.transport.ssl.truststore.path: certs/development
```
3. Save and quit the file by pressing Escape followed by `:wq!`.

4. Restart Elasticsearch:
```
systemctl restart elasticsearch
```
### Use the `elasticsearch-setup-passwords` Tool to Set the Password for Each Built-In User on the `master-1` Node

1. On the `master-1` node, set the built-in user passwords using the `elasticsearch-setup-passwords` utility:
```
/usr/share/elasticsearch/bin/elasticsearch-setup-passwords interactive
```
2. Use the following passwords:
```
User: elastic
Password: la_elastic_503

User: apm_system
Password: la_apm_system_503

User: kibana
Password: la_kibana_503

User: logstash_system
Password: la_logstash_system_503

User: beats_system
Password: la_beats_system_503

User: remote_monitoring_user
Password: la_remote_monitoring_user_503
```
### Deploy Kibana on the `master-1` Node

1. On the `master-1` node, download the Kibana 7.6 RPM:
```
curl -O https://artifacts.elastic.co/downloads/kibana/kibana-7.6.0-x86_64.rpm
```
2. On the `master-1` node, install Kibana:
```
rpm --install kibana-7.6.0-x86_64.rpm
```
3. On the `master-1` node, configure Kibana to start on system boot:
```
systemctl enable kibana
```
### Configure Kibana to Bind to the Site-Local Address, Listen on Port 8080, and Connect to Elasticsearch
1. On the `master-1` node, open the `/etc/kibana/kibana` file:
```
vim /etc/kibana/kibana.yml
```
2. Change the following line:
```
#server.port: 5601
```
to
```
server.port: 8080
```
3. Change the following line:
```
#server.host: "localhost"
```
to
```
server.host: "10.0.1.101"
```
4. Change the following lines:
```
#elasticsearch.username: "kibana"
#elasticsearch.password: "pass"
```
to
```
elasticsearch.username: "kibana"
elasticsearch.password: "la_kibana_503"
```
5. Save and quit the file by pressing Escape followed by `:wq!`.

6. Start Kibana:
```
systemctl start kibana
```
7. View its progress:
```
less /var/log/messages
```
8. When we see a message saying `"http server running at http://10.0.1.101:8080"`, it means it's ready.

9. After Kibana has finished starting up, navigate to `http://<PUBLIC_IP_ADDRESS_OF_MASTER-1>:8080` in your web browser and log in as:

* Username: `elastic`
* Password: `la_elastic_503`

### Deploy Metricbeat on Each Node

1. Download the Metricbeat 7.6 RPM:
```
curl -O https://artifacts.elastic.co/downloads/beats/metricbeat/metricbeat-7.6.0-x86_64.rpm
```
2. Install Metricbeat:
```
rpm --install metricbeat-7.6.0-x86_64.rpm
```
3. Configure Metricbeat to start on system boot:
```
systemctl enable metricbeat
```
### Configure Metricbeat on Each Node to Use the System Module to Ingest System Telemetry to Elasticsearch and Visualize It in Kibana

1. Open the `/etc/metricbeat/metricbeat.yml` file:
```
vim /etc/metricbeat/metricbeat.yml
```
2. Change the following lines:
```
setup.kibana:

  # Kibana Host
  # Scheme and port can be left out and will be set to the default (http and 5601)
  # In case you specify and additional path, the scheme is required: http://localhost:5601/path
  # IPv6 addresses should always be defined as: https://[2001:db8::1]:5601
  #host: "localhost:5601"
  ```
to
```
setup.kibana:

  # Kibana Host
  # Scheme and port can be left out and will be set to the default (http and 5601)
  # In case you specify and additional path, the scheme is required: http://localhost:5601/path
  # IPv6 addresses should always be defined as: https://[2001:db8::1]:5601
  host: "10.0.1.101:8080"
  ```
3. Change the following lines:
```
output.elasticsearch:
  # Array of hosts to connect to.
  hosts: ["localhost:9200"]

  # Protocol - either `http` (default) or `https`.
  #protocol: "https"

  # Authentication credentials - either API key or username/password.
  #api_key: "id:api_key"
  #username: "elastic"
  #password: "changeme"
  ```
to
```
output.elasticsearch:
  # Array of hosts to connect to.
  hosts: ["10.0.1.101:9200"]

  # Protocol - either `http` (default) or `https`.
  #protocol: "https"

  # Authentication credentials - either API key or username/password.
  #api_key: "id:api_key"
  username: "elastic"
  password: "la_elastic_503"
  ```
4. Save and quit the file by pressing Escape followed by `wq!`.

5. On the `master-1` node, push the index templates and ingest pipelines to Elasticsearch and the module dashboards to Kibana:
```
metricbeat setup
```
6. On each node, start Metricbeat:
```
systemctl start metricbeat
```
### Deploy Filebeat on Each Node

1. Download the Filebeat 7.6 RPM:
```
curl -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.6.0-x86_64.rpm
```
2. Install Filebeat:
```
rpm --install filebeat-7.6.0-x86_64.rpm
```
3. Configure Filebeat to start on system boot:
```
systemctl enable filebeat
```
### Configure Filebeat on Each Node to Use the System Module to Ingest System Logs to Elasticsearch and Visualize Them in Kibana

1. Open the `/etc/filebeat/filebeat.yml` file:
```
vim /etc/filebeat/filebeat.yml
```
2. On each node, change the following lines:
```
setup.kibana:

  # Kibana Host
  # Scheme and port can be left out and will be set to the default (http and 5601)
  # In case you specify and additional path, the scheme is required: http://localhost:5601/path
  # IPv6 addresses should always be defined as: https://[2001:db8::1]:5601
  #host: "localhost:5601"
  ```
to
```
setup.kibana:

  # Kibana Host
  # Scheme and port can be left out and will be set to the default (http and 5601)
  # In case you specify and additional path, the scheme is required: http://localhost:5601/path
  # IPv6 addresses should always be defined as: https://[2001:db8::1]:5601
  host: "10.0.1.101:8080"
  ```
3. On each node, change the following lines:
```
output.elasticsearch:
  # Array of hosts to connect to.
  hosts: ["localhost:9200"]

  # Protocol - either `http` (default) or `https`.
  #protocol: "https"

  # Authentication credentials - either API key or username/password.
  #api_key: "id:api_key"
  #username: "elastic"
  #password: "changeme"
  ```
to
```
output.elasticsearch:
  # Array of hosts to connect to.
  hosts: ["10.0.1.101:9200"]

  # Protocol - either `http` (default) or `https`.
  #protocol: "https"

  # Authentication credentials - either API key or username/password.
  #api_key: "id:api_key"
  username: "elastic"
  password: "la_elastic_503"
  ```
4. Save and quit the file by pressing Escape followed by `wq!`.

5. Enable the `system` module:
```
filebeat modules enable system
```
6. On the `master-1` node, push the index templates and ingest pipelines to Elasticsearch and the module dashboards to Kibana:
```
filebeat setup
```
7. On each node, start Filebeat:
```
systemctl start filebeat
```
### Use Kibana to Explore Your System Logs and Telemetry Data

1. Navigate to `http://<PUBLIC_IP_ADDRESS_OF_MASTER-1>:8080` in your web browser and log in as:

* Username: `elastic`
* Password: `la_elastic_503`
2. On the side navigation bar, click the dashboards icon.

3. In the search bar, enter "Filebeat System" to find your sample dashboards.

4. Click the **Syslog dashboard ECS** result. Give it a few minutes for more data to populate.

5. Switch to the **Last 1 hour** view.

6. In the pie graph, select the **Kibana** portion.

7. Click **Apply** in the dialog.

8. Scroll through the logs along the bottom, and click to expand one of them.

9. Remove the filters.

10. Click to view the **Sudo commands** dashboard to see who's accessing the hosts with `sudo`. We should just see `cloud_user`.

11. On the side navigation bar, click the dashboards icon.

12. In the search bar, enter "Metricbeat System" to find your sample dashboards.

13. Click the **Overview ECS** result, which provides a bird's-eye view of all the hosts we're collecting telemetry from.

14. Under Top Hosts By Memory (Realtime)[Metricbeat System]ECS, click our `master-1` node (the `ip-10-0-1-101.ec2.internal` bar), which will take us to the host overview dashboard.

15. Change the timeframe to **Last 15 minutes** to get a closer look at the data we've collected so far.

### Conclusion

Congratulations on successfully completing this hands-on lab!
