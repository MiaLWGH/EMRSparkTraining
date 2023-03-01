# LAB - Getting Started with Spark on EMR
This lab aims to help engineers who are not familiar with EMR to get some fundamental understanding of how to use EMR to run Spark data processing jobs. In this lab, we are going to launch a sample cluster, submit a few Spark jobs, observe and troubleshoot, as well as tune Spark parameters to ensure the job can complete successfully. After completing this lab, engineers should be able to submit simple Spark jobs to EMR and tell the difference between the client deploy mode and cluster deploy mode in Spark. The estimated completion time of this lab is about 15 minutes. 

## Step 0 - Launch an EMR cluster
In this step, we are going to create required IAM roles and launch an EMR sample cluster that only contains one master node and one core node. 

1. First of all, let us first create the IAM default roles needed for an EMR cluster. You can skip to 2 if you already have the default roles. Please run the following AWS CLI command:
```
aws emr create-default-roles
```
This command will create two IAM roles, `EMR_DefaultRole` and `EMR_EC2_DefaultRole`. 

Note: The first role defines the allowable actions for Amazon EMR when provisioning resources and performing service-level tasks that are not       performed in the context of an EC2 instance running within a cluster. For example, the service role is used to provision EC2 instances when a cluster launches. The second one is assigned to every EC2 instance in an Amazon EMR cluster when the instance launches. Application processes that run on top of the Hadoop ecosystem assume this role for permissions to interact with other AWS services.

2. Login to your AWS Console and navigate to EMR console. In this lab, we assume engineers are using the new console. Click Create cluster and make sure the following information in the prompt page:
  + Under Name and applications, select EMR release emr-6.9.0 and Spark for Application bundle.
  + Under Cluster configuration, choose instance type m4.large for both Primary and Core instance groups. Remove Task instance group and ensure the size of Core instance group is 1 instance.
  + Under Networking, select a public subnet for the cluster.
  + Under Security configuration and permissions, select a feasible key pair that will be used to SSH to the master node. Select the `EMR_DefaultRole` for the Service role for Amazon EMR and `EMR_EC2_DefaultRole` for the IAM role for instance profile.
  
Keep all other default settings and then click create cluster. The launching process may take about 10 minutes. 
  
After the cluster being launched, what is the cluster status now?
  
## Step 1 - Prepare and submit a Spark job
  
