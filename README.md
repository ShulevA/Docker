# ELK Stack

### ELK - elasticsearch, logstash, kibana

**Elasticsearch** - mechanism of indexing and storing the received information, as well as full-text search on it.

**Logstash** - tool for obtaining, converting and storing data in a common storage.

**Kibana** - interface for Elasticsearch that can search and display data in the form of tables, graphs, charts, etc.

### ELK Stack Architecture

Simple architecture of ELK stack:

![](https://www.guru99.com/images/tensorflow/082918_1504_ELKStackTut2.png "Simple architecture of ELK stack")

Production ELK stack _(our case)_ may look like this:

![](/home/artem/Desktop/ELK stack production_3.jpg "Production architecture of ELK stack")

#### 1. Log delivery to logstash-shipper

Docker supports various logging mechanisms that allow you to receive information from running containers and services. 
These mechanisms are called logging drivers. We are interested in a driver called gelf.  
In order for the docker to use the gelf log driver, you must specify the logging parameter:
```  
  service:
    image: service image
    logging:
      driver: gelf
      options:
        gelf-address: "udp://1.2.3.4:12201"
        tag: "my service"
```
You must also specify the IP address to which the logs will be sent via UDP. 
Default udp port for gelf logs in logstash 12201. It can be changed in logstash-shipper.conf _(deploy_repo/Logstash/)_.

#### 2. Log delivery from logstash-shipper to redis-broker

Logs getting into the logstash-shipper pass through 
[filters](https://www.elastic.co/guide/en/logstash/current/filter-plugins.html) and are sent to the redis-broker. 
These parameters are defined in logstash-shipper.conf.  
Output section looks like this:
```
output {
  redis {
    host => "1.2.3.4"
    port => 6379
    data_type => "list"
    key => "logstash"
    codec => json
    password => "password"
  }
}
```
Password is indicated in redis.conf _(ELK/)_.

#### 3. Log delivery from redis-broker to elasticsearch

Logstash-indexer is responsible for the delivery of logs from the redis-broker to elasticsearch. 
It picks them up from redis-broker and sends them to the elasticsearch with index and date stamp:
```
input {
    redis {
        host => "1.2.3.4"
        port => 6379
        type => "redis-input"
        data_type => "list"
        key => "logstash"
        codec => "json"
        password => "password"
    }
}

output {
    elasticsearch {
        hosts => "elasticsearch:9200"
        index => "logs-%{+YYYY.MM.dd}"
    }
}
```

#### 4. View logs in kibana

Kibana is protected by nginx and is available at the nginx-auth service port 8080. 
Username and password are specified in .env file. If index pattern is created, then logs are available on the discover tab. 
Else, you need to create it in tab Management > Index Patterns > Create index pattern. In our case, index-name is "logs-". 
Time filter field name is @timestamp. 

![](/home/artem/Desktop/kibana.png)

#### Delete old logs in ELK

##### Manual methods

`1)` If you are using time series index names you can do something like:
```
curl -XDELETE http://1.2.3.4:9200/index-yyyy.mm*
```
`2)` If you're not using date stamp in index - use Elasticsearch Curator. In a running curator container 
_(use: docker exec -it \<container id\> bash)_ on ELK server:

In the folder /etc/curator
```
curator --config config.yml Actions/delete_indices.yml
```
This will delete all logs older than 15 days. 

In general, the syntax is as follows:
```
curator [--config CONFIG.YML] ACTION_FILE.YML
```

##### Automatic method

Using cron, service elasticsearch-curator _(container name curator)_ automatically launches a command
every 1, 16 day of the month:
```
curator --config config.yml Actions/delete_indices.yml
```
Cron's task looks like this:
```
* * 1,16 * * /etc/curator/delte_old_logs.sh
```