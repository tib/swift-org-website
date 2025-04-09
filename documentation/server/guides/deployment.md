---
redirect_from: "server/guides/deployment"
layout: page
title: Deploying to Servers or Public Cloud
---

The following guides can help with the packaging and deployment of a server-side Swift application to private servers or public cloud providers.

## Packaging

<ul class="grid-level-0">
    <li class="grid-level-1">
        <h3>Docker</h3>
        <p>This guide explains how to use Docker to build and run a server-side Swift application on your local machine. It demonstrates how to create Docker images and use Docker Compose to host the required infrastructure locally, including a PostgreSQL database service with a TLS connection.</p>
        <a href="/documentation/server/guides/packaging" class="cta-secondary">Packaging Applications using Docker</a>
    </li>
    <li class="grid-level-1">
        <h3>Static Linux SDK</h3>
        <p>The Swift Static Linux SDK allows you to build your program as a fully statically linked executable, with no external dependencies at all (not even the C library), which means that it will run on any Linux distribution as the only thing it depends on is the Linux system call interface.</p>
        <a href="/documentation/articles/static-linux-getting-started" class="cta-secondary">Getting Started with the Static Linux SDK</a>
    </li>
</ul>


## Deploying to AWS

<ul class="grid-level-0">
    <li class="grid-level-1">
        <h3>Setting up Amazon RDS</h3>
        <p>Amazon RDS (Relational Database Service) is a managed service for setting up, operating, and scaling relational databases in the cloud. This guide covers the steps to create a PostgreSQL RDS service on AWS.</p>
        <a href="/documentation/server/guides/deploying/setting-up-amazon-rds/" class="cta-secondary">Read this guide</a>
    </li>
    <li class="grid-level-1">
        <h3>Securing RDS & using a VPN </h3>
        <p>Private access to RDS</p>
        <a href="/documentation/server/guides/deploying/securing-rds-and-using-a-vpn/" class="cta-secondary">Read this guide</a>
    </li>    
    <li class="grid-level-1">
        <h3>Using ECR to push Docker images</h3>
        <p>Lorem ipsum dolor sit amet</p>
        <a href="/documentation/server/guides/deploying/using-ecr-to-push-docker-images/" class="cta-secondary">Read this guide</a>
    </li>
    <li class="grid-level-1">
        <h3>Deploying to Fargate</h3>
        <p>Deploying from ECR to Fargate as a service.</p>
        <a href="/documentation/server/guides/deploying/deploying-to-fargate/" class="cta-secondary">Read this guide</a>
    </li>
    <li class="grid-level-1">
        <h3>Deploying to Fargate using Vapor & MongoDB</h3>
        <p>This guide illustrates how to deploy a Server-Side Swift workload on AWS. The workload is a REST API for tracking a To Do List. It uses the Vapor framework to program the API methods. The methods store and retrieve data in a MongoDB Atlas cloud database. The Vapor application is containerized and deployed to AWS on AWS Fargate using the AWS Copilot toolkit.</p>
        <a href="/documentation/server/guides/deploying/aws-copilot-fargate-vapor-mongo" class="cta-secondary">Read this guide</a>
    </li>
    <li class="grid-level-1">
        <h3>Setting up EC2</h3>
        <p>This guide describes how to launch an AWS instance running Amazon Linux 2 and configure it to run Swift. The approach taken here is a step by step approach through the console.</p>
        <a href="/documentation/server/guides/deploying/setting-up-amazon-ec2/" class="cta-secondary">Read this guide</a>
    </li>
    <li class="grid-level-1">
        <h3>Deploying to EC2</h3>
        <p>7. AWS Fargate Task Creation and Run</p>
        <a href="/documentation/server/guides/deploying/aws/" class="cta-secondary">Read this guide</a>
    </li>
    
    <li class="grid-level-1">
        <h3>Deploying to Lambda</h3>
        <p>This guide illustrates how to deploy a server-side Swift workload on AWS using the AWS Serverless Application Model (SAM) toolkit.</p>
        <a href="/documentation/server/guides/deploying/aws-sam-lambda" class="cta-secondary">AWS Lambda using the Serverless Application Model (SAM)</a>
    </li>
</ul>


## Deploying to other cloud providers

<ul class="grid-level-0">
    <li class="grid-level-1">
        <h3>DigitalOcean</h3>
        <p>This guide will walk you through setting up an Ubuntu virtual machine on a DigitalOcean Droplet. To follow this guide, you will need to have a DigitalOcean account with billing configured.</p>
        <a href="/documentation/server/guides/deploying/digital-ocean" class="cta-secondary">Deploying to DigitalOcean</a>
    </li>
    <li class="grid-level-1">
        <h3>Heroku</h3>
        <p>Heroku is a popular all-in-one hosting solution.</p>
        <a href="/documentation/server/guides/deploying/heroku" class="cta-secondary">Deploying to Heroku</a>
    </li>
    <li class="grid-level-1">
        <h3>Google Cloud Platform (GCP)</h3>
        <p>This guide describes how to build and run your Swift Server on serverless architecture with Google Cloud Build and Google Cloud Run. We’ll use Artifact Registry to store the Docker images.</p>
        <a href="/documentation/server/guides/deploying/gcp" class="cta-secondary">Deploying to Google Cloud Platform (GCP)</a>
    </li>
    <li class="grid-level-1">
        <h3>Ubuntu Linux</h3>
        <p>TODO: move this to the building section?</p>
        <a href="/documentation/server/guides/deploying/ubuntu" class="cta-secondary">Deploying to Ubuntu</a>
    </li>
</ul>

If you are deploying to your own servers (e.g. bare metal, VMs or Docker) there are several strategies for packaging Swift applications for deployment, see the packaging guides section for more information.

_Have a guide for other popular public clouds like Azure? Add it by submitting pull requests to the [Swift.org site](https://github.com/swiftlang/swift-org-website)_.

