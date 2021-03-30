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