# "AWS-codepipeline-ECS"
  
 I have implemented a full code pipeline incorportating commit, build and deploy steps.

The advanced demo consists of 4 stages :-

STAGE 1 : Configure Security & Create a CodeCommit Repo

STAGE 2 : Configure CodeBuild to clone the repo, create a container image and store on ECR

STAGE 3 : Configure a CodePipeline with commit and build steps to automate build on commit.

STAGE 4 : Create an ECS Cluster, TG's , ALB and configure the code pipeline for deployment to ECS Fargate and Route 53


:::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::


# STAGE 1 - CODE COMMIT

In this part of the advanced demo you will be creating and configuring a code commit repo as well as configuring access from your local machine.

## Generating an SSH key authenticate with codecommit

Full official instructions for Linux and macOS https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-ssh-unixes.html  

Full offical instructions for windows https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-ssh-windows.html
Run `cd ~/.ssh`
Run `ssh-keygen -t rsa -b 4096` and call the key 'codecommit', don't set any password for the key.  
Run `cat ~/.ssh/codecommit.pub` and copy the output onto your clipboard.  
In the AWS console move to the IAM console ( https://us-east-1.console.aws.amazon.com/iamv2/home?region=us-east-1#/home )  
Move to Users=>iamadmin & open that user  
Move to the `Security Credentials` tab.  
Under the AWS Codecommit section, upload an SSH key & paste in the copy of your clipboard.  
Copy down the `SSH key id` into your clipboard  

From your terminal, run `nano ~/.ssh/config` and at the top of the file add the following:

```
Host git-codecommit.*.amazonaws.com
  User KEY_ID_YOU_COPIED_ABOVE_REPLACEME
  IdentityFile ~/.ssh/codecommit
```

Change the `KEY_ID_YOU_COPIED_ABOVE_REPLACEME` placeholder to be the actual SSH key ID you copied above. 
Save the file and run a `chmod 600 ~/.ssh/config` to configure permissions correctly.  

Test with `ssh git-codecommit.us-east-1.amazonaws.com` and if successful it should show something like this.  

```
You have successfully authenticated over SSH. You can use Git to interact with AWS CodeCommit. Interactive shells are not supported.Connection to git-codecommit.us-east-1.amazonaws.com closed by remote host.
```

## Creating the code commit repo for cat pipeline

Move to the code commit console (https://us-east-1.console.aws.amazon.com/codesuite/codecommit/repositories?region=us-east-1)  
Create a repository.  
Call it `catpipeline-codecommit-XXX` where XXX is some unique numbers.  
Once created, locate the connection steps details and click `SSH`  
Locate the instructions for your specific operating system.  
Locate the command for `Clone the repository` and copy it into your clipboard.  
In your terminal, move to the folder where you want the repo stored. Generally i create a `repos` folder in my home folder using the `mkdir ~/repos` command, then move into this folder with `cd ~/repos`  
then run the clone command, which should look something like `ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/catpipeline-codecommit-XXX`  

## Adding the demo code

Download this file https://github.com/acantril/learn-cantrill-io-labs/raw/master/aws-codepipeline-catpipeline/01_LABSETUP/container.zip  
Copy the ZIP into the repo folder you created in the previous step.  
Extract the zip file into that folder.  
Delete the zip file  

then from the terminal move into that folder  
Run these commands.  

``` 
git add -A . 
git commit -m “container of cats” 
git push 

```
Ok so now you have a codecommit repo, with some code and are ready to move to the next step. Before proceeding to the next step you should be familiar with the Elastic Container Repo, we will be using this to store a docker image which we create from this source. In my video courses I have a theory lesson on ECR coming up next, if you are using this text only version - you will need to 1) do your own ECR research or 2) already be familiar with ECR.

:::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::

# STAGE 2 - CODE BUILD

Welcome to stage 2 of this demo where you will configure the Elastic Container Registry and use the codebuild service to build a docker container and push it into this registry.

## CREATE A PRIVATE REPOSITORY

Move to the Container Services console, the repositories (https://us-east-1.console.aws.amazon.com/ecr/repositories?region=us-east-1)  
Create a Repository.  
It should be a private repository.  
..and for the alias/name pick 'catpipeline'.  
Note down the URL and name (it should match the above name). 

This is the repository that codebuild will store the docker image in, created from the codecommit repo.   

## SETUP A CODEBUILD PROJECT

Next, we will configure a codebuild project to take what's in the codecommit repo, build a docker image & store it within ECR in the above repository.

Move to the codebuild console (https://us-east-1.console.aws.amazon.com/codesuite/codebuild/projects?region=us-east-1)  

Create code build project
  
### PROJECT CONFIGURATION
For `Project name` put `catpipeline-build`.  
Leave all other options in this section as default.  

### SOURCE
For `Source Provider` choose `AWS CodeCommit`  
For `Repository` choose the repo you created in stage 1  
Check the `Branch` checkbox and pick the branch from the `Branch` dropdown (there should only be one).  

### ENVIRONMENT
for `Environment image` pick `Managed Image`  
Under `Operating system` pick `Amazon Linux 2`  
under `Runtime(s)` pick `Standard`
under `Image` pick `aws/codebuild/amazonlinux2-x86_64-standard:X.0` where X is the highest number.  
Under `Image version` `Always use the latest image for this runtime version`  
Under `Envrironment Type` pick `Linux`  
Check the `Privileged` box (Because we're creating a docker image)  
For `Service role` pick `New Service Role` and leave the default suggested name which should be something like `codebuild-catpipeline-service-role`  
Expand `Additional Configuration`  
We're going to be adding some environment variables

Add the following:-

```
AWS_DEFAULT_REGION with a value of us-east-1
AWS_ACCOUNT_ID with a value of your AWS_ACCOUNT_ID_REPLACEME
IMAGE_TAG with a value of latest
IMAGE_REPO_NAME with a value of your ECR_REPO_NAME_REPLACEME
```

### BUILDSPEC
The buildspec.yml file is what tells codebuild how to build your code.. the steps involved, what things the build needs, any testing and what to do with the output (artifacts).

A build project can have build commands included... or, you can point it at a buildspec.yml file, i.e one which is hosted on the same repository as the code.. and that's what you're going to do.  

Check `Use a buildspec file`  
you don't need to enter a name as it will use by default buildspec.yml in the root of the repo. If you want to use a different name, or have the file located elsewhere (i.e in a folder) you need to specify this here.  

### ARTIFACTS
No changes to this section, as we're building a docker image and have no testing yet, this part isn't needed.

### LOGS

This is where the logging is configured, to Cloudwatch logs or S3 (optional).  

For `Groupname` enter `a4l-codebuild`  
and for `Stream Name` enter `catpipeline`  

Create the build Project

## BUILD SECURITY AND PERMISSIONS

Our build project will be accessing ECR to store the resultant docker image, and we need to ensure it has the permissons to do that. The build process will use an IAM role created by codebuild, so we need to update that roles permissions with ALLOWS for ECR.  

Go to the IAM Console (https://us-east-1.console.aws.amazon.com/iamv2/home#/home)  
Then Roles  
Locate and click the codebuild cat pipeline role i.e. `codebuild-catpipeline-build-service-role`  
Click the `Permissions` tab and we need to Add a permission and it will be an `inline policy`  
Select to edit the raw `JSON` and delete the skeleton JSON, replacing it with

```
{
  "Statement": [
	{
	  "Action": [
		"ecr:BatchCheckLayerAvailability",
		"ecr:CompleteLayerUpload",
		"ecr:GetAuthorizationToken",
		"ecr:InitiateLayerUpload",
		"ecr:PutImage",
		"ecr:UploadLayerPart"
	  ],
	  "Resource": "*",
	  "Effect": "Allow"
	}
  ],
  "Version": "2012-10-17"
}
```


Move on and review the policy.  
Name it `Codebuild-ECR` and create the policy
This means codebuild can now access ECR.  

## BUILDSPEC.YML

Create a file in the local copy of the `catpipeline-codecommit-XXX` repo called `buildspec.yml`  
Into this file add the following contents :-  (**use a code editor and convert tabs to spaces, also check the indentation is correct**)  

```
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t $IMAGE_REPO_NAME:$IMAGE_TAG .
      - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker image...
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
```

Then add this locally, commit and stage

``` 
git add -A .
git commit -m “add buildspec.yml”
git push 
```

## TEST THE CODEBUILD PROJECT

Open the CodeBuild console (https://us-east-1.console.aws.amazon.com/codesuite/codebuild/projects?region=us-east-1)  
Open `catpipeline-build`  
Start Build  
Check progress under phase details tab and build logs tab  

## TEST THE DOCKER IMAGE

Use this link to deploy an EC2 instance with docker installed (https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/quickcreate?templateURL=https://learn-cantrill-labs.s3.amazonaws.com/aws-codepipeline-catpipeline/ec2docker.yaml&stackName=DOCKER) accept all details, check the checkbox and create the stack.
Wait for this to move into the `CREATE_COMPLETE` state before continuing.  

Move to the EC2 Console ( https://us-east-1.console.aws.amazon.com/ec2/home?region=us-east-1#Home: )  
Instances  
Select `A4L-PublicEC2`, right click, connect  
Choose EC2 Instance Connect, leave everything with defaults and connect.  


Docker should already be preinstalled and the EC2 instance has a role which gives ECR permissions which you will need for the next step.

test docker via  `docker ps` command
it should output an empty list  


run a `aws ecr get-login-password --region us-east-1`, this command gives us login information for ECR which can be used with the docker command. To use it use this command.

you will need to replace the placeholder with your AWS Account ID (with no dashes)

`aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ACCOUNTID_REPLACEME.dkr.ecr.us-east-1.amazonaws.com`

Go to the ECR console (https://us-east-1.console.aws.amazon.com/ecr/repositories?region=us-east-1)  
Repositories  
Click the `catpipeline` repository  
For `latest` copy the URL into your clipboard  

run the command below pasting in your clipboard after docker p
`docker pull ` but paste in your clipboard after the space, i.e 
`docker pull ACCOUNTID_REPLACEME.dkr.ecr.us-east-1.amazonaws.com/catpipeline:latest` (this is an example, you will need your image URI)  

run `docker images` and copy the image ID into your clipboard for the `catpipeline` docker image

run the following command replacing the placeholder with the image ID you copied above.  

`docker run -p 80:80 IMAGEID_REPLACEME`

Move back to the EC2 console tab  
Click `Instances` and get the public IPv4 address for the A4L-PublicEC2 instance.  
open that IP in a new tab, ensuring it's http://IP not https://IP  
You should see the docker container running, with cats in containers... if so, this means your automated build process is working.  

:::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::

# STAGE 3 - CODE PIPELINE

In this stage of the demo you will create a code pipeline which will utilise CODECOMMIT and CODEBUILD to create a continous build process. The aim is that every time a new commit is made to the comecommit repo, a new docker image is created and pushed to ECR. Initially you will be skipping the codedeploy step which will be added next.

## PIPELINE CREATION

Move to the codepipeline console (https://us-east-1.console.aws.amazon.com/codesuite/codepipeline/pipelines)  
Create a pipeline  
For `Pipeline name` put `catpipeline`.  
Chose to create a `new service role` and keep the default name, it should be called `AWSCodePipelineServiceRole-us-east-1-catpipeline` 
Expand `Advanced Settings` and make sure that `default location` is set for the artifact store and `default AWS managed key` is set for the `Encryption key`  
Move on

### Source Stage

Pick `AWS CodeCommit` for the `Source Provider`  
Pick `catpipeline-codecommit` for the report name
Select the branch from the `Branch name` dropdown
From `Detection options` pick `Amazon CloudWatch Events` and for `Output artifact format` pick `CodePipeline default`  
Move on

### Build Stage

Pick `AWS CodeBuild` for the `Build provider`  
Pick `US East (N.Virginia)` for the `Region`  
Choose the project you created earlier in the `Project name` search box  
For `Build type` choose `Single Build`
Move on

### Deploy Stage

Skip deploy Stage, and confirm  
Create the pipeline

## PIPELINE TESTS
The pipeline will do an initial execution and it should complete without any issues.  
You can click details in the build stage to see the progress...or wait for it to complete  

Open the S3 console in a new tab (https://s3.console.aws.amazon.com/s3/) and open the `codepipeline-us-east-1-XXXX` bucket  
Click `catpipeline/`  
This is the artifacts area for this pipeline, leave this tab open you will be using in later in the demo.  

## UPDATE THE BUILD STAGE

Find your local copy of the catpipeline-codecommit repo and update the buildspec.yml file as below
**(use a code editor and convert tabs to spaces, also check the indentation is correct)**  

```
version: 0.2

phases:
  pre_build:
	commands:
	  - echo Logging in to Amazon ECR...
	  - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
	  - REPOSITORY_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME
	  - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
	  - IMAGE_TAG=${COMMIT_HASH:=latest}
  build:
	commands:
	  - echo Build started on `date`
	  - echo Building the Docker image...          
	  - docker build -t $REPOSITORY_URI:latest .
	  - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$IMAGE_TAG    
  post_build:
	commands:
	  - echo Build completed on `date`
	  - echo Pushing the Docker image...
	  - docker push $REPOSITORY_URI:latest
	  - docker push $REPOSITORY_URI:$IMAGE_TAG
	  - echo Writing image definitions file...
	  - printf '[{"name":"%s","imageUri":"%s"}]' "$IMAGE_REPO_NAME" "$REPOSITORY_URI:$IMAGE_TAG" > imagedefinitions.json
artifacts:
  files: imagedefinitions.json
```

## TEST A COMMIT

run

```
git add -A .
git commit -m "updated buildspec.yml"
git push
```

Watch the pipeline run, and generate a new version of the image in ECR automatically when the commit happens.

:::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::

STAGE 4 - CODE DEPLOY
In this stage, you will configure automated deployment of the cat pipeline application to ECS Fargate

Configure a load balancer
First, you're going to create a load balancer which will be the entry point for the containerised application

Go to the EC2 Console (https://us-east-1.console.aws.amazon.com/ec2/v2/home?region=us-east-1#Home:) then Load Balancing -> Load Balancers -> Create Load Balancer.
Create an application load balancer.
Call it catpipeline
Internet Facing IPv4 For network, select your default VPC and pick ALL subnets in the VPC.
Create a new security group (this will open a new tab) Call it catpipeline-SG and put the same for description Delect the default VPC in the list Add an inbound rule, select HTTP and for the source IP address choose 0.0.0.0/0 Create the security group.

Return to the original tab, click the refresh icon next to the security group dropdown, and select catpinepine-SG from the list and remove the default security group.

Under listners and routing make sure HTTP:80 is configured for the listner.
Create a target group, this will open a new tab call it catpipelineA-TG, ensure that IP, HTTP:80, HTTP1 and the default VPC are selected.
Click next and then create the target group, for now we wont register any targets.
Return to the original tab, hit the refresh icon next to target group and pick catpipelineA-TG from the list.
Then create the load balancer. This will take a few minutes to create, but you can continue on with the next part while this is creatign.

Configure a Fargate cluster
Move to the ECS console (https://us-east-1.console.aws.amazon.com/ecs/home?region=us-east-1#/getStarted) Clusters, Create a Cluster.
Move on, and name the cluster allthecatapps We will be using the default VPC so make sure it's selected and that all subnets in the VPC are listed.
Create the cluster.

Create Task and Container Definitions
Go to the ECS Cluster (https://us-east-1.console.aws.amazon.com/ecs/home?region=us-east-1#/clusters)
Move to Task Definitions and create a task definition.
Call it catpipelinedemo.
In Container Details, Name put catpipeline, then in Image URI move back to the ECR console and clikc Copy URI next to the latest image.
Scroll to the bottom and click Next

and for operating system family put Linux/X86_64
Pick 0.5vCPU for task CPU and 1GB for task memory.
Select ecsTaskExecutionRole under task role and task execution role.
Click Next and then Create.

DEPLOY TO ECS - CREATE A SERVICE
Click Deploy then Create Service. for Launch type pick FARGATE for Service Name pick catpipelineservice
for Desired Tasks pick 2 Expand Deployment Options, then for Deployment type pick rolling update Expand Networking.
for VPC pick the default VPC for Subnets make sure all subnets are selected.
for Security Group, choose User Existing Security Group and ensure that Default and catpipeline-SG are selected.
for public IP choose Turned On
for Load Balancer Type pick Application Load Balancer for Load balancer name pick catpipeline
for container to load balance select 'catpipeline:80:80'
select Use an existing Listener select 80:HTTP from the dropdown for choose Use an existing target group and for Target group name pick catpipelineA-TG Expand Service auto scaling and make sure it's not selected.
Click create
Wait for the service to finished deploying.

The service is now running with the :latest version of the container on ECR, this was done using a manual deployment

TEST
Move to the load balancer console (https://us-east-1.console.aws.amazon.com/ec2/v2/home?region=us-east-1#LoadBalancers)
Pick the catpipeline load balancer
Copy the DNS name into your clipboard
Open it in a browser, ensuring it is using http:// not https://
You should see the container of cats website - if it fits, i sits

ADD A DEPLOY STAGE TO THE PIPELINE
Move to the code pineline console (https://us-east-1.console.aws.amazon.com/codesuite/codepipeline/pipelines?region=us-east-1) Click catpipeline then edit Click + Add Stage
Call it Deploy then Add stage
Click + Add Action Group
for Action name put Deploy
for Action Provider put Amazon ECS
for Region pick US East (N.Virginia)
for Input artifacts select Build Artifact (this will be the imagedefinitions.json info about the container)
for Cluster Name pick allthecatapps
for Service Name pick catpipelineservice
for Image Definitions file put imagedefinitions.json
Click Done Click Save & Confirm

TEST
in the local repo edit the index.html file and add  - WITH AUTOMATION to the h1 line text. Save the file.
then run

git add -A .
git commit -m "test pipeline"
git push
watch the code pipeline console (https://us-east-1.console.aws.amazon.com/codesuite/codepipeline/pipelines/catpipeline/view?region=us-east-1)

make sure each pipeline step completes

Go back to the tab with the application open via the load balancer, refresh and see how the text has changed.

:::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::
