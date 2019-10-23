# Threat Feed Dashboard

![kibana dashboard](/img/main_dashboard.png)
## Integration of IntelMQ, ElasticSearch and Kibana

At first we should know about them and their usages.

* **IntelMQ** is a solution for IT security teams (CERTs & CSIRTs, SOCs, abuse departments, etc.) for collecting and processing security feeds (such as log files) using a message queuing protocol.

* **ElasticSearch** lets you perform and combine many types of searches — structured, unstructured, geo, metric — any way you want.

* **Kibana** is an open source data visualization plugin for Elasticsearch.

When I tried to integrate them, there was very little guide. Though their personal documentations are really great. So, I start preparing a guideline based on my experience. 


>**Software Requirements:** 
OS : Ubuntu 18.04 LTS. 
IntelMQ : 2.1.0
IntelMQ-Manager : 2.1.0
ElasticSearch : 6.8.3
Kibana : 6.7.1
Java : Open JDK 8
Python : 3.6.8

>**Hardware Requirements:** 
Minimum Core i5
Ram :  8 GB
Core : 2
Process : 2

Based on OS and Software version, your installation guide could be changed, but integration procedure will be same.

## Step 1 : Installation

### How to install IntelMQ?

* Firstly, update your ubuntu source list.

 In Ubuntu 18.04 (enable the universe repositories by appending universe in /etc/apt/sources.list to deb http://[...].archive.ubuntu.com/ubuntu/ bionic main)

* Before installation, you have to resolve some dependencies. Just copy and paste these below command.

```bash
$sudo apt-get install python3-pip python3-dnspython python3-psutil python3-redis python3-requests python3-termstyle python3-tz python3-dateutil
$sudo apt-get install redis-server
```
* Optional dependencies:

```bash
$sudo apt-get install bash-completion jq
$sudo apt-get install python3-sleekxmpp python3-pymongo python3-psycopg2
```

* Now we are ready for installing intelmq. We need root privilege. So,

```bash
$sudo -i
$pip3 install intelmq
$useradd -d /opt/intelmq -U -s /bin/bash intelmq
$sudo intelmqsetup
```
* After installation, prior to the first run, copy all .conf files from /opt/intelmq/ect/examples files to /opt/intelmq/etc directory

```bash
$cd /opt/intelmq/etc
$cp -a examples/* .
```
* Configure Services. Enable and Start Redise Service

```bash
$systemctl enable redis-server.service
$systemctl start redis-server.service
```

>Intelmq basic dataflow diagram

![dataflow](/img/inelmq-dataflow.png)

### How to install IntelMQ-Manager?

IntelMQ-Manager is very useful browser base gui tool. If you prefer GUI over console, then intelmq-manager will help you lots.
You can install it by downloading .deb package but unfortunately it does not work for me. So, I have installed it using following commands.

* Before installation, there are some dependencis. To resolve this, run this following command. 

```bash
$sudo apt-get install git apache2 php libapache2-mod-php7.2
```

* Add intelmq-manager in the source list and install intelmq-manager using this command. During installation,you will be asked to set username and password. You will need this for very time when using intelmq-manager.

```bash
$sudo sh -c "echo 'deb http://download.opensuse.org/repositories/home:/sebix:/intelmq/xUbuntu_18.04/ /' > /etc/apt/sources.list.d/home:sebix:intelmq.list"
$wget -nv https://download.opensuse.org/repositories/home:sebix:intelmq/xUbuntu_18.04/Release.key -O Release.key
$sudo apt-key add - < Release.key
$sudo apt-get update
$sudo apt-get install intelmq-manager
```

**NOTE:** After installation you may face some issues in intelmq-manager for python libraryYou can ignore them. In that case, intelmq-manager gui will not behave properly. So, it is better to solve this.

The warning or error message looks like :

```
 RequestsDependencyWarning: urllib3 (1.24.3) or chardet (3.0.4) doesn't match a supported version!RequestsDependencyWarning)
```
To solve this issue you have to fix correct version of urllib3 library.

After some r&d, finally I have found correct version would be 1.21.1 for urllib3

```bash
$pip3 install urllib3==1.21.1
```

* For Debian and Ubuntu you need to make the configuration files writable by the group:

```bash
$chmod 664 /etc/intelmq/*.conf /etc/intelmq/manager/positions.conf
```
* Now visit http://localhost:80 to browse intelmq-manager( you need username and password).

### How to install ElasticSearch?

* Choose your preferred elasticsearch .deb package from elasticsearch website.

* Run the following command to start and stop elasticsearch

```bash
$sudo -i service elasticsearch start
$sudo -i service elasticsearch stop
```

or using these following command to start automatically during boot 

```bash
$sudo systemctl enable elasticsearch.service
$sudo systemctl start elasticsearch.service
$sudo systemctl stop elasticsearch.service
```
* Now hit http://localhost:9200 and check

### How to install Kibana?

* After installing kibana from .deb file, run the following commands

```bash
sudo -i service kibana start
sudo -i service kibana stop
```
* Hit http://localhost:5601 to browser kibana

## Step 2 : Integration

### ElasticMapper

* **elasticmapper** is a tool(python script) to generate the Elasticsearch mapping required to specify the proper fields and field types which will be inserted into the Elasticsearch database. This tool uses the IntelMQ harmonization file to automatically generate the mapping and provides a quick way to send the mapping directly to Elasticsearch or write the generated mapping to a local file. 

* Before run elasticmapper script install the following dependency:

```bash
$pip3 install elasticsearch
```
* Now download elasticmapper script from git intemq repo(intemq/contrib/elasticsearch directory).

* Go to the download folder and give execut permission to **elasticmapper** file.

```bash
$chmod +x elasticmapper
```

**Note:** After ElasticSearch version 7, index-type is deprecated. So, you have to update elasticmapper python code. Update data json generated code, remove index-type. To avoid this error, I have installed elastisearch 6.8.3 . After updating data mapping, it looks like :  

```
data = {
    "mappings": {
           "properties": properties
     }
 }
 ```
* Finally you are ready to run elasticmapper 
 
 ```bash
 $ ./elasticmapper --harmonization-file=intelmq/intelmq/etc/harmonization.conf --index=intelmq --index-type=events --host=127.0.0.1 --harmonization-fallback
 ```


### Intelmq-manager

* Using intelmq-manager you can draw your feed architecture or topology. It is one kind of darg and drop model. Your topology will be :

```
Feed BOT > Parser BOT > Expert BOT > Output
```

* Can start and stop any bot.

* Monitor bots status.

* Check topoloy's status or health( notify misconfiguration, error etc)

>After drawing a new topolgy, you must update pipeline.conf and runtime.conf file (location : /opt/intelmq/etc/)

>For any bot please check /opt/intelmq/ect/BOT file. You will get maximum clue from here. When you want to add a new bot, you can check BOT file, copy your required bot's json portion and add it in runtime.conf file. 
In pipeline, you should define bots input and output data queue. To write a proper and error free pipeline.conf file, you should carefully read /opt/intelmq/ect/examples/pipeline.conf file first. Then you can understand how to write a pipeline.conf file.

### Create elasticsearh-output

* Add a elasticsearch-output node from intelmq-manager>configuration (gui). Go left side menubar Output > Elasticsearch, drag and drop new node. 
Then update proper configuration for runtime.conf and pipeline.conf. Add destination-queue and source-queue.

![topology](/img/intelmq-conf.png)

---

![pipeline.cof](/img/pipeline.conf.png)

---

![runtime.cof](/img/runtime.conf.png)

### Run your BOTS

* Now you can run your BOTS from intelmq-manager gui or you can using console 

```bash
$intelmqctl start
```

* Check bots status
```bash
$intelmqctl status
```

* Stop bots
```bash
$intelmqctl stop
```

* Check current configure
```bash
$intelmqctl check
```

* After starting bots, if everything ok, you will get **intelmq** data in **kibana** for creating *index-pattern*. 

### Visualize on dashboard

![index-pattern](/img/kibana_intelmq_index.png)

* Create index-pattern.

* Using index-pattern draw your required graphs. 

* Create you dashboard using this graphs.

**For more details please read user guide provided by intelmq, intelmq-manager, elasticsearch and kibana**
