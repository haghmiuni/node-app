### node-app repo for deploying  a sample app docker image, Codebuild to ECR (CICD/ECS)

```

Pre-requisits:

Always test your app Locally and tag it  (see if works locally- enable access to port 80 in instance SG)

Clone my git
git clone https://github.com/haghmiuni/node-app.git

cd node-app

docker build -t my-custom-nginx .
docker images
docker run --name my-custom-nginx-container -p 80:8080 -d my-custom-nginx
docker ps -a

Try browsing to Public_IP_Of_Instance:80 

-------------------------------------------------------------------------------------

create ECR repository:
name=node-app

-------------------------------------------------------------------------------------

Create an ECS Cluster:
Name=nodeapp-cluster

-------------------------------------------------------------------------------------
Create a task defination:

- Task Definition Name=app-node
- select minimum values for vCPU and memory

Click Add Container:
Under Standar section:
name=nodeapp
Image=397374846593.dkr.ecr.us-east-1.amazonaws.com/node-app:latest
MEM Softlimit=128
Port mappings=8080

Under ENVIRONMENT:
CPU units=128

Click Create

-------------------------------------------------------------------------------------

Create Target Group FIRST:
Name=node-app-tg
Target type=IP
Protocol=HTTP
Port=8080

in Advance setion:
Healthy threshold=2

Click Create

-------------------------------------------------------------------------------------

Create ALB Next:

Name=node-app-alb

port=80
select 2 Subnets

next--->next ( configure Security Group)
select=all-sg

next---->Configure Routing
select above target group that was created)
node-app-alb

Next---> Register Targets ( do'nt select ANY!) 
Next---> create

-------------------------------------------------------------------------------------

Go to ECS Cluster
we will add a Service to our Cluster now and select it:

on Service tab---> create service:
Launch type=FARGATE
Task Definition=node-app
Cluster=nodeapp-cluster
service Nmae=nodeapp-service
Number of tasks=3
Deployment type=Rolling update

Click Next Step:

Cluster VPC=select our VPC
Subnets=select 2 Subnets
Security groups= Make sure click edit and select our existing all-sg

go to Load balancing section: 
Load balancer name= our loadbalancer name

Container to load balance section:
Container name:port= ( make sure it is =nodeapp:8080:8080 and CLICK "add to Load balancer")
Production listener port=80:HTTP
Target group name= select out Target group above
Enable service discovery integration=uncheck it!!!

Cleick Next:
Set Auto Scaling=No Set Auto Scaling

Click Next--->Create Service

-------------------------------------------------------------------------------------

At this Point, 3 Fargate Instances gets created. to check it go to service that we created and select "Task" tab and you will see 3 Fargate because we siad we want Desired count=3)

At this point also, these tasks will fail because we don't ANY image in ECR!!! So let's Push image to ECR:

on an linux instance cd to "~/projects/node-app" repo that was clone previously "Pre-requisits" of this doc. this GitHub repo contains a Dockerfile for our apps that we will create image and uploaded it ECR.


Go to ECR: click node-app repository and click "View Push commands" and on an linux instance with docker package installed run all of the commands

The following is the commands:
cd ~/projects/node-app

cred=$(aws ecr get-login-password --region us-east-1)

docker login --username AWS --password $cred 397374846593.dkr.ecr.us-east-1.amazonaws.com/node-app
docker build -t node-app .
docker images node-app
docker tag node-app:latest 397374846593.dkr.ecr.us-east-1.amazonaws.com/node-app:latest
docker images
docker push 397374846593.dkr.ecr.us-east-1.amazonaws.com/node-app:latest

Ckeck ECR for image with "latest" TAG

-------------------------------------------------------------------------------------

Let's do some cross checks:

Go to ECS and check 3 tasks are in running state.
Got to Target group an you should see 3 healthy Fargate instances (note: IPs in 2 different AZs!!!)
Go to ALB and get the endpoint and open it in the browser. You should see a page with "....app version 1 ...." content.

-------------------------------------------------------------------------------------

For CICD, we need first to create Codebuild:

Create Codebuild: ( Will pull buildspec.yml from node-app repo in my GitHub)
Click "create Build project"
project name=nodeapp
source=github ( make sure you are authenticated to your Github account) click "Repository in my GitHub account", select "node-app" repository
webhook=Check it and Event type=PUSH

In Environment section:

click Managed Image
Operating system=Ubuntu
Runtime(s)=standard
Image=standard:1.0
check Privileged="Enable this flag if you want to build Docker images or want your builds to get elevated privileges"

Service role: "New Service role" and name it as "codebuild-nodeapp-service-role-1"
Leave the rest of the settings with default settings
click Create "Build project"

Go to IAM and choose "codebuild-nodeapp-service-role-1" role and add modify it to add ECR permission as well!!! ( add "AmazonEC2ContainerRegistryFullAccess" Managed Policy.

NOW Start the buid the Build project above and check the "Build logs" and look for status of this Project Build. ( you should see that all stages in "buildspec.yml" are with status of success)
And go ECR and should see a new image with a Image tag ( this means that code build worked!)

-------------------------------------------------------------------------------------

Now we will take care of the CICD. We need to create Codepipeline:
Go to Codepipeline:

Pipeline name=pipeline-node-app
click Next
Source provider=GitHub
Connect to Github
Repository= select your repo ("haghmiuni/node-app")
Branch=master

Change detection options=GitHub webhooks (recommended)
click Next

in "Add build stage" page:

Build provider= Select CodeBuild
Project name= selct "i.e nodeapp"
click Next

In "Add deploy stage" page
Deploy provider= Amazon ECS
Cluster name=nodeapp-cluster
Service name=nodeapp-service
leave the rest of the settings as default
click Next
click "Create pipeline"

Codepipeline goes to all of the 3 stages ) Source---> Stage---Deploy) of provisioning ( it will take a few minutes to be complete so wait for it to be completed)
You can also check Codepipeline for it in "Execution history"

-------------------------------------------------------------------------------------

Now perform a push from your Github to invoke a pipeline:

modify server.js in node-app repo:

just chanage color: from red to blue or any color that you like. commit the chnages.

Check your pipeline status for automatically deploying the change of new image to Fargate containers (wait for it to be completed)

--------------------------------------------------------------------------------------
Open a Browser and type the link of ALB endpoint for port 80.

```


