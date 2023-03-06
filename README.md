# Microservices User and Shift Management Applications
These kubernetes resources (yaml) deploy two containers that scale independently. Two RDS databases hosted on AWS are attached to the containers.
+ The first container runs API that returs users from the external (outside the cluster) PostgreSQL
+ The second container returns shifts from the RDS mySQL database outside the cluster.

## Prerequisite
+ For this project, it is assumed that you have a running EKS cluster running on AWS.
+ Create security groups with the required ports open.
+ Install AWS CLI following the official documentation https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
+ Create a user with programatic access on IAM service.
+ Setup local AWS authentication and Authorisation using: aws configure.

## These kubernetes resources addresses the following requirements:
+ Scales to accommmodate high traffic during the day and low traffic during the night.
+ Auto scales at an avearge CPU of 70%.
+ Uses a rollingupdate Deployment strategy.
+ Limits the access of the development team in the cluster to only deploy and rollback.
+ Creates different environments (staging and production).
+ An alternative auto-scalling of the deployment based on network latency instead of CPU.

## Step 1:
### Clone the repository to your local machine:
```
git clone https://github.com/ThinkOn-Chibuike/DevOps-take-home-test.git
```
### Create the Staging and Production environmnets using Namespace Objects
```
kubectl apply -f nameSpaces.yml
```
### Create Secret object for the databases credentials
```
kubectl apply -f dbSecret.yml
```
NB. base64 is used to encode the databases root user password

## Step 2:
### Create a Deployment object that defines the two containers; one for the user API and the other for the shift
```
kubectl apply -f deploy-prod.yml
kubectl apply -f deploy-stage.yml
```
This will deploy the containers in both production and staging environmnet.
Alternatively, you could have a single Deployment object without spacing a namespace in the configuration, an then use the command below
```
kubectl apply -f deploy.yml -n <namespace> # simply specify the namespace
```
The deployment uses a RollingUpdate strategy, which is the default deployment stratedy.
To rollback a Deployment, run the command below
```
kubectl rollout undo userMgt-restapi
```

## Step 3:
Create an ExternalName and loabBalancer services for the containers to access the RDS databases hosted on AWS that are outide the cluster and for the end users to access the application respectively. ClusterIP service for internal communication within the cluster is created by default, except otherwise stated.
```
kubectl apply -f dbExternalNameSVc.yml
kubectl apply -f userMgt-LoadbSvc.yml
```
It is best practice to have the databse placed outside the kubernetes cluster to avoid single point of failure, achieve high scalability and availability, easy configuration management and quick data recovery. 

## Step 4: Auto scales at an avearge CPU of 70%.
Create a HorizontalPodAutoscaler object to scale the container at 70% CPU utilization.
In other to achieve this, compute resource was assigned to the container at the creation of the Deployment object.
HorizontalPodAutoscaler object uses the assigned resources to detwemine the 70% utilization
```
kubectl apply -f userMgt-hpa.yml
```
To accommmodate high traffic during the day and low traffic during the night
We need to create a CronJob object.
```
kubectl apply -f cronjob.yml
```
This CronJob object runs a `kubectl` command once a day at 6:00am to set the number of replicas in the deployment based on the current time of day.

## Step 5: Limit the access of the development team in the cluster to only deploy and rollback
To restrict the access of the development team, create the role and role binding objects using Role-Based Access Control RBAC authorization
```
kubectl apply -f role-roleBinding.yml
```

## Step 6: An alternative auto-scalling of the deployment based on network latency instead of CPU
In order to auto scale the deployment based on network latency, metrics of the pods needs to be generated.
This can be done using Helm chart to install Metric-server, Prometheus and Grafana.
Once the metics is scrapped using nodeExporter, then create the HorizontalPodAutoscaler that auto scales based on network latency.
```
kubectl apply -f hpa-networkLatency.yml
```
In this configuration, I've specified the metric property with a name and selector,
to select the network_latency metric from the Prometheus metrics server for pods with the app: userMgt-restapi label.
I've also specified the target property with a type of Value and an averageValue of 80ms,
which means that the HPA will scale the number of replicas of the deployment up or down to maintain an average network_latency value of 80 milliseconds per pod.

