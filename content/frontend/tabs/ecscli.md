---
title: "Embedded tab content"
disableToc: true
hidden: true
---

## Set environment variables from our build
```
export clustername=$(aws cloudformation describe-stacks --stack-name fargate-demo --query 'Stacks[0].Outputs[?OutputKey==`ClusterName`].OutputValue' --output text)
export target_group_arn=$(aws cloudformation describe-stack-resources --stack-name fargate-demo-alb | jq -r '.[][] | select(.ResourceType=="AWS::ElasticLoadBalancingV2::TargetGroup").PhysicalResourceId')
export vpc=$(aws cloudformation describe-stacks --stack-name fargate-demo --query 'Stacks[0].Outputs[?OutputKey==`VpcId`].OutputValue' --output text)
export ecsTaskExecutionRole=$(aws cloudformation describe-stacks --stack-name fargate-demo --query 'Stacks[0].Outputs[?OutputKey==`ECSTaskExecutionRole`].OutputValue' --output text)
export subnet_1=$(aws cloudformation describe-stacks --stack-name fargate-demo --query 'Stacks[0].Outputs[?OutputKey==`PrivateSubnetOne`].OutputValue' --output text)
export subnet_2=$(aws cloudformation describe-stacks --stack-name fargate-demo --query 'Stacks[0].Outputs[?OutputKey==`PrivateSubnetTwo`].OutputValue' --output text)
export subnet_3=$(aws cloudformation describe-stacks --stack-name fargate-demo --query 'Stacks[0].Outputs[?OutputKey==`PrivateSubnetThree`].OutputValue' --output text)
export security_group=$(aws cloudformation describe-stacks --stack-name fargate-demo --query 'Stacks[0].Outputs[?OutputKey==`ContainerSecurityGroup`].OutputValue' --output text)

cd ~/environment
```
This creates our infrastructure, and sets several environment variables we will use to
automate deploys.

## Configure `ecs-cli` to talk to your cluster:
```
ecs-cli configure --region $AWS_REGION --cluster $clustername --default-launch-type FARGATE --config-name fargate-demo
```
We set a default region so we can reference the region when we run our commands.


## Authorize traffic:
```
aws ec2 authorize-security-group-ingress --group-id "$security_group" --protocol tcp --port 3000 --cidr 0.0.0.0/0
```
We know that our containers talk on port 3000, so authorize that traffic on our security group:

## Deploy our frontend application:
```
cd ~/environment/ecsdemo-frontend
envsubst < ecs-params.yml.template >ecs-params.yml

ecs-cli compose --project-name ecsdemo-frontend service up \
    --target-group-arn $target_group_arn \
    --private-dns-namespace service \
    --enable-service-discovery \
    --container-name ecsdemo-frontend \
    --container-port 3000 \
    --cluster-config fargate-demo \
    --vpc $vpc
    
```
Here, we change directories into our frontend application code directory.
The `envsubst` command templates our `ecs-params.yml` file with our current values.
We then launch our frontend service on our ECS cluster (with a default launchtype 
of Fargate)

Note: ecs-cli will take care of building our private dns namespace for service discovery,
and log group in cloudwatch logs.

## View running container:
```
ecs-cli compose --project-name ecsdemo-frontend service ps \
    --cluster-config fargate-demo
```
We should have one task registered.

## Check reachability (open url in your browser):
```
alb_url=$(aws cloudformation describe-stacks --stack-name fargate-demo-alb --query 'Stacks[0].Outputs[?OutputKey==`ExternalUrl`].OutputValue' --output text)
echo "Open $alb_url in your browser"
```
This command looks up the URL for our ingress ALB, and outputs it. You should 
be able to click to open, or copy-paste into your browser.

## View logs:
```
#substitute your task id from the ps command 
ecs-cli logs --task-id a06a6642-12c5-4006-b1d1-033994580605 \
    --follow --cluster-config fargate-demo
```
To view logs, find the task id from the earlier `ps` command, and use it in this
command. You can follow a task's logs also.

## Scale the tasks:
```
ecs-cli compose --project-name ecsdemo-frontend service scale 3 \
    --cluster-config fargate-demo
ecs-cli compose --project-name ecsdemo-frontend service ps \
    --cluster-config fargate-demo
```
We can see that our containers have now been evenly distributed across all 3 of our
availability zones.


