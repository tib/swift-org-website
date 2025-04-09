---
redirect_from: "server/guides/packaging"
layout: page
title: Packaging Applications using Docker
---

Once an application is built for production, it still needs to be packaged before it can be deployed to servers. There are several strategies for packaging Swift applications for deployment. One of the most popular ways to package applications these days is using container technologies such as [Docker](https://www.docker.com).

This guide explains how to package a server-side Swift application using Docker 
and how to communicate with a PostgreSQL database service - both running in separate Docker containers. The provided sample repository contains a simple Todo server implementation using the most popular server-side Swift web frameworks, Hummingbird and Vapor. The sample uses Docker compose for local orchestration and demonstrates secure SSL communication between the web service and the database. 

Packaging a Hummingbird and a Vapor application is very similar, however there are a few differences. In this guide we're going to explain how the Hummingbird sample is packages, but you should be able to follow along using the Vapor sample as well. Before you start, make sure that Docker is installed on your machine.


## Packaging using Docker

This guide demonstrates how to set up a PostgreSQL service with self-signed certificates using Docker, alongside a Hummingbird server running in a separate Docker container. The two services are configured to communicate over a TLS-encrypted channel.

### SSL Certificate generation

To establish a secure SSL connection between the Swift web service and PostgreSQL, SSL certificates are required. You can use the `generate.sh` script to quickly generate self-signed certificates for your services.

When deploying a server-side Swift application, securing the connection between the web server and a PostgreSQL database is critical. TLS (Transport Layer Security) provides encrypted communication, ensuring that data such as credentials and queries cannot be read or altered during transmission. It also supports authentication, allowing the server to verify the database’s identity and, optionally, the database to verify the client.

TLS is essential when services run on separate machines or across the internet. It guarantees confidentiality, integrity, and trust. Most PostgreSQL clients, including those used in Swift, support TLS and can be configured to use certificates for secure, verified connections.

First, the necessary files for establishing the TLS connection will be generated using the following script (`certificates/generate.sh`). Feel free to change the configuration based on your needs:

```sh
#!/bin/sh

# Generate CA (Certificate Authority) private key
openssl genpkey -algorithm RSA -out ca.key

# Generate CA certificate (PEM format)
openssl req -new -x509 -key ca.key -out ca.pem -days 365 -subj "/CN=PostgreSQL-CA"

# Generate server private key
openssl genpkey -algorithm RSA -out server.key

# Create a CSR with correct CN (Common Name) & SAN (Subject Alternative Name)
openssl req -new -key server.key -out server.csr -subj "/CN=localhost"

# SAN for localhost, db, and 127.0.0.1
echo "subjectAltName=DNS:localhost,DNS:db,IP:127.0.0.1" > san.cnf

# Sign the server certificate with CA & SAN
openssl x509 -req -in server.csr -CA ca.pem -CAkey ca.key -CAcreateserial -out server.pem -days 365 -extfile san.cnf
```

To run the certificate generator execute the following command:

```sh
# Generate certificates and private keys
cd hummingbird-docker/docker/certificates/
./generate.sh
```

With the certificates and private key files prepared, the next step is to configure PostgreSQL by enabling TLS support.

### Packaging the PostgreSQL database

This script (`db/config-ssl.sh`) generates the required SSL configuration file for PostgreSQL. It will be executed within the Docker container.

```sh
#!/bin/sh
set -e

# Append SSL configuration to postgresql.conf
echo "ssl = on" >> "$PGDATA/postgresql.conf"
echo "ssl_cert_file = '/certs/server.pem'" >> "$PGDATA/postgresql.conf"
echo "ssl_key_file = '/certs/server.key'" >> "$PGDATA/postgresql.conf"
echo "ssl_ca_file = '/certs/ca.pem'" >> "$PGDATA/postgresql.conf"

# Allow SSL connections only
echo "local all all trust" > "$PGDATA/pg_hba.conf"
echo "hostssl all all 0.0.0.0/0 md5" >> "$PGDATA/pg_hba.conf"
echo "hostssl all all ::/0 md5" >> "$PGDATA/pg_hba.conf"
```

To enable TLS support in PostgreSQL, a custom Docker image must be built. The following Dockerfile is based on the official PostgreSQL image, installs additional utilities, copies the generated SSL certificate files into the container, and executes the previously introduced configuration script.

```Dockerfile
# Use the official PostgreSQL image (using the "latest" tag)
FROM postgres:latest

# Update package lists, install netcat-openbsd (useful for waiting on network connections), and clean up apt cache
#RUN apt-get update && apt-get install -y netcat-openbsd && rm -rf /var/lib/apt/lists/*

# Set the working directory
WORKDIR /var/lib/postgresql

# Copy SSL certificate files into the /certs/ directory in the container
COPY docker/certificates/server.pem docker/certificates/server.key docker/certificates/ca.pem /certs/
# Set ownership to the postgres user and restrict permissions on the private key for security
RUN chown postgres:postgres /certs/* && chmod 600 /certs/server.key

# Copy an initialization script that sets up SSL into the Docker entrypoint directory
# Scripts in this directory are executed when the container is initialized
COPY docker/db/config-ssl.sh /docker-entrypoint-initdb.d/
RUN chmod +x /docker-entrypoint-initdb.d/config-ssl.sh

# Expose the PostgreSQL default port to allow external connections
EXPOSE 5432
```

To build the custom PostgreSQL Docker image with TLS support, the docker build command is used with the `-f` flag to specify the location of the Dockerfile. The `-t` flag assigns a name and version tag to the resulting image. This example builds the image from `docker/db/Dockerfile` and tags it as `my-tls-db:1.0.0`:

```sh
# build the docker image and use the name:tag format to tag it
docker build -f docker/db/Dockerfile . -t my-tls-db:1.0.0
```

Since the server must communicate with the database, both containers need to be part of the same Docker network. Docker networks can be managed using the following commands. To begin, create a custom network:

```sh
# create a custom network
docker network create my-network

# list available networks
docker network ls

# remove the network if no longer needed
# docker network rm my-network 
```

Once the image is built, the PostgreSQL container can be started using the docker run command. The `--name` flag assigns the container the name db, and `--network` connects it to a custom Docker network (`my-network`) for service discovery. 

Environment variables are set to configure the default user, password, and database. The `-v` flag mounts a local volume (`./pg_data`) to persist database files across restarts, while port `5432` is exposed for external access. The `--rm` flag ensures the container is automatically removed when stopped.

```sh
# run the docker image using env variables, a local pg_data volume and port exposed
docker run --rm --name db --network my-network \
-e "POSTGRES_USER=postgres" \
-e "POSTGRES_PASSWORD=postgres" \
-e "POSTGRES_DB=postgres" \
-v ./pg_data:/var/lib/postgresql/data \
-p 5432:5432 \
my-tls-db:1.0.0
```

With the database service running, the next step is to package the Swift backend application.

### Packaging the Hummingbird server

This multi-stage Dockerfile builds the Swift web service and produces a minimal final image optimized for deployment.

The first stage uses the official Swift image to compile the application in release mode with static linking of the standard library. Once the build is complete, the resulting artifacts are moved to a staging directory for easier access in the final stage.

The second stage is based on a popular Ubuntu distribution. It updates system packages, copies the build artifacts from the staging area, and configures the container for execution. Backtrace support is enabled to aid debugging, and the application is set to run under a non-root user for improved security.

```Dockerfile
# Build stage using a Swift image with version 6.0.3-noble
FROM swift:6.1-noble AS build

# Set the working directory for building the Swift application
WORKDIR /build

# Copy package manifest files to leverage Docker cache during dependency resolution
COPY ./Package.* ./

# Resolve and download package dependencies
RUN swift package resolve

# Copy the rest of the Swift project source files
COPY ./ .

# Build the Swift application in release mode with a static Swift standard library
RUN swift build -c release --static-swift-stdlib 

# Switch working directory to a staging area for assembling the final artifacts
WORKDIR /staging

# Copy the compiled executable from the build output into the staging directory
RUN cp "$(swift build --package-path /build -c release --show-bin-path)/Server" ./

# Copy the swift-backtrace-static tool, used for improved backtraces in Swift binaries
RUN cp "/usr/libexec/swift/linux/swift-backtrace-static" ./

# Find and copy all resource directories from the build output into the staging directory
RUN find -L "$(swift build --package-path /build -c release --show-bin-path)/" -regex '.*\.resources$' -exec cp -Ra {} ./ \;

# ---------------------------------------------------------------------------
# Final stage based on an Ubuntu image with the 'noble' tag
# ---------------------------------------------------------------------------
FROM ubuntu:noble

# Update package lists, upgrade packages, and clean up to reduce image size
RUN export DEBIAN_FRONTEND=noninteractive DEBCONF_NONINTERACTIVE_SEEN=true \
    && apt-get -q update \
    && apt-get -q dist-upgrade -y \
    && rm -r /var/lib/apt/lists/*

# Create a dedicated system user 'hummingbird' with its home directory at /app
RUN useradd --user-group --create-home --system --skel /dev/null --home-dir /app hummingbird

# Set the working directory to the application directory
WORKDIR /app

# Copy the staging files from the build stage into /app and set proper ownership
COPY --from=build --chown=hummingbird:hummingbird /staging /app

# Copy additional SSL certificate files needed for the application
COPY docker/certificates/ca.pem /app/

# Set an environment variable to configure Swift backtraces for debugging
ENV SWIFT_BACKTRACE=enable=yes,sanitize=yes,threads=all,images=all,interactive=no,swift-backtrace=./swift-backtrace-static

# Switch to the non-root 'hummingbird' user for security
USER hummingbird:hummingbird

# Expose the port on which the application will listen
EXPOSE 8080

# Define the entrypoint to run the 'Todos' executable and default command-line arguments
ENTRYPOINT ["/app/Server"]
CMD ["--hostname", "0.0.0.0", "--port", "8080"]
```

To create a local Docker image from a Dockerfile, use the docker build command from the application’s source directory. In the example below, a custom Dockerfile located at `docker/server/Dockerfile` is used to build the image and tag it as `my-server:1.0.0`:

```sh
docker build -f docker/server/Dockerfile . -t my-server:1.0.0
```

Once the image is built, it can be tested by running a container with the `docker run` command. This example starts the container with the name server, connects it to the `my-network` Docker network, sets the necessary environment variables for database communication, and exposes port `8080` to the host machine:

```sh
docker run --rm --name server --network my-network \
-e "DATABASE_HOST=db" \
-e "DATABASE_PORT=5432" \
-e "DATABASE_USER=postgres" \
-e "DATABASE_PASSWORD=postgres" \
-e "DATABASE_NAME=postgres" \
-e "ROOT_CERT_PATH=ca.pem" \
-p 8080:8080 \
my-server:1.0.0
```

> NOTE: The root certificate path is relative to the app/ directory, as the certificates are copied into this location during the image build process.

After starting the server, navigating to [http://localhost:8080](http://localhost:8080) in a browser should display the `Hello, world!` message. To test the API, a todo item can be created using the following cURL command:

```sh
curl -X POST http://localhost:8080/todos \
-H "Content-Type: application/json" \
-d '{"title": "Buy milk"}'
```

To retrieve the list of available todos, open [http://localhost:8080/todos](http://localhost:8080/todos) in a browser or use a similar HTTP client.

The next step is to orchestrate the database and server services together using Docker Compose.


### Docker compose

Begin by creating a `docker-compose.yaml` file with the following content:

```yml
services:
  db:
    build:
      context: .
      dockerfile: docker/db/Dockerfile
    volumes:
      - postgres:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    restart: always
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: postgres
    healthcheck:
      test:
        - CMD-SHELL
        - >
          sh -c "pg_isready -d '
            sslmode=verify-full
            sslrootcert=/var/lib/postgresql/ca.crt
            sslcert=/var/lib/postgresql/server.crt
            sslkey=/var/lib/postgresql/server.key
            user=$${POSTGRES_USER}
            password=$${POSTGRES_PASSWORD}
            dbname=$${POSTGRES_DB}
          '"
#      test: ["CMD-SHELL", "nc -z localhost 5432"]
      interval: 1s
      timeout: 5s
      retries: 10

  server:
    build:
      context: .
      dockerfile: docker/server/Dockerfile
    ports:
      - "8080:8080"
    depends_on:
      db:
        condition: service_healthy
    environment:
      DATABASE_HOST: db
      DATABASE_PORT: 5432
      DATABASE_USER: postgres
      DATABASE_PASSWORD: postgres
      DATABASE_NAME: postgres
      ROOT_CERT_PATH: ca.pem
    command: ["--hostname", "0.0.0.0", "--port", "8080"]

volumes:
    postgres:
```

The `db` service defines a PostgreSQL database container built from a custom Dockerfile located at `docker/db/Dockerfile`. The context: `.` setting specifies the root directory as the build context. A named volume (postgres) is mounted to `/var/lib/postgresql/data` to persist database data across container restarts. The container maps port `5432` on the host to port `5432` in the container, making the database accessible to external tools or services.

Environment variables configure the PostgreSQL instance with a default user, password, and database. The `restart: always` policy ensures the service is restarted automatically if it stops unexpectedly. A `healthcheck` is configured to verify the database’s readiness using `pg_isready` with TLS options. This health check ensures the database is fully initialized and accepting secure connections before other services attempt to connect. The health check runs every second with a timeout of 5 seconds and will retry up to 10 times.

The `volumes` section defines a named volume called `postgres`, used by the `db` service. This volume ensures persistent storage of database data, independent of the container lifecycle. When the container is removed or restarted, the volume retains all PostgreSQL data, allowing seamless recovery without data loss.

To start the database service, use the `docker compose up db` command. To reset the whole database service using a fresh configuration and volume, use docker compose down, followed by docker compose up and the `--build` option to rebuild the database image and recreate all the resources.

```sh
# run the database service
docker compose up db

# reset the whole service, rebuild everything and re-configure
docker compose down --volumes && docker compose up db --build db
```

The `server` service builds the Swift web server from a Dockerfile located at `docker/server/Dockerfile`. It also uses the current directory (`.`) as the build context. Port `8080` is exposed on both the host and container, allowing external access to the HTTP server. The `depends_on` directive ensures the server will only start once the `db` service passes its health check, providing a reliable startup sequence.

Environment variables configure the database connection, specifying the host (`db`), port, credentials, and the name of the root certificate used for TLS validation. The command field overrides the container’s default entrypoint to explicitly set the server hostname and port. This setup ensures the server is bound to all interfaces (`0.0.0.0`) and listens on port `8080`, making it accessible from outside the container.

To start the server service, use the `docker compose up server` command.

To start all services at once, use `docker compose up`. To build all images without starting the services, run `docker compose build`.

## Summary

At this point, the application's Docker image is ready to be deployed to the server hosts (which need to run docker), or to one of the platforms that supports Docker deployments.

See [Docker's documentation](https://docs.docker.com/engine/reference/commandline/) for more complete information about Docker.














