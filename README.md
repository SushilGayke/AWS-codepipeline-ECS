# AWS-codepipeline-ECS
  
 I have implemented a full code pipeline incorportating commit, build and deploy steps.

The advanced demo consists of 4 stages :-

STAGE 1 : Configure Security & Create a CodeCommit Repo

STAGE 2 : Configure CodeBuild to clone the repo, create a container image and store on ECR

STAGE 3 : Configure a CodePipeline with commit and build steps to automate build on commit.

STAGE 4 : Create an ECS Cluster, TG's , ALB and configure the code pipeline for deployment to ECS Fargate and Route 53

