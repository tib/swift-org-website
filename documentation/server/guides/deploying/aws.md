---
redirect_from: "server/guides/deploying/aws"
layout: page
title: Deploying to AWS on Amazon Linux 2
---


## Compile on instance

There are two alternative ways to compile code on the instance, either by:

- [downloading and using the toolchain directly on the instance](#compile-using-a-downloaded-toolchain),
- or by [using docker, and compiling inside a docker container](#compile-with-docker)

### Compile using a downloaded toolchain

Run the following command in the SSH terminal. Note that there may be a more up to date version of the swift toolchain. Check [https://swift.org/download/#releases](/download/#releasess) for the latest available toolchain url for Amazon Linux 2.

```
SwiftToolchainUrl="https://swift.org/builds/swift-5.4.1-release/amazonlinux2/swift-5.4.1-RELEASE/swift-5.4.1-RELEASE-amazonlinux2.tar.gz"
sudo yum install ruby binutils gcc git glibc-static gzip libbsd libcurl libedit libicu libsqlite libstdc++-static libuuid libxml2 tar tzdata ruby -y
cd $(mktemp -d)
wget ${SwiftToolchainUrl} -O swift.tar.gz
gunzip < swift.tar.gz | sudo tar -C / -xv --strip-components 1
```

Finally, check that Swift is correctly installed by running the Swift REPL: `swift`.

![Invoke REPL](/assets/images/server-guides/aws/repl.png)

Let's now download and build an test application. We will use the `--static-swift-stdlib` option so that it can be deployed to a different server without the Swift toolchain installed. These examples will deploy SwiftNIO's [example HTTP server](https://github.com/apple/swift-nio/tree/master/Sources/NIOHTTP1Server), but you can test with your own project.

```
git clone https://github.com/apple/swift-nio.git
cd swift-nio
swift build -v --static-swift-stdlib -c release
```

## Compile with Docker

Ensure that Docker and git are installed on the instance:

```
sudo yum install docker git
sudo usermod -a -G docker ec2-user
sudo systemctl start docker
```

You may have to log out and log back in to be able to use Docker. Check by running `docker ps`, and ensure that it runs without errors.

Download and compile SwiftNIO's [example HTTP server](https://github.com/apple/swift-nio/tree/master/Sources/NIOHTTP1Server):

```
docker run --rm  -v "$PWD:/workspace"  -w /workspace swift:5.4-amazonlinux2   /bin/bash -cl ' \
     swift build -v --static-swift-stdlib -c release
```
## Test binary
Using the same steps as above, launch a second instance (but don't run any of the bash commands above!). Be sure to use the same SSH keypair.

From within the AWS management console, navigate to the EC2 service and find the instance that you just launched. Click on the instance to see the details, and find the internal IP. In my example, the internal IP is `172.31.3.29`

From the original build instance, copy the binary to the new server instance:
```scp .build/release/NIOHTTP1Server ec2-user@172.31.3.29```

Now connect to the new instance:
```ssh ec2-user@172.31.3.29```

From within the new instance, test the Swift binary:
```
NIOHTTP1Server localhost 8080 &
curl localhost:8080
```

From here, options are endless and will depend on your application of Swift. If you wish to run a web service be sure to open the Security Group to the correct port and from the correct source. When you are done testing Swift, shut down the instance to avoid paying for unneeded compute. From the EC2 dashboard, select both instances, select "Actions" from the menu, then select "Instance state" and then finally "terminate".

![Terminate Instance](/assets/images/server-guides/aws/terminate.png)
