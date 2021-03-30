# HDI Kafka Setup	

In this tutorial, we will be exploring the use case for running Kafka on Azure HDI and the various steps involved. This was done for a customer where an on-premise Kafka was currently deployed. The customer wanted to replicate the Kafka stream from on-premises to another Kafka running on cloud (with Mirrormaker). Subsequently, this stream would be read by a spark process running in Databricks.   

In this tutorial, we would be going through the following steps:

* Deploy Azure HDI (Kafka)
* Access and configure Kafka on HDI post deployment
* Setup On-prem Kafka Instance (this will be simulated with installing Kafka in a VM)
* Deploy Databricks Cluster
* Setup a sample spark code to read from the HDI Kafka instance



### Deploy Azure HDI (Kafka)

For the purpose of this tutorial, we assume that you have access to an Azure subscription and are able to create resources. 

Search for HDI in Azure Marketplace. Click 'Create'.

![](./images/deploy_hdi_1.jpg)

In this example, we will create an HDI Cluster just for Kafka. Choose 'Cluster Type' as 'Kafka' and provide a userid/password for cluster login.

We'll keep all other options as default, for now. Click 'Review + Create'. 

![](./images/deploy_hdi_2.jpg)

Once all validations have succeeded, the cluster will get created. This deployment could take anywhere between 10-30 mins. 

![](./images/deploy_hdi_3.jpg)



### Access and configure Kafka on HDI post deployment

Once our cluster is successfully deployed, we can connect to the cluster using the ssh command.  Edit the command below by replacing the name of the cluster and then enter the command. 

`ssh sshuser@CLUSTERNAME-ssh.azurehdinsight.net`

Once connected, the screen output would be as follows:

```
Authorized uses only. All activity may be monitored and reported.
sshuser@hXXXXXXXX-ssh.azurehdinsight.net's password:
Welcome to Ubuntu 16.04.7 LTS (GNU/Linux 4.15.0-1109-azure x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
   Welcome to Ubuntu 16.04.7 LTS (GNU/Linux 4.15.0-1109-azure x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

12 packages can be updated.
5 of these updates are security updates.
To see these additional updates run: apt list --upgradable
*** /dev/sda1 will be checked for errors at next reboot ***
*** System restart required ***

Welcome to Kafka on HDInsight.
The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

sshuser@hn0-hdiscb:~$
```

After connecting to the cluster, we need to identify the *Apache Zookeeper* and *Broker* information. These are referenced by Kafka and other utilities as will be seen later in the tutorial. 

We will use Ambari REST API on the cluster to get host information. 

Tip: Install [jq](https://stedolan.github.io/jq/), a command line processor to help with parsing JSON documents.

To install jq, execute the following:

```
sudo apt -y install jq
```

Set up password variable. Replace `PASSWORD` with cluster login password and enter:

```
export password='PASSWORD'
```

We need to now extract the cluster name (in correct case). The following command extracts the cluster name and stored it in a variable. 

```
export clusterName=$(curl -u admin:$password -sS -G "http://headnodehost:8080/api/v1/clusters" | jq -r '.items[].Clusters.cluster_name')
```

Let's also extract Zookeeper hosts information:

```
export KAFKAZKHOSTS=$(curl -sS -u admin:$password -G https://$clusterName.azurehdinsight.net/api/v1/clusters/$clusterName/services/ZOOKEEPER/components/ZOOKEEPER_SERVER | jq -r '["\(.host_components[].HostRoles.host_name):2181"] | join(",")' | cut -d',' -f1,2);
```

To confirm the Kafka Hosts information, type the following and note the hosts:

```
echo $KAFKAZKHOSTS

zk0-hdixxx.fuxxxxxxdlujcxxxxxaid.bx.internal.cloudapp.net:2181,zk1-hdiscb.fuxxxxxxdlujcxxxxxaid.bx.internal.cloudapp.net:2181
```

Let's also setup a variable to store the Apache Kafka Broker host information:

```
export KAFKABROKERS=$(curl -sS -u admin:$password -G https://$clusterName.azurehdinsight.net/api/v1/clusters/$clusterName/services/KAFKA/components/KAFKA_BROKER | jq -r '["\(.host_components[].HostRoles.host_name):9092"] | join(",")' | cut -d',' -f1,2);
```

Let's validate the Kafka Brokers:

```
echo $KAFKABROKERS

wn0-hdiscb.fuxxxxxxdlujcxxxxxaid.bx.internal.cloudapp.net:9092,wn1-hdiscb.fuxxxxxxdlujcxxxxxaid.bx.internal.cloudapp.net:9092
```



### Setup On-Prem Kafka

In this scenario, our customer already has a Kafka deployment on-premises that they want to replicate messages from. In this tutorial, we will simulate this by deploying Kafka on an Azure VM and setup replication from this Kafka instance to the HDI Kafka that we have deployed above. 

An important consideration here is also for Eventhub, as an option for Azure HDI Kafka. Eventhub is also able to get data from a Kafka instance, with the additional benefits of then being able to store this data in ADLS and/or leveraging Stream Analytics to then integrate with the stream.

Let's start with deploying a VM on Azure. For this, an ideal option is the OpenLogic Centos VM (7.5) V1. Since we wouldn't be doing much heavy lifting here, we can use the Standard B2ms shape for it, without any replication. 

Deploy the VM using marketplace.

![](./images/config_onprem_kafka_1.jpg)

Keep all settings as default and click 'Review + Create'.

![](./images/config_onprem_kafka_2.jpg)

Once the VM is deployed, we'll connect to using ssh and install Kafka from command line. 

Download the latest Kafka and extract it:

```
wget https://apachemirror.sg.wuchna.com/kafka/2.7.0/kafka_2.13-2.7.0.tgz

tar -xzf kafka_2.13-2.7.0.tgz
```

Install Java 8+

```
sudo yum install java-1.8.0-openjdk
```

Validate Java Install:

```
java -version
openjdk version "1.8.0_282"
OpenJDK Runtime Environment (build 1.8.0_282-b08)
OpenJDK 64-Bit Server VM (build 25.282-b08, mixed mode)
```

Configure JAVA_HOME

```
#Find the Java Path
update-alternatives --config java


There is 1 program that provides 'java'.

  Selection    Command
-----------------------------------------------
*+ 1           java-1.8.0-openjdk.x86_64 (/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.282.b08-1.el7_9.x86_64/jre/bin/java)

Enter to keep the current selection[+], or type selection number:

#Update .bash_profile and add JAVA_HOME
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.282.b08-1.el7_9.x86_64/jre

#Save the file and source it
source .bash_profile

echo $JAVA_HOME
```

Start the Kafka environment:

```
cd kafka_2.13-2.7.0

# Start the ZooKeeper service
$ bin/zookeeper-server-start.sh config/zookeeper.properties
```

Then open another terminal session and run:

```
cd kafka_2.13-2.7.0

# Start the Kafka broker service
$ bin/kafka-server-start.sh config/server.properties
```

All services started !

![](./images/config_onprem_kafka_3.jpg)

Let's create a topic to store events (open another terminal):

```
cd kafka_2.13-2.7.0

bin/kafka-topics.sh --create --topic quickstart-events --bootstrap-server localhost:9092

Created topic quickstart-events.
```

Let's validate the partition count of the new topic:

```
bin/kafka-topics.sh --describe --topic quickstart-events --bootstrap-server localhost:9092

Topic: quickstart-events        PartitionCount: 1       ReplicationFactor: 1    Configs: segment.bytes=1073741824
        Topic: quickstart-events        Partition: 0    Leader: 0       Replicas: 0     Isr: 0
```

Let's send some data to this topic with a console producer. (Ctrl+C to exit)

```
bin/kafka-console-producer.sh --topic quickstart-events --bootstrap-server localhost:9092
This is Event 1
This is Event 2

```

Let's read this back from the console consumer:

```
bin/kafka-console-consumer.sh --topic quickstart-events --from-beginning --bootstrap-server localhost:9092
This is Event 1
This is Event 2
^CProcessed a total of 2 messages
```

