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

An important consideration here is also for Eventhub, as an option for Azure HDI Kafka. Eventhub is also able to get data from a Kafka instance, with the additional benefits of then being able to store this data in ADLS and/or leveraging Stream Analytics to then integrate with the stream