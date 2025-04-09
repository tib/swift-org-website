---
layout: page
title: Setting up Amazon RDS 
---

Amazon RDS (Relational Database Service) is a managed service for setting up, operating, and scaling relational databases in the cloud. This guide covers the steps to create a PostgreSQL RDS service on AWS. 

See the [packaging guide](/documentation/server/guides/packaging/) for detailed instructions on setting up a local database service with a secure TLS connection. It also covers the initial configuration steps and how to generate the necessary certificates.

In this guide, the AWS Console is used to create a new PostgreSQL database instance with custom configuration.

## Step-by-Step Instructions

Access the [AWS Console](https://aws.amazon.com/console/). Navigate to **Aurora and RDS** by using the search:

![Amazon RDS](/assets/images/server-guides/amazon-rds/rds-01.png)

Select **Databases** from the side menu:

![Amazon RDS](/assets/images/server-guides/amazon-rds/rds-02.png)

Click on the **Create Database** button:

![Amazon RDS](/assets/images/server-guides/amazon-rds/rds-03.png)

Choose a **Database Creation Method:** select **Standard create** select **PostgreSQL** as the engine type:

![Amazon RDS](/assets/images/server-guides/amazon-rds/rds-04.png)

Select the desired engine version and pick a tier based on your needs:

![Amazon RDS](/assets/images/server-guides/amazon-rds/rds-05.png)

> NOTE: the **Free tier** uses a single instance without redundancy.

Set a **DB Instance Identifier** for example, set it to **database-1**. Under the **Credentials** section specify the master username and a custom password,  pick the **Self managed** option and enter the desired password:

![Amazon RDS](/assets/images/server-guides/amazon-rds/rds-06.png)


Under **Instance Configuration**, choose **db.t4g.micro** as the instance type (or any other type as needed). For **Storage**, select **gp2** as the **Type** and set **Allocated Storage** to **20 GB**. These options may vary based on specific requirements:

![Amazon RDS](/assets/images/server-guides/amazon-rds/rds-07.png)


- **VPS / VPC / VPN - Connectivity:**
    - Do not use an EC2 compute resource; select the default VPS and subnet group.
    - For public access, create a new security group (e.g., **postgres-sg**).
    - **TODO**: how to enable / disable public access?
- **Other Settings:**
    - Leave additional settings at their default values.
    - Enable all **Log Exports**.
- **Database Name:** Use the default name **postgres**.


![Amazon RDS](/assets/images/server-guides/amazon-rds/rds-08.png)

**Security Group:**
- ⚠️ The created security group will automatically set the inbound rule source to your actual IP address. (based on your local machine IP address or router IP address).
- **Optional:** If you want to allow access from any IP (for example, if your IP address changes frequently), you can modify the auto-created security group:
    - Go to **EC2 > Security Groups**.
    - Find and select the security group (e.g., **postgres-sg**).
    - Under **Inbound rules**, click **Edit inbound rules**.
    - Change the **Source** from your specific IP (e.g., `203.0.113.42/32`) to **`0.0.0.0/0`** to allow all incoming connections on the specified port (typically 5432 for PostgreSQL).
    - ⚠️ **Note:** Setting the source to `0.0.0.0/0` allows connections from **any IP address**, making the database **accessible to everyone on the internet** if other conditions permit. While this doesn’t automatically expose the database publicly, it significantly increases potential exposure. Ensure strong credentials are used, and consider restricting access by IP or using a VPN for improved security, especially in production environments.
- NOTE: not enough to publicly access from outside.

![Amazon RDS](/assets/images/server-guides/amazon-rds/rds-09.png)

Click on create database to finish:

![Amazon RDS](/assets/images/server-guides/amazon-rds/rds-10.png)

## Testing the connection

You can test your Amazon RDS PostgreSQL database connection by using the psql command (you need to install the postgresql tools in order to use it):

```sh
# brew on macOS 
brew install postgresql
# apt on Linux
apt-get install postgresql
# yum on Linux
yum install postgresql
```

### Without Certificate Verification

```sh
psql \
-h your_amazon_rds_database_endpoint \
-p 5432 \
-U your_amazon_rds_user_name \
-d your_amazon_rds_database_name
```

### With Certificate Verification

You can use Secure Socket Layer (SSL) or Transport Layer Security (TLS) from your application to encrypt a connection to a database running and RDS instance.

The certificate files (e.g. `eu-central-1-bundle.pem` for the EU central 1 region) are available online at the [Amazon User Guide RDS with SSL page](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.SSL.html#UsingWithRDS.SSL.CertificatesAllRegions).

Please download the one for your selected region and use the following command to test the secure connection:

```sh
PGSSLMODE=verify-full \
PGSSLROOTCERT=eu-central-1-bundle.pem \
psql \
-h your_amazon_rds_database_endpoint \
-p 5432 \
-U your_amazon_rds_user_name \
-d your_amazon_rds_database_name
```

> This example assumes that the RDS instance is hosted in the eu-central-1 (Frankfurt) AWS region. Be sure to change the certificate file accordingly if your setup differs.


## Accessing RDS using Docker

In this section, we explain how to access the previously created Amazon RDS PostgreSQL database service from your local Swift web service using Docker. We are going to update the sample app from the [packaging guide](/documentation/server/guides/packaging). 

We have to move the certificate file over to the container, this can be done by updating the `Dockerfile` for the Hummingbird server:

```Dockerfile
FROM swift:6.1-noble AS build
WORKDIR /build
COPY ./Package.* ./
RUN swift package resolve
COPY ./ .
RUN swift build -c release --static-swift-stdlib
WORKDIR /staging
RUN cp "$(swift build --package-path /build -c release --show-bin-path)/Server" ./
RUN cp "/usr/libexec/swift/linux/swift-backtrace-static" ./
RUN find -L "$(swift build --package-path /build -c release --show-bin-path)/" -regex '.*\.resources$' -exec cp -Ra {} ./ \;

FROM ubuntu:noble
RUN export DEBIAN_FRONTEND=noninteractive DEBCONF_NONINTERACTIVE_SEEN=true \
    && apt-get -q update \
    && apt-get -q dist-upgrade -y \
    && rm -r /var/lib/apt/lists/*
RUN useradd --user-group --create-home --system --skel /dev/null --home-dir /app hummingbird
WORKDIR /app

COPY --from=build --chown=hummingbird:hummingbird /staging /app

# Copy additional SSL certificate files needed for the application
COPY ./docker/server/eu-central-1-bundle.pem /app/

ENV SWIFT_BACKTRACE=enable=yes,sanitize=yes,threads=all,images=all,interactive=no,swift-backtrace=./swift-backtrace-static
USER hummingbird:hummingbird
EXPOSE 8080
ENTRYPOINT ["/app/Server"]
CMD ["--hostname", "0.0.0.0", "--port", "8080"]
```


You should also re-configure the Docker Compose file to use the real AWS PostgreSQL endpoint, verify the SSL connection using the `eu-central-1-bundle.pem` (bundled into the Hummingbird server image):

```yaml
services:
  server:
    build:
      context: .
      dockerfile: docker/server/Dockerfile
    ports:
      - "8080:8080"
    environment:
      DATABASE_HOST: your_amazon_rds_database_endpoint
      DATABASE_PORT: 5432
      DATABASE_USER: your_amazon_rds_user_name
      DATABASE_PASSWORD: your_amazon_rds_password
      DATABASE_NAME: your_amazon_rds_database_name
      ROOT_CERT_PATH: eu-central-1-bundle.pem
    command: ["--hostname", "0.0.0.0", "--port", "8080"]
```


Build and run the docker image using the docker build or the compose command and test the running service with `curl` or any other tool similarly like we di d in the packaging guide.


```sh
docker compose up

# create a todo
curl -X POST http://localhost:8080/todos \
  -H "Content-Type: application/json" \
  -d '{"title": "Finish the Amazon RDS integration", "order": 1}'

# list todos
curl -X GET http://localhost:8080/todos
```

This will launch the web service on port `8080` with the proper configuration for connecting to the AWS-hosted PostgreSQL RDS. After starting the container, verify that the service is running correctly by using the `curl` commands.

These tests confirm that the web service is operational and correctly interacting with your AWS RDS instance over a secure SSL connection.


> ⚠️ WARNING: This tutorial exposes the Amazon RDS to the public, which is not recommended and not a production ready secure solution.  

In follow-up tutorial, we explain how to create a VPN connection to Amazon AWS. This will allow you to access the RDS instance in a more secure way by restricting access to authorized networks only, thereby enhancing the overall security of your application.


