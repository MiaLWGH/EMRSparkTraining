# LAB - Getting Started with Spark on EMR
This lab aims to help engineers who are not familiar with EMR to get some fundamental understanding of how to use EMR to run Spark data processing jobs. In this lab, we are going to launch a sample cluster, submit a few Spark jobs, observe and troubleshoot, as well as tune Spark parameters to ensure the job can complete successfully. After completing this lab, engineers should be able to submit simple Spark jobs to EMR and tell the difference between the client deploy mode and cluster deploy mode in Spark. The estimated completion time of this lab is about 15 minutes. 

## Step 0 - Launch an EMR cluster
In this step, we are going to create required IAM roles and launch an EMR sample cluster that only contains one master node and one core node. 

1. First of all, let us first create the IAM default roles needed for an EMR cluster. You can skip to 2 if you already have the default roles. Please run the following AWS CLI command:
```
aws emr create-default-roles
```
This command will create two IAM roles, `EMR_DefaultRole` and `EMR_EC2_DefaultRole`. 

> Note: The first role defines the allowable actions for Amazon EMR when provisioning resources and performing service-level tasks that are not       performed in the context of an EC2 instance running within a cluster. For example, the service role is used to provision EC2 instances when a cluster launches. The second one is assigned to every EC2 instance in an Amazon EMR cluster when the instance launches. Application processes that run on top of the Hadoop ecosystem assume this role for permissions to interact with other AWS services.

2. Login to your AWS Console and navigate to EMR console. In this lab, we assume engineers are using the new console. Click Create cluster and make sure you follow below rules in the configuration page:
  + Under Name and applications, select EMR release emr-6.9.0 and Spark for Application bundle.
  + Under Cluster configuration, choose instance type m4.large for both Primary and Core instance groups. Remove Task instance group and ensure the size of Core instance group is 1 instance.
  + Under Networking, select a public subnet for the cluster.
  + Under Cluster logs, select an S3 bucket for logging. 
  + Under Security configuration and permissions, select a key pair that will be used to SSH to the master node. Select the `EMR_DefaultRole` for the Service role for Amazon EMR and `EMR_EC2_DefaultRole` for the IAM role for instance profile.
  
Keep all other default settings and create cluster. The cluster will be up within 10 minutes. 
  
## Step 1 - Prepare and submit a Spark job
Next, we are going to submit a sample Spark job to the cluster, which will retrieve source data from S3 bucket, process and return results file to S3 bucket. Please run the following command in your local machine to download the sample input data and a PySpark script from a GitHub repository:
```
git clone https://github.com/MiaLWGH/EMRSparkTraining
```
> Note: File `food_establishment_data.csv` is the sample input data which is a modified version of Health Department inspection results in King County, Washington, from 2006 to 2020. File `health_violations.py` is a sample PySpark script that processes food establishment inspection data and returns a results file in your S3 bucket. The results file lists the top ten establishments with the most "Red" type violations.

Please upload the input data file `food_establishment_data.csv` and PySpark script `health_violations.py` to your S3 bucket. 

Let us go back to EMR console to submit the Spark job. Find tab Steps and Click Add step. Fill in the following information. Please modify the S3 bucket URIs to your own ones:
```
Type: Custom JAR
Name: My Spark App
JAR location: command-runner.jar
Arguments: spark-submit s3://<bucketname>/health_violations.py --data_source s3://<bucketname>/food_establishment_data.csv --output_uri s3://<bucketname>/Output/
```
After submitting, each step will have a step ID and its running status is showing in the console. This application should complete within 2 minutes. 

If your job failed, click on the status `Failed`, can you find any clue?

If your step completed successfully, find tab Applications and open `Spark UI`, you shall be able to see one application in the list. You will be able to check more information about this application if you click the App ID. Explore the Spark UI, and on page of Executors, can you find the answer for the following questions about executors?
1. Besides driver, how many executors did you use?
2. Which node was the driver running on? How about the executor(s)?
3. Where was the YARN Application Master?

## Step 2 - Resubmit the Spark job in cluster mode
Let's then try to submit the same job, but in cluster mode and see what will happen. We add `--deploy-mode cluster` in Arguments when submitting the job:
```
Type: Custom JAR
Name: My Spark App
JAR location: command-runner.jar
Arguments: spark-submit --deploy-mode cluster s3://<bucketname>/health_violations.py --data_source s3://<bucketname>/food_establishment_data.csv --output_uri s3://<bucketname>/Output/
```
Check the step status. Can it finish within 2 minutes? If you want to find the application that is still running, you can click 'Show incomplete applications' in Spark UI. Can you see any executor launched for this application?

Let's do a simple calculation to find out why no executor started and application is stuck. Go to `Environment` tab in Spark UI. In the long list of parameters, can you find the default value of the following two parameters?
```
spark.driver.memory
spark.executor.memory
```
Remember, we are using instance type [m4.large](https://aws.amazon.com/ec2/instance-types/) for the nodes in the cluster, where each node has 2 vCPUs and 8 GB memory. In an EMR cluster, each node needs to reserve part of the memory to run system daemons. Thus, the memory that can be used for running application is actually less than 8 GB, which is controlled by a parameter named `yarn.nodemanager.resource.memory-mb`. Given we are running the Spark application in cluster mode now, both driver and executor are running at the core node. We then have the following constraint:
```
spark.driver.memory + spark.executor.memory < yarn.nodemanager.resource.memory-mb
```
Can you find the value of `yarn.nodemanager.resource.memory-mb` for m4.large in [this documentation](https://docs.aws.amazon.com/emr/latest/ReleaseGuide/emr-hadoop-task-config.html#emr-hadoop-task-config-m4)? Is the current setting breaking the rule?

Since we now understand why the application stuck, let us stop it. Select the running step and click Action -> Cancel steps. In the prompt window, select `Cancel the step and force it to exit`. In EMR, cancelling step cannot directly kill the YARN application. Thus we need to SSH to the master node to kill it:
```
ssh -i ~/my_EMR_key_pair.pem hadoop@ec2-XXX-XXX-XXX-XXX.ap-southeast-2.compute.amazonaws.com
```
Next, run the following command to list the currently running applications:
```
yarn application -list
```
In the returned result, you shall be able to find an application ID in the format `application_XXXXXXXXXXXXXX_XXXX`. Copy and paste it into the following command:
```
yarn application -kill application_XXXXXXXXXXXXXX_XXXX
```
To double check if the application is successfully killed, you can rerun the list command. 

Next, to make the job successfully run in cluster mode, we can try to tune the parameters. One option is to override the executor memory size to 2 GB by adding `--executor-memory 2g` in Arguments:
```
Type: Custom JAR
Name: My Spark App
JAR location: command-runner.jar
Arguments: spark-submit --deploy-mode cluster --executor-memory 2g s3://<bucketname>/health_violations.py --data_source s3://<bucketname>/food_establishment_data.csv --output_uri s3://<bucketname>/Output/
```
Does it work this time? What's the value of spark.executor.memory in the Environment page in Spark UI? Where are the driver and executor running in this application?

## Step 3 - Further tune Spark parameters
Now let us assume you need 5 GB memory for driver and executor respectively. Can you make the job run successfully at this cluster?
Tips:
+ Which deploy mode do you want to use?
+ The full list of Spark properties for Spark 3.3.0 can be found in [Spark documentation](https://spark.apache.org/docs/3.3.0/configuration.html).
+ Can you have more nodes in the cluster?

Extra Questions:
1. When stop an application, what will you see if you skip cancelling the step and kill the application directly via SSH to master node?
2. While the stuck applicatino is still running, if you add a new node in the cluster, do you think the application can complete successfully?
