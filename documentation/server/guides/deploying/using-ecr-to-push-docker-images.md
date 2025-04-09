---
layout: page
title: Using ECR to push Docker images
---

In this tutorial, you will learn how to create an AWS user, assign essential policies, and generate access keys for CLI-based operations. This setup is fundamental for managing AWS services like Amazon ECS, Amazon ECR, and CloudWatch Logs. By the end of this article, we're going to move the HTTP server's docker image to Amazon's infrastructure.


## User creation

### Open AWS IAM Console

- Navigate to **AWS Identity and Access Management (IAM)**.
- In the IAM dashboard, select **Users**.
- Click on the **Create user** button.

### User Details

- **User Name:** Enter your desired user name.
- **Access Type:** Do **not** enable AWS Management Console access; this user will be used for CLI interactions only.

### Attach Policies

- Click **Next** to proceed to permissions.
- Select **Attach policies directly**. This is the simplest way to grant the necessary permissions.
- Search and attach the following policies:
    - **AmazonECS_FullAccess:** Grants full access to Amazon Elastic Container Service (ECS), including clusters, tasks, and CloudWatch logs.
    - **AmazonEC2ContainerRegistryFullAccess:** Provides permissions for managing Amazon Elastic Container Registry (ECR), such as pushing/pulling images and creating repositories.
    - **CloudWatchLogsReadOnlyAccess:** Allows read-only access to CloudWatch Logs.

### Review and Create User

- Click **Next** to review your configuration.
- Verify that all selections are correct.
- Click **Create user** to finalize the user creation.

### Create Access Key for CLI Access

- From the user list, select the newly created user.
- Click on **Create access key**.
- Choose the **Command Line Interface (CLI)** option to enable CLI access.
- Confirm by checking the box: "I understand the above recommendation and want to proceed to create an access key."
- Click **Next** to proceed.

### Tag the Access Key (Optional)

- On the subsequent page, you have the option to add a tag to the access key for organizational purposes.
- Complete this step if desired, then proceed.

### Save Your Credentials

- You will be presented with an **Access key** and a **Secret access key**.
- **Important:** Save the Secret access key in a secure location. **Note:** If you lose or forget your Secret access key, you cannot retrieve it. Instead, you will need to create a new access key and deactivate the old one.


# AWS CLI Configuration

This section walks you through configuring the AWS CLI after generating your access key and secret access key. With the AWS CLI properly configured, you’ll be able to manage your AWS services directly from the command line.

### Installing the AWS CLI

If you haven’t installed the AWS CLI yet, you can do so using Homebrew on MacOS:

```bash
brew install awscli
```

### Configuring the AWS CLI

Run the following command to set up your AWS CLI credentials and configuration:

```bash
aws configure
```

During the configuration, you will be prompted for the following details:

- **AWS Access Key ID:** Enter your access key.
- **AWS Secret Access Key:** Enter your secret access key.
- **Default region name:** Specify the region your account uses (you can find the current region in the top-right corner of the AWS Management Console, e.g., *N. Virginia (us-east-1)*).
- **Default output format:** Leave this blank to use the default JSON format.

### Verifying the Configuration

After completing the configuration, verify your settings by running:

```bash
aws configure list
```

The AWS CLI stores the credentials in the `~/.aws/credentials` file and the configuration settings in the `~/.aws/config` file.

## Testing Your Configuration

Test your AWS CLI setup by listing your S3 buckets:

```bash
aws s3 ls
```

**Expected outcomes:**

- **Successful configuration:** The command will list your S3 buckets.
- **Access Denied / No Buckets:**
An "Access Denied" error or an empty response (no buckets) is an expected outcome if your account does not have any S3 buckets or lacks permission for that specific operation. This outcome confirms that your CLI connection is set up correctly.
- **Authenticator Issues:**
If you encounter errors related to authentication (such as invalid credentials or signature mismatches), this indicates a bad CLI configuration. In other words, an authenticator issue signals that your AWS Access Key ID and Secret Access Key may be misconfigured or entered incorrectly.


## Building & Pushing the Docker Image to AWS ECR

In this section, we describe how to build a Docker image using Docker Compose and push it to an AWS ECR repository. 

### Building the Docker Image

Instead of using the traditional `docker build` command, we use Docker Compose to build the image. The compose file defines the service along with the build context, Dockerfile, ports, environment variables, and run command.

TODO: link other article or embed the compose file.

To build the Docker image using the configuration, run the following command:

```bash
docker compose build
```

This command reads the compose file, builds the image as defined (using the specified Dockerfile and build context), and prepares the container configuration including ports and environment variables.

After building the image, list all Docker images to verify that the newly built image is named with the folder prefix (e.g. `<folder-name>-<image-name>`):

```bash
docker image ls
```

### Creating the AWS ECR Repository

Before tagging and pushing your Docker image, you need to create a repository on AWS ECR.

1. **Using the AWS Console:**
    - Navigate to **Amazon Elastic Container Registry** and select **Create repository**.
2. **Using the CLI:**
    - Execute the following command (we are using `web-service-repo` as the repository name):

```bash
aws ecr create-repository --repository-name web-service-repo
```

1. **Obtain the Repository URI:**
    - After creation, copy the repository URI (e.g., `163008249569.dkr.ecr.eu-central-1.amazonaws.com/web-service-repo`).

### Tagging and Pushing the Docker Image

1. **Tag the Local Image:**
    - Tag your local image using the repository URI. Replace `<repository_uri>` with your actual repository URI:

```bash
docker tag <my-local-image-name>:latest <repository_uri>:latest
```

- **Example:**

```bash
docker tag hummingbird-todos:latest 163008249569.dkr.ecr.eu-central-1.amazonaws.com/web-service-repo:latest
```

1. **Authenticate with AWS ECR:**
    - Log in to AWS ECR with the following command. Replace `<region>` and `<registry_uri>` appropriately (note that `<registry_uri>` is the repository URI without the path):

```bash
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <registry_uri>
```

- **Example:**

```bash
aws ecr get-login-password --region eu-central-1 | docker login --username AWS --password-stdin 163008249569.dkr.ecr.eu-central-1.amazonaws.com
```

1. **Push the Docker Image:**
    - Finally, push the tagged image to your AWS ECR repository:

```bash
docker push <repository_uri>:latest
```

- **Example:**

```bash
docker push 163008249569.dkr.ecr.eu-central-1.amazonaws.com/web-service-repo:latest
```

> Tip: You can also find these push commands in the AWS Console under Amazon Elastic Container Registry -> Repositories -> select your repository -> View push commands. Note that the commands shown there might use docker build instead of docker compose build.

With these steps, you have successfully built your Docker image using Docker Compose and pushed it to AWS ECR.

