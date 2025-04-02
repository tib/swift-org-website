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
        <p>1. This guide covers the process of creating a Docker image and using Docker Compose to host the necessary infrastructure on your local machine. You’ll learn how to use a pre-defined docker-compose file and Dockerfile, build the image locally.</p>
        <a href="/documentation/server/guides/packaging" class="cta-secondary">Packaging using Docker</a>
    </li>
    <li class="grid-level-1">
        <h3>Building for Linux</h3>
        <p>The Swift Static Linux SDK allows you to build your program as a fully statically linked executable, with no external dependencies at all (not even the C library), which means that it will run on any Linux distribution as the only thing it depends on is the Linux system call interface.</p>
        <a href="/documentation/articles/static-linux-getting-started" class="cta-secondary">Getting Started with the Static Linux SDK</a>
    </li>
</ul>


## Deploying to AWS

<ul class="grid-level-0">
    <li class="grid-level-1">
        <h3>Setting up RDS</h3>
        <p>Lorem ipsum dolor sit amet</p>
        <a href="" class="cta-secondary">Read this guide</a>
    </li>
    <li class="grid-level-1">
        <h3>Using RDS during development </h3>
        <p>2. + 3. Lorem ipsum dolor sit amet</p>
        
        <a href="" class="cta-secondary">Read this guide</a>
    </li>
    <li class="grid-level-1">
        <h3>Securing RDS & using a VPN (private access)</h3>
        <p>4. + 5. Lorem ipsum dolor sit amet</p>
        <a href="" class="cta-secondary">Read this guide</a>
    </li>    
    <li class="grid-level-1">
        <h3>6. 7. 8. Using ECR to push Docker images</h3>
        <p>Lorem ipsum dolor sit amet</p>
        <a href="" class="cta-secondary">Read this guide</a>
    </li>
    <li class="grid-level-1">
        <h3>9. Using Fargate as a service deploying from ECR</h3>
        <p>Lorem ipsum dolor sit amet</p>
        <a href="" class="cta-secondary">Read this guide</a>
    </li>
    <li class="grid-level-1">
        <h3>1. Setting up EC2</h3>
        <p>An AWS EC2 instance is a virtual computer in the cloud used to run apps, websites, or services. In this tutorial, you will learn how to set up an AWS EC2 instance, configure its security settings, create a key pair, and connect to it via SSH. This setup is essential for deploying and managing applications on AWS.</p>
        <a href="" class="cta-secondary">Read this guide</a>
    </li>
    <li class="grid-level-1">
        <h3>Deploying to EC2</h3>
        <p>7. AWS Fargate Task Creation and Run</p>
        <a href="/documentation/server/guides/deploying/aws" class="cta-secondary">Read this guide</a>
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
</ul>

If you are deploying to your own servers (e.g. bare metal, VMs or Docker) there are several strategies for packaging Swift applications for deployment, see the packaging guides section for more information.

_Have a guide for other popular public clouds like Azure? Add it by submitting pull requests to the [Swift.org site](https://github.com/swiftlang/swift-org-website)_.

