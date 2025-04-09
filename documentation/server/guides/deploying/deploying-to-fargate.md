---
layout: page
title: Deploying to Fargate
---

In this guide, we will run an AWS Fargate service using the Docker image we pushed earlier. AWS Fargate allows you to deploy containerized applications without managing the underlying EC2 instances, simplifying the process of scaling and managing your application. We’ll walk through configuring the service to use your pre-built Docker image and highlight the key steps for a seamless deployment.

## Overview

This guide describes how to run an AWS FARGATE task using:

- A CloudWatch log group for logging
- Two IAM roles (one for task execution and one for container operations)
- An ECS cluster
- A task definition that specifies the Docker image from ECR

Later in this tutorial, before creating the AWS task, make sure you will have the following information ready:

- **Task Execution Role ARN:** (e.g., `arn:aws:iam::<account-id>:role/ecsTaskExecutionRole`)
- **Container Role ARN:** (e.g., `arn:aws:iam::<account-id>:role/ecsTaskRole`)
- **Docker Image URI:** Your image in ECR (e.g., `963008249569.dkr.ecr.eu-central-1.amazonaws.com/swift-builder`)
- **CloudWatch Log Group Name:** As created in CloudWatch Logs
- **PostgreSQL Database Endpoint:** The endpoint of your previously created RDS database (e.g., `test-database-instance.cgc65h5t3cov.eu-central-1.rds.amazonaws.com`)

## Create a CloudWatch Log Group

1. Open the **CloudWatch** console and navigate to **Logs** → **Log groups**.
2. Click on **Create log group**.
3. Enter your desired log group name and click **Create**.

## Create IAM Roles for the Task

### Task Execution Role

This role gives the task permission to create CloudWatch logs.

1. Open the **IAM** console and go to **Roles**.
2. Click on **Create role**.
3. Under **Trusted entity**, select **AWS service**, then choose **Elastic Container Service**.
4. Choose **Elastic Container Service Task** as the use case and click **Next**.
5. Select the following policy:
    - `AmazonECSTaskExecutionRolePolicy`
6. Click **Next**, provide a role name (e.g., `ecsTaskExecutionRole`), and click **Create role**.
7. Open the newly created role and note its ARN (e.g., `arn:aws:iam::<account-id>:role/ecsTaskExecutionRole`).

### Container Role

This role allows the running container to access resources such as EFS. For now, we’ll create the role without attaching any specific permissions.

1. Open the **IAM** console and go to **Roles**.
2. Click on **Create role**.
3. Under **Trusted entity**, select **AWS service**, then choose **Elastic Container Service**.
4. Choose **Elastic Container Service Task** as the use case and click **Next**.
5. We don’t need any specific policy at this stage, so leave all policies unselected.
6. Click **Next**, provide a role name (e.g., `ecsTaskContainerRole`), and click **Create role**.
7. Open the newly created role and note its ARN (e.g., `arn:aws:iam::<account-id>:role/ecsTaskRole`).

## Create an ECS Cluster

1. Open the **Elastic Container Service (ECS)** console.
2. Go to **Clusters** and click on **Create cluster**.
3. Choose **AWS Fargate (serverless)**.
4. Enter a name for your cluster and click **Create**.

## Prepare the Task Definition

Next, create a new task definition using the JSON editor. Navigate to Elastic Container Service → Task Definitions, and click "Create new task definition with JSON." Then, copy the following template and replace the `<...>` placeholders with your specific values:

```json
{
  "family": "<arbitrary_family_name>",
  "networkMode": "awsvpc",
  "requiresCompatibilities": [
    "FARGATE"
  ],
  "cpu": "16384",
  "memory": "32768",
  "executionRoleArn": "<task_execution_role_ARN>",
  "taskRoleArn": "<task_role_ARN>",
  "containerDefinitions": [
    {
      "name": "<arbitrary_container_name>",
      "image": "<your_docker_image_URI>",
      "cpu": 16384,
      "memory": 32768,
      "essential": true,
      "portMappings": [
        {
          "containerPort": 8080,
          "hostPort": 8080,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "DATABASE_HOST",
          "value": "<aws_database_endpoint_here>"
        },
        {
          "name": "DATABASE_PORT",
          "value": "5432"
        },
        {
          "name": "DATABASE_USER",
          "value": "postgres"
        },
        {
          "name": "DATABASE_PASSWORD",
          "value": "postgres"
        },
        {
          "name": "DATABASE_NAME",
          "value": "postgres"
        },
        {
          "name": "ROOT_CERT_PATH",
          "value": "eu-central-1-bundle.pem"
        }
      ],
      "command": [
        "--hostname",
        "0.0.0.0",
        "--port",
        "8080"
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "<log_group_name>",
          "awslogs-region": "<region>",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

After creating the task definition, it will appear in your task definitions list under the provided family name.


## Fargate Service Creation

Follow these steps to create and run your Fargate service:

1. **Access the ECS Cluster:**
    
    Go to the **Elastic Container Service (ECS)** console, navigate to **Clusters**, and open the cluster you created earlier.
    
2. **Create a Service:**
    
    Under the **Services** tab, click on the **Create** button.
    
3. **Configure Compute Settings:**
    - In the **Compute configuration (advanced)** section, the default values are acceptable.
    - Choose **Capacity provider strategy** with a single FARGATE provider.
    - Set the **Platform version** to **LATEST**.
4. **Deployment Configuration:**
    - In the **Deployment configuration** section, select **Application type** as **Service**.
    - Choose the task definition family that you created earlier.
    - Enter a custom service name.
5. **Networking Configuration:**
    - In the **Networking** section, create a new security group for the service.
    - Provide a security group name, for example: `hummingbird-todos-service-sg`, and add a brief description.
    - For inbound rules, add a rule with the following settings:
        - **Type:** Customized TCP
        - **Port Range:** 8080
        - **Source:** Anywhere (this allows anyone to access the service for testing purposes)
    - Enable the **Public IP address** option.
6. **Create the Service:**
    
    Click on the **Create** button. Wait until the service starts a task and it is in **RUNNING** status.
    
7. **Access the Service:**
    - Once the service starts the task, open the service and navigate to the **Task** tab.
    - Select the running task; within its configuration section, locate the public IP address.
    - Test your web service by executing a command similar to:
        
        ```bash
        curl -X GET http://<public-ip>:8080/todos
        ```
        

## Disabling the Service

To temporarily stop your service, set the **Desired tasks** count to `0`. This will stop all running tasks associated with the service, but preserve the service configuration for later use.

This completes the AWS Fargate service creation. 


