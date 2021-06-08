# Docker-Compose App → Amazon ECS App

*NOTE* : This demo/code is not-production ready, its sole goal is to let you experiment.

You will incur costs please remember to clean resources.

From local docker build, using the docker-compose construct to an ECS App 

This demo is inspired by this blog: https://aws.amazon.com/blogs/containers/deploy-applications-on-amazon-ecs-using-docker-compose/

## Integration Setup


 * It is assumed that you have an active AWS CLI profile, credentials and the required IAM policy permissions.
    
    Start here (https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html) if no AWS CLI configuration exist.
    If you plan to use an EC2 instance create and associate the proper IAM Role.
    
    See the docker ECS integration Requirements section here (https://docs.docker.com/cloud/ecs-integration/#:~:text=The%20integration%20between%20Docker%20and,run%20applications%20quickly%20and%20easily) for detailed required policy permissions
    You will also need IAM policy permissions to bootstrap the infra Cloudformation stack. (VPC, ECR, and more.. check CFN template)
    
 * Grab docker desktop, https://hub.docker.com/signup/awsedge?utm_source=awsedge
    *Note*: This will also work on ubuntu server 20.04, setup is beyond this guide:
     Start here for ubuntu setup:
       https://docs.docker.com/engine/install/ubuntu/
       https://docs.docker.com/compose/install/

 *  Create the docker context (More info: https://docs.docker.com/cloud/ecs-integration/)

    `docker context create ecs myecscontext`
    
    Make sure you choose the proper AWS CLI profile that points to the proper region, I tested an AWS CLI profile using the eu-central-1 Region

    List the contexts:

    `docker context ls`

    This should return the default and myecscontext contexts (the default is used for local deployment)
    Mind the * this is the active context

 * Clone the Repo

   git clone https://github.com/kbiton/docker-ecs-app.git

 * Deploy the Infra stack: This is an opinionated stack that deploys a VPC, ALB, ECS Repo and few VPC Endpoints, Feel free to customize as you see fit.
    
    `cd infra/`

    `aws cloudformation create-stack --stack-name infra-docker-ecs --template-body file://cfn-infra.yaml --capabilities CAPABILITY_IAM`

    *note*: This will deploy resources into your account and you will incur costs!
    
 * Get the required outputs for the docker compose ecs integration
    
    `aws cloudformation describe-stacks --stack-name infra-docker-ecs  --query "Stacks[0].Outputs[0].OutputValue"`

    Output should be similar  → "vpc-xxxxxxxxxxxxxxxxx"

    `aws cloudformation describe-stacks --stack-name infra-docker-ecs  --query "Stacks[0].Outputs[3].OutputValue"`

    Output should be similar  → "infra-docker-ecs-alb"

    
 * Edit `app/env.sh` and populate the (VPC and Load Balancer Vars) values.

   Complete the ECR endpoint with your AWS account ID
   
   Source the env:

   `. app/env.sh`
    
 * This demo uses Amazon ECR (A repository has been created using the infra cloudformation stack)

    *note*: Your IAM user should be able to push/pull images.

    Authenticate your Docker client with the registry: 

    `aws ecr get-login-password --region eu-central-1 | docker login --username AWS --password-stdin
    ${DOCKER_REGISTRY}`

    I prefer the ECR credentials helper: https://github.com/awslabs/amazon-ecr-credential-helper

 * Build and push the image

   `docker context ls`

   `docker context use default`

   `docker context ls`  | Validate that the default context is active (* mark)

   `docker-compose build`

   `docker-compose push`

   If successful, your image should be build locally and pushed to the ECR Repo

 * Deploy the application to Amazon ECR

   `docker context ls`

   `docker context use myecscontext`

   `docker context ls`  | Validate that the ECR context is active (* mark)

   `docker compose up`  | Mind the "docker compose up and NOT "docker-compose up"

   At this point docker reads the compose file and creates a Cloudformation template (on the fly) then deploys it to AWS.

   Head over to the AWS Console and examine the Cloudformation stack

 * Grab the LB endpoint:

   `aws cloudformation describe-stacks --stack-name infra-docker-ecs  --query "Stacks[0].Outputs[1].OutputValue"`


## Cleanup


 * While the docker context points to your ECS based context

    `cd app/` 

    `docker compose down` | This will tear down the docker ECS cloudformation stack    

    `aws cloudformation delete-stack --stack-name infra-docker-ecs` | This will tear down the infra stack

           
 * Remove the ECR Image(s) and Repo

     `aws ecr batch-delete-image --repository-name "dev/prjneo/frontend" --image-ids imageTag=latest`

     `aws ecr delete-repository --repository-name “dev/prjneo/frontend”`


