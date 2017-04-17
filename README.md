# Docker-ELK

[Docker](http://www.docker.com) containers are great to implement your DevOps infrastructure but there is also a need to monitor your infra and the logs which are generated from your Docker containers. Given that you would probably be tearing down your containers quite regularly you need to be able to persist the logs somewhere . One solution is using ELK which stands for {ElasticSearch](https://www.elastic.co/products/elasticsearch) , [Logstash](https://www.elastic.co/products/logstash) and [Kibana](https://www.elastic.co/products/kibana)  . However the solution can be used in an scenario where you want to have an ELK server up an running quickly to monitor any log files.

Basically you would be sending log information to Logstash which then sends it to ElasticSearch for storage and the captured logs are then viewabled through Kibana , which is pretty cool :) !

In a typical scenario your docker containers might not be on the same docker host as the one hosting the ELK servers , so you will see that there are 2 docker-compose files :
* docker-compose-elk.yml - this is to setup the ELK server
* docker-compose-filebeat.yml - this is to setup [Filbeat](https://www.elastic.co/products/beats/filebeat) , which will be monitoring your remote log files and pushing the changes to Logstash .



## Prerequisites

* Docker is installed [click here](http://javedmandary.blogspot.com/2017/01/install-docker-in-2-commands-on-ubuntu.html)
* Docker-compose is [installed](https://docs.docker.com/compose/install/) 
* Run the following command on your host "sudo sysctl -w vm.max_map_count=262144" else ElasticSearch might fail with an vm.max_map_count error check the following for [more details](https://github.com/docker-library/elasticsearch/issues/111)


## Setting up the ELK server

* Start by downloading the codes in this repository to you docker host environment where you want to setup your ELK infrastructure 
* Now open up the docker-compose-elk.yml file , things to note :
** elasticsearch is exposed on port 9200 , kibana on port 5601 and logstash on port 5044 , you want to make sure that no firewall are blocking these ports
** files within the logstash sub directory on your host are being mapped with a volume on the logstash container
*** the logstash directory contains information around how your logs should look like , feel free to play around and modify

```
version: "2"
services:
  elasticsearch:
    image: elasticsearch:5.1.1
    ports: 
      - 9200:9200
    container_name: elasticsearch
  kibana:
    image: kibana:5.1.1
    ports:
      - 5601:5601
    depends_on:
      - elasticsearch
    container_name: kibana
  logstash:
    image: logstash:5.1.1
    command: logstash -f /etc/logstash/conf.d/logstash.conf
    ports:
      - 5044:5044
    volumes:
      - ./logstash:/etc/logstash/conf.d
      - ./logstash/patterns:/opt/logstash/patterns
    depends_on:
      - elasticsearch
    container_name: logstash
```

* Fire up the docker-compose-elk.yml file to start the ELK containers by executing the following :

```
docker-compose -f ./docker-compose-elk.yml up
```

Wait for some time and then check whether kibana is up by opening the following url ( change hostname to your ELK server ip address ):
```
http://HOST_NAME:5601
```


## Setting up Filebeat 

Filebeat will be monitoring log files on your remote docker host ( or could be local as well) and then pushing the log data to the ELK server through logstash on port 5044.

* Start by downloading the codes in this repository to you docker host environment where you want to setup Filebeat to monitor your logs ( docker container logs or not)
* Open up the docker-compose-filebeat.yml file in your favorite editor , you might want to change /home/ubuntu/logs/tomcat6-myapp-server to the location of your logs 

```
version: "2"
services:

  filebeat:
    image: prima/filebeat
    container_name: filebeat
    volumes:
      - ./filebeat/filebeat.yml:/filebeat.yml
      - /home/ubuntu/logs/tomcat6-myapp-server:/var/log
```

* Open up ./filebeat/filebeat.yml file:
** change the /var/log/app.log line to /var/log/NAME_OF_YOUR_LOGFILE.log , note that you can add a number of logs to be monitored , chedk the filebeat prospector configurations on the official site
** Change hosts value of localhost to the ip address of your ELK server

```
filebeat:
  prospectors:
    -
      paths:
        - /var/log/app.log
      input_type: log
      document_type: myapp_log
      scan_frequency: 10s
   
output:
  logstash:
    hosts: ["localhost:5044"]
logging:
  files:
    rotateeverybytes: 10485760 # = 10MB
  selectors: ["*"]
  level: warning
```

* Start Filebeat log monitoring  by starting docker-compose-file.yml file by executing the following :

```
docker-compose -f ./docker-compose-elk.yml up
```
