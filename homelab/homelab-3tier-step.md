# Building an Enterprise-Grade 3-Tier Public Key Infrastructure for Your Homelab

*A comprehensive guide to constructing an offline-rooted, hardware-secured, containerized Certificate Authority stack on Ubuntu using step-ca and PostgreSQL.*

---

## Executive Summary & Architecture Overview

In this guide, we construct an enterprise-class **3-Tier Public Key Infrastructure (PKI)** for a local network and homelab environment [31, 113]. Modeled after production identity architectures, this design decouples root trust from operational certificate issuance through an intermediate policy layer, maximizing defense-in-depth:

1. **Tier 1 (Root CA):** Offline & isolated. Private key protected by a **YubiHSM 2** hardware security module.
2. **Tier 2 (Policy CA):** Semi-offline policy tier. Private key protected by hardware token (**YubiKey 5 PIV**) or host-bound **TPM 2.0** [10, 33, 46].
3. **Tier 3 (Issuing CA):** Online, automated microservice running in **Docker** containers [158]. Private key protected via encrypted **text/password file**, backed by a high-performance **PostgreSQL** database [153].

This setup provides an internal **ACME server** (identical to the protocol used by Let's Encrypt) capable of automatically issuing and renewing short-lived TLS certificates (e.g., 24-hour duration) for homelab workloads, microservices, and network endpoints [31, 137].

---

## Why Build a 3-Tier PKI in Your Homelab?

* **Defense-in-Depth:** In a traditional 2-tier PKI, compromising the intermediate CA forces a complete re-bootstrapping of all client trust stores [101, 113]. A 3-tier architecture adds an intervening **Policy Tier**, allowing you to revoke a compromised Issuing CA and spin up a new one without touching the Root CA or re-distributing trust anchors across your fleet [113].
* **Hardware-Rooted Assurance:** Storing CA private keys on hardware security modules (YubiHSM, YubiKey, TPM 2.0) ensures private keys can never be extracted in plaintext, even if the underlying OS is compromised [10, 31, 112].
* **Elimination of Self-Signed Certificates:** Replaces ad-hoc self-signed certs with centralized cryptographic identity recognized across all homelab devices [32].
* **Passive Revocation via Automated ACME:** Issuing short-lived certificates (24-hour lifetime) eliminates reliance on complex Certificate Revocation Lists (CRLs) or fragile OCSP responders [135, 137].

---

## 3-Tier Architecture & Hardware Specifications

| PKI Tier | Role & Operational Status | Cryptographic Key Protection | Hosting Model |
| :--- | :--- | :--- | :--- |
| **Tier 1: Root CA** | Trust Anchor (Offline / Air-Gapped) | **YubiHSM 2** (PKCS#11) | Dedicated Air-Gapped Host / Live Boot |
| **Tier 2: Policy CA** | Intermediate Policy Issuer (Semi-Offline) | **YubiKey 5 NFC (PIV)** or **TPM 2.0** | Admin Workstation / Air-Gapped System |
| **Tier 3: Issuing CA** | Online ACME & TLS Issuer (24/7) | Encrypted **Password / Text File** Secret | **Docker Container** on Ubuntu Server |

### Infrastructure Stack Summary
* **Host Platform:** Ubuntu Server 24.04 LTS (x86_64 or ARM64) [33]
* **Database Backend:** PostgreSQL 16 (Containerized microservice) [153]
* **CA Software:** Smallstep `step-ca` container & `step` CLI [33, 37]
* **Network Protocol:** Automated Certificate Management Environment (ACME v2) [31, 127]

---

## Part 1: System & Base Environment Setup

### 1.1 Install Prerequisites on Ubuntu Host
Log into your Ubuntu Server host and update system dependencies, Docker, and hardware utilities [33, 35]:

```bash
# Update base system
sudo apt update && sudo apt upgrade -y

# Install prerequisite tools and hardware drivers
sudo apt install -y \
    curl \
    jq \
    pcscd \
    yubikey-manager \
    tpm2-tools \
    libpcsclite-dev \
    gnupg2 \
    docker.io \
    docker-compose-v2

# Configure Docker permissions
sudo usermod -aG docker $USER
```

### 1.2 Prepare Project Directory Structure
Create a clean directory layout for persistent storage, configuration files, and database mounts:

```bash
mkdir -p ~/homelab-pki/{config,certs,secrets,postgres_data}
cd ~/homelab-pki
```

---

## Part 2: Tier 1 — Bootstrapping the Offline Root CA on YubiHSM

The **Root CA** represents the apex of trust in your hierarchy [101, 112]. Its private key is generated and locked inside a **YubiHSM 2**.

```
 +-------------------------------------------------------+
 |                 TIER 1: ROOT CA                       |
 |  - Subject: "Homelab Root CA"                         |
 |  - Status: Offline / Air-Gapped                       |
 |  - Key Protection: YubiHSM 2 (PKCS#11)                |
 +-------------------------------------------------------+
                            |
                            | Signs Policy CA Cert
                            v
```

### 2.1 Configure YubiHSM 2 PKCS#11 Middleware
Install the YubiHSM PKCS#11 connector and toolchain on your air-gapped setup [10]:

```bash
sudo tee /etc/yubihsm_pkcs11.conf > /dev/null << 'EOF'
connector = http://localhost:12345
debug
EOF
```

### 2.2 Initialize Root CA Certificate with YubiHSM
Create an air-gapped cold storage mount (e.g. encrypted USB drive `/mnt/cold-storage`) to keep the root initialization output and encrypted backups [42, 45]:

```bash
export STEPPATH=/mnt/cold-storage/root-ca

# Initialize PKI structures with step CLI
step ca init --pki \
    --name="Homelab Root CA" \
    --dns="root.ca.internal" \
    --address=":443" \
    --provisioner="root-admin@homelab.internal" \
    --deployment-type=standalone

# Generate Root CA key pair directly within YubiHSM slot via PKCS#11
yubihsm-shell -a generate-key \
    --object-id=1 \
    --label="Homelab Root CA Key" \
    --algorithm=ecdsa-p256 \
    --capabilities=sign-attestation,sign-ecdsa
```

*Note: The Root CA certificate (`root_ca.crt`) will be copied to all homelab client trust stores, while the root private key remains locked within the YubiHSM [46, 101].*

---

## Part 3: Tier 2 — Generating the Policy Tier CA (YubiKey / TPM Protected)

The **Policy CA** sits between the Root CA and operational Issuing CAs [113]. It defines organizational policy constraints (path lengths, allowed domain name spaces) and issues certificates to online Issuing CAs.

```
 +-------------------------------------------------------+
 |                 TIER 2: POLICY CA                     |
 |  - Subject: "Homelab Intermediate Policy CA"          |
 |  - Status: Semi-Offline / Administrative              |
 |  - Key Protection: YubiKey 5 (PIV) OR TPM 2.0        |
 +-------------------------------------------------------+
                            |
                            | Signs Issuing CA Cert
                            v
```

### 3.1 Option A: Hardware Protection via YubiKey 5 (PIV Slot 9c)
Insert a YubiKey 5 into your administrative machine to store the Policy CA signing key [33, 46]:

```bash
# Enable PCSC daemon
sudo systemctl enable --now pcscd

# Import Policy CA certificate into YubiKey PIV Slot 9c (Digital Signature)
ykman piv certificates import 9c policy_ca.crt
```

### 3.2 Option B: Hardware Protection via TPM 2.0
If using host-bound hardware protection via an onboard TPM 2.0 module on Ubuntu [10, 18]:

```bash
# Store Policy CA signing credentials securely using TPM 2.0 and systemd-creds
systemd-creds setup
```

---

## Part 4: Tier 3 — Deploying the Containerized Issuing CA & PostgreSQL Stack

The **Issuing CA** is the online tier that processes ACME requests and issues short-lived TLS certificates for services [113, 213]. Its private key is encrypted with password protection and stored in a secure secret mount within the container stack [18, 48].

```
 +-------------------------------------------------------+
 |                TIER 3: ISSUING CA                     |
 |  - Subject: "Homelab Issuing CA 1"                    |
 |  - Status: Online (24/7 Containerized Service)        |
 |  - Key Protection: Text / Password File               |
 |  - Persistence: PostgreSQL 16 Database Backend        |
 +-------------------------------------------------------+
                            |
                            | Issues Short-Lived Leaf Certs (24h)
                            v
                 +----------------------+
                 | Homelab Services &   |
                 | Microservices (ACME) |
                 +----------------------+
```

### 4.1 Generate Password-Protected Issuing CA Key & Certificate
Sign the Issuing CA certificate using the Tier 2 Policy CA [45, 48]:

```bash
# Create Issuing CA Keypair signed by Policy CA with password protection
step certificate create "Homelab Operational Issuing CA" \
    ~/homelab-pki/certs/issuing_ca.crt \
    ~/homelab-pki/secrets/issuing_ca_key \
    --profile=intermediate-ca \
    --ca=policy_ca.crt \
    --ca-key="yubikey:slot-id=9c" \
    --password-file=~/homelab-pki/secrets/issuing_ca_password.txt
```

### 4.2 Configure `step-ca` (`ca.json`) for PostgreSQL Integration
Edit `~/homelab-pki/config/ca.json` to define certificate lifetime limits, encryption keys, and the PostgreSQL database backend [50, 153]:

```json
{
  "root": "/home/ubuntu/homelab-pki/certs/root_ca.crt",
  "federatedRoots": [],
  "crt": "/home/ubuntu/homelab-pki/certs/issuing_ca.crt",
  "key": "/home/ubuntu/homelab-pki/secrets/issuing_ca_key",
  "keyPassword": "/home/ubuntu/homelab-pki/secrets/issuing_ca_password.txt",
  "address": ":9000",
  "dnsNames": ["ca.homelab.internal", "10.0.0.10"],
  "db": {
    "type": "postgresql",
    "dataSource": "host=postgres port=5432 user=stepca password=<DB_PASSWORD> dbname=stepcadb sslmode=disable"
  },
  "authority": {
    "provisioners": []
  },
  "tls": {
    "cipherSuites": [
      "TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256",
      "TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384",
      "TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256"
    ],
    "minVersion": 1.2,
    "maxVersion": 1.3
  }
}
```

### 4.3 Create Microservices Stack with Docker Compose
Define the containerized infrastructure in `~/homelab-pki/docker-compose.yml` [153, 158]:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    container_name: pki-postgres
    restart: always
    environment:
      POSTGRES_USER: stepca
      POSTGRES_PASSWORD: <DB_PASSWORD>
      POSTGRES_DB: stepcadb
    volumes:
      - ./postgres_data:/var/lib/postgresql/data
    networks:
      - pki-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U stepca -d stepcadb"]
      interval: 5s
      timeout: 5s
      retries: 5

  step-ca:
    image: smallstep/step-ca:latest
    container_name: pki-issuing-ca
    restart: always
    depends_on:
      postgres:
        condition: service_healthy
    ports:
      - "443:9000"
    volumes:
      - ./config:/home/step/config:ro
      - ./certs:/home/step/certs:ro
      - ./secrets:/home/step/secrets:ro
    environment:
      - DOCKER_STEPCA_INIT_PASSWORD_FILE=/home/step/secrets/issuing_ca_password.txt
    command: ["/usr/local/bin/step-ca", "/home/step/config/ca.json"]
    networks:
      - pki-network

networks:
  pki-network:
    driver: bridge
```

### 4.4 Start the Microservices Stack
Launch the containers in detached mode [153]:

```bash
docker compose up -d
docker compose logs -f step-ca
```

---

## Part 5: Configuring ACME Provisioner & Issuing Certificates

With the 3-tier CA online, enable the **ACME provisioner** to transform your Issuing CA into a private Let's Encrypt endpoint [31, 53].

### 5.1 Add ACME Provisioner
Execute the provisioner addition within the running container or via `step` CLI [53]:

```bash
docker exec -it pki-issuing-ca step ca provisioner add acme --type=acme
docker compose restart step-ca
```

### 5.2 Bootstrap Clients & Install Root Certificate
On any homelab endpoint or VM, download and install the Root CA certificate into the system trust store [51, 57]:

```bash
# Bootstrap local step environment
step ca bootstrap \
    --ca-url https://ca.homelab.internal \
    --fingerprint <ROOT_CA_FINGERPRINT> \
    --install
```

### 5.3 Issue Automated Certificates via Certbot
Use standard ACME clients to request short-lived (24h) TLS certificates [58, 127]:

```bash
certbot certonly --standalone \
    --server https://ca.homelab.internal/acme/acme/directory \
    -d dashboard.homelab.internal \
    --agree-tos
```

---

## Operational Maintenance & Security Checklist

1. **Root CA Key Protection:** Ensure the YubiHSM 2 remains disconnected from power/USB except during scheduled Intermediate Policy CA renewals or emergency revocation routines [112].
2. **PostgreSQL Database Backups:** Schedule automated database snapshots for audit trail preservation and transaction logging [153]:
   ```bash
   docker exec pki-postgres pg_dump -U stepca stepcadb > ~/homelab-pki/backups/pki_db.sql
   ```
3. **Passive Revocation Routine:** Clients must execute renewal scripts every 12 to 16 hours. Certificates automatically expire after 24 hours, mitigating key compromise window risks without CRL/OCSP overhead [44, 137].

---

## Summary & Further Reading

You have successfully constructed a production-grade, 3-tier Public Key Infrastructure:
* **Root CA** locked in hardware on a **YubiHSM** [31, 112].
* **Policy CA** secured via **YubiKey 5 / TPM 2.0** [10, 33, 46].
* **Issuing CA** operating as an automated **Docker microservice** backed by **PostgreSQL** [153, 158].

### Related Documents in Notebook:
* *Build a Tiny Certificate Authority For Your Homelab*
* *step-ca Certificate Authority Overview*
* *Everything You Should Know About Certificates and PKI*
