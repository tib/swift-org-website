---
redirect_from: "server/guides/packaging"
layout: page
title: Packaging Applications using Docker
---

Once an application is built for production, it still needs to be packaged before it can be deployed to servers. 

There are several strategies for packaging Swift applications for deployment. This guide explains how to package a server-side Swift application using Docker and how to communicate with a PostgreSQL database service - both running in separate Docker containers. 

The provided sample repository uses Docker Compose for local orchestration and demonstrates secure SSL communication between the web service and the database.

One of the most popular ways to package applications these days is using container technologies such as [Docker](https://www.docker.com).








Here is an example `Dockerfile` that builds and packages the application on top of CentOS:

```Dockerfile
#------- build -------
FROM swift:centos8 as builder

# set up the workspace
RUN mkdir /workspace
WORKDIR /workspace

# copy the source to the docker image
COPY . /workspace

RUN swift build -c release --static-swift-stdlib

#------- package -------
FROM centos
# copy executables
COPY --from=builder /workspace/.build/release/<executable-name> /

# set the entry point (application name)
CMD ["<executable-name>"]
```

To create a local Docker image from the `Dockerfile` use the `docker build` command from the application's source location, e.g.:

```bash
$ docker build . -t <my-app>:<my-app-version>
```

To test the local image use the `docker run` command, e.g.:

```bash
$ docker run <my-app>:<my-app-version>
```

Finally, use the `docker push` command to publish the application's Docker image to a Docker repository of your choice, e.g.:

```bash
$ docker tag <my-app>:<my-app-version> <docker-hub-user>/<my-app>:<my-app-version>
$ docker push <docker-hub-user>/<my-app>:<my-app-version>
```

At this point, the application's Docker image is ready to be deployed to the server hosts (which need to run docker), or to one of the platforms that supports Docker deployments.

See [Docker's documentation](https://docs.docker.com/engine/reference/commandline/) for more complete information about Docker.











## Archive (Tarball, ZIP file, etc.)

Since cross-compiling Swift for Linux is not (yet) supported on Mac or Windows, we need to use virtualization technologies like Docker to compile applications we are targeting to run on Linux.

That said, this does not mean we must also package the applications as Docker images in order to deploy them. While using Docker images for deployment is convenient and popular, an application can also be packaged using a simple and lightweight archive format like tarball or ZIP file, then uploaded to the server where it can be extracted and run.

Here is an example of using Docker and `tar` to build and package the application for deployment on Ubuntu servers:

First, use the `docker run` command from the application's source location to build it:

```bash
$ docker run --rm \
  -v "$PWD:/workspace" \
  -w /workspace \
  swift:bionic \
  /bin/bash -cl "swift build -c release --static-swift-stdlib"
```

Note we are bind mounting the source directory so that the build writes the build artifacts to the local drive from which we will package them later.

Next we can create a staging area with the application's executable:

```bash
$ docker run --rm \
  -v "$PWD:/workspace" \
  -w /workspace \
  swift:bionic  \
  /bin/bash -cl ' \
     rm -rf .build/install && mkdir -p .build/install && \
     cp -P .build/release/<executable-name> .build/install/'
```

Note this command could be combined with the build command above--we separated them to make the example more readable.

Finally, create a tarball from the staging directory:

```bash
$ tar cvzf <my-app>-<my-app-version>.tar.gz -C .build/install .
```

We can test the integrity of the tarball by extracting it to a directory and running the application in a Docker runtime container:

```bash
$ cd <extracted directory>
$ docker run -v "$PWD:/app" -w /app bionic ./<executable-name>
```

Deploying the application's tarball to the target server can be done using utilities like `scp`, or in a more sophisticated setup using configuration management system like `chef`, `puppet`, `ansible`, etc.


## Source Distribution

Another distribution technique popular with dynamic languages like Ruby or Javascript is distributing the source to the server, then compiling it on the server itself.

To build Swift applications directly on the server, the server must have the correct Swift toolchain installed. [Swift.org](/download/#linux) publishes toolchains for a variety of Linux distributions, make sure to use the one matching your server Linux version and desired Swift version.

The main advantage of this approach is that it is easy. Additional advantage is the server has the full toolchain (e.g. debugger) that can help troubleshoot issues "live" on the server.

The main disadvantage of this approach that the server has the full toolchain (e.g. compiler) which means a sophisticated attacker can potentially find ways to execute code. They can also potentially gain access to the source code which might be sensitive. If the application code needs to be cloned from a private or protected repository, the server needs access to credentials which adds additional attack surface area.

In most cases, source distribution is not advised due to these security concerns.


