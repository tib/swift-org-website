---
layout: page
title: Securing RDS & using a VPN
---

This guide explains how to create an AWS VPN Client Endpoint by generating the necessary certificate files and certificates, importing them into AWS Certificate Manager (ACM), configuring a security group, and setting up the client VPN endpoint. Follow these steps to establish secure VPN access to your VPC.

## Prerequisites

- **OpenSSL:** Ensure the `openssl` command is available. If not, install it and adjust the configuration files as needed.
- **AWS Account:** Access to AWS Console with permissions to manage ACM, VPC, and VPN configurations.
- **Basic Networking Knowledge:** Familiarity with CIDR notation and VPC subnet configuration.
- A VPN client, e.g.: OpenVPN:

```yaml
brew install --cask openvpn-connect
```

---

## Configuration Files

Before generating any certificates, you must create the following configuration files. Each file includes specific settings for the certificate it will be used to generate.

### ca.cnf

This configuration file is used to create the Certificate Authority (CA) certificate. It sets default key size, message digest, and distinguished name fields. The `v3_ca` section defines the CA-specific extensions including setting `basicConstraints` to `CA:true`.

```
[ req ]
default_bits       = 2048
default_md         = sha256
default_keyfile    = ca.key
distinguished_name = req_distinguished_name
x509_extensions    = v3_ca
prompt             = no

[ req_distinguished_name ]
C  = US
CN = vpn.domain.com

[ v3_ca ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = CA:true
```

### client.cnf

This configuration file is for generating the client certificate used in VPN authentication. It specifies certificate properties such as key usage (digitalSignature, keyEncipherment), extended key usage for client authentication (`clientAuth`), and includes a subject alternative name (SAN).

```
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn
req_extensions = req_ext

[ dn ]
CN = vpn.domain.com

[ req_distinguished_name ]
C = US
CN = vpn.domain.com

[ req_ext ]
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = vpn.domain.com
```

### server.cnf

This configuration file is used to generate the server certificate for VPN authentication. It is similar to the client configuration but specifies the extended key usage for server authentication (`serverAuth`). It also includes the appropriate SAN entry.

```
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn
req_extensions = req_ext

[ dn ]
CN = vpn.domain.com

[ req_distinguished_name ]
C = US
CN = vpn.domain.com

[ req_ext ]
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = vpn.domain.com
```


## Generating Certificates and Keys

After creating the configuration files, generate the CA, server, and client certificates using OpenSSL.

### CA Certificate

Generate the Certificate Authority (CA) private key and self-signed CA certificate:

```bash
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt -config ca.cnf
```

### Server Certificate

Generate the server key, certificate signing request (CSR), and sign the certificate with the CA:

```bash
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr -config server.cnf
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365 -sha256 -extfile server.cnf -extensions req_ext
```

### Client Certificate

Generate the client key, CSR, and sign the certificate with the CA:

```bash
openssl genrsa -out client.key 2048
openssl req -new -key client.key -out client.csr -config client.cnf
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days 365 -sha256 -extfile client.cnf -extensions req_ext
```

## Import Certificates into AWS Certificate Manager

### Server Certificate Import

1. Open the **AWS Certificate Manager (ACM)** in the AWS Console.
2. Navigate to **List certificates** and choose the **Import certificate** option.
3. For the **Server Certificate** import:
    - **Certificate body:** Paste the contents of `server.crt`
    - **Certificate private key:** Paste the contents of `server.key`
    - **Certificate chain:** Paste the contents of `ca.crt`
4. Add a custom tag to identify this as the *server certificate*.

### Client Certificate Import

1. In ACM, use the **Import certificate** option again for the **Client Certificate**.
2. For the client import:
    - **Certificate body:** Paste the contents of `client.crt`
    - **Certificate private key:** Paste the contents of `client.key`
    - **Certificate chain:** Paste the same `ca.crt` used for the server certificate.
3. Add a custom tag to identify this as the *client certificate*.


## Create a Security Group for VPN Connection

1. Open the **VPC Dashboard** and navigate to **Security** > **Security Groups**.
2. Click the **Create security group** button.
3. Provide a custom name (e.g., `vpn-access`).
4. Configure the following rules:
    - **Outbound Rule:** For simplicity, set the rule type to "All traffic" with a destination of `0.0.0.0/0` to allow access to any destination within your VPC.
5. Click **Create security group** to finalize the setup.


## Create Client VPN Endpoint

1. Open the **VPC Dashboard** and navigate to **Virtual Private Network (VPN)** > **Client VPN Endpoints**.
2. Click **Create client VPN endpoint**.
3. Configure the endpoint with the following settings:
    - **Name tag:** Set a custom name for the VPN client configuration.
    - **Client IPv4 CIDR:** Choose a subnet that does not overlap with your VPC’s IPv4 CIDR (e.g., if your VPC is `172.31.0.0/16`, a CIDR like `10.1.0.0/16` is acceptable).
    - **Server certificate ARN:** Select the server certificate imported earlier.
    - **Authentication options:** Choose mutual authentication and select the client certificate ARN imported previously.
    - **Other parameters:** Set the DNS server IP (typically your VPC subnet start address + 2, e.g., for `172.31.0.0/16`, use `172.31.0.2`).
    - **Split-tunnel:** Enable this option to allow split traffic between AWS and the internet.
    - **VPC Selection:** Choose the appropriate VPC.
    - **Security Group:** Associate the security group you created (e.g., `vpn-access`).
4. Click **Create client VPN endpoint**.


## Configure VPN Endpoint Association and Authorization

1. Once the VPN endpoint is created, it will initially be in a **pending** state.
2. **Associate Target Network:**
    - Go to the endpoint list and open your newly created endpoint.
    - Under **Target network associations**, click **Associate target network**.
    - Select the desired VPC and the subnet where your target resources (e.g., EC2 instances) reside.
    - Click **Associate target network**.
3. **Set Authorization Rules:**
    - Under **Authorization rules**, click **Add authorization rule**.
    - Configure the rule:
        - **Destination network:** Enter your VPC subnet (e.g., `172.31.0.0/16`).
        - **Access:** For simplicity, choose "Allow access to all users".
4. Wait approximately 5-10 minutes for the network association, route creation, and authorization rule updates. The VPN status should eventually change to **Available**.
5. Once available, click **Download client configuration** to retrieve the configuration file for your OpenVPN client.


## Configure OpenVPN Client on Your PC

1. **Install OpenVPN Client:**
    - Download and install the OpenVPN client appropriate for your operating system.
2. **Modify the Client Configuration File:**
    - Open the downloaded configuration file with a text editor.
    - Immediately after the `<ca>...</ca>` section, add two new sections:
        - `<cert>...</cert>`: Paste the contents of the generated `client.crt` file.
        - `<key>...</key>`: Paste the contents of the generated `client.key` file.
3. **Import and Connect:**
    - Import the modified configuration file into your OpenVPN client.
    - Use the client to connect to the VPN. Once connected, you will have access to the resources within your VPC.


```yaml
client
dev tun
proto udp
remote cvpn-endpoint-06bc6ea32e32738b9.prod.clientvpn.eu-central-1.amazonaws.com 443
remote-random-hostname
resolv-retry infinite
nobind
remote-cert-tls server
cipher AES-256-GCM
verb 3
<ca>
-----BEGIN CERTIFICATE-----
PASTE_CA_HERE
-----END CERTIFICATE-----

</ca>
<cert>
-----BEGIN CERTIFICATE-----
PASTE_CERT_HERE
-----END CERTIFICATE-----
</cert>
<key>
-----BEGIN PRIVATE KEY-----
PASTE_PRIVATE_KEY_HERE
-----END PRIVATE KEY-----
</key>

reneg-sec 0

verify-x509-name vpn.domain.com name
```

This VPN connection is now configured and will serve as the secure pathway to access your PostgreSQL RDS instance.


# Accessing AWS RDS via VPN with a Private Address

In this section, we transition from using a publicly accessible Amazon RDS instance to configuring the RDS instance to be private. With the RDS instance set to "Not publicly accessible" and a VPN connection, your locally running Hummingbird server will only be able to connect when the VPN is active.

## Modifying the RDS Instance to Private

Before attempting to reconnect from your local environment, update the RDS instance settings in the AWS Console as follows:

1. **Navigate to the AWS Console:**
    - Go to **Aurora and RDS -> Databases** and select the database instance created in the RDS setup guide.
2. **Modify the Instance:**
    - Click on the **Modify** button.
    - Under the **Connectivity** section, open the **Additional Configuration**.
    - Select the **Not publicly accessible** option.
    - Click the **Continue** button.
    - Choose the **Apply Immediately** option.
    - Click on the **Modify DB Instance** button.
    - Wait around **5 minutes** for the changes to take effect.

## Testing the RDS Connection without VPN

With the RDS instance now private, try to run the Docker Compose configuration as before:

```bash
docker compose up
```

You should receive a connection error indicating that your local environment cannot reach the private RDS instance directly.

## Establishing the VPN Connection

To access the private RDS instance, establish a VPN connection:

1. **Connect to the VPN:**
    - Use the VPN client configured with the OpenVPN settings
2. **Re-run the Docker Compose Setup:**
    - Once the VPN connection is active, run the Docker Compose setup again:

```bash
docker compose up
```

This time, the connection to the RDS instance should succeed because the VPN provides secure, private network access to your AWS resources.

Of course if you host your application inside the Amazon network, e.g. an EC2 instance will be able to access the RDS service without a VPN service running. However, a local VPN connection can be extremely useful when debugging your application using an Amazon RDS database.

In the next guide we're going to show you how to move the local HTTP server to the cloud.



