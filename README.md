# Enterprise IAM Expansion: Secure LDAPS Federation (Keycloak & Active Directory)

## 📌 Project Overview
This project expands a local enterprise cybersecurity homelab (hosted on VirtualBox) by implementing a centralized Identity and Access Management (IAM) pipeline. I successfully federated a Windows Server 2025 Active Directory Domain Controller with a Keycloak Identity Provider (IdP) hosted on Ubuntu 22.04 via Docker. 

[Read the Full Incident Report on Medium](https://medium.com/@pompey.lamont01/defeating-javas-sslhandshakefailed-secure-ldaps-federation-with-keycloak-and-active-directory-217f2d5da1e0?postPublishedType=initial)

To ensure strict zero-trust principles, unencrypted LDAP (Port 389) was bypassed entirely in favor of strictly encrypted LDAPS (Port 636) using a custom Enterprise Root Certificate Authority.

## 🏗️ Architecture & Environment
*   **Hypervisor:** Oracle VirtualBox (Migrated from VMware Workstation to resolve virtual network instability)
*   **Identity Provider:** Keycloak (Docker Container) on Ubuntu 22.04 (`10.0.0.8`)
*   **Domain Controller:** Windows Server 2025 (`10.0.0.5`)
*   **Domain Name:** `corp.project-x-dc.com`
*   **Security Protocol:** LDAPS (TCP Port 636)

## 🚧 Key Challenges & Engineering Solutions

### 1. Active Directory Certificate Binding (Port 636 Connection Refused)
Active Directory does not natively serve LDAPS without a valid certificate. 
*   **Solution:** Installed the Active Directory Certificate Services (AD CS) role. Generated a "Domain Controller Authentication" template to enforce the *Server Authentication* purpose. Explicitly trusted the Enterprise Root CA in the Local Machine store and forcefully restarted the `NTDS` service to successfully bind the certificate to TCP Port 636.

### 2. Docker Truststore Injection
Keycloak (running on Java) inherently distrusts self-signed enterprise certificates, causing connection drops.
*   **Solution:** Extracted the AD CS certificate across the network using `openssl s_client`. Mounted the `ad-cert.pem` file directly into the Keycloak Docker container via `volumes` and explicitly defined the `--truststore-paths` flag in the compose command to force Java to respect the internal CA.

### 3. Defeating Java Strict Hostname Verification (`SSLHandshakeFailed`)
Even with a valid truststore, Keycloak threw an `SSLHandshakeFailed` error because the Keycloak UI was configured to reach `10.0.0.5`, which did not match the Subject Alternative Name (SAN) on the certificate (`WIN-06QSBAFSSS6.corp.project-x-dc.com`).
*   **Solution:** Modified the `docker-compose.yml` to include an `extra_hosts` DNS mapping. This allowed the Keycloak container to resolve the exact Windows Server hostname to the local `10.0.0.5` IP, achieving a perfect SNI match and successfully synchronizing the AD users into the Keycloak database.
