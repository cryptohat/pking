# Building an Enterprise-Grade 3-Tier PKI with OmniTrust otPKI and ILM for Your Homelab

*A comprehensive guide to constructing an offline-rooted, hardware-secured, containerized Certificate Authority stack on Ubuntu using OmniTrust otPKI, OmniTrust Identity Lifecycle Management (ILM) as a Registration Authority (RA), CRL publishing, and PostgreSQL.*

---

## Executive Summary & Architecture Overview

In this guide, we construct an enterprise-class **3-Tier Public Key Infrastructure (PKI)** for a local network and homelab environment [31, 113, 191]. Modeled after production zero-trust identity architectures, this design decouples root trust from operational certificate issuance through an intermediate policy layer and an enterprise Registration Authority (RA) control plane:

1. **Tier 1 (Root CA):** Offline & air-gapped. Private key generated and protected within a **YubiHSM 2** hardware security module [10, 112].
2. **Tier 2 (Policy CA):** Semi-offline policy tier. Private key protected by a hardware token (**YubiKey 5 PIV**) or host-bound **TPM 2.0** [10, 33, 46].
3. **Tier 3 (Issuing CA):** Online microservice running **OmniTrust otPKI** in Docker containers [153, 191]. Private key protected via encrypted **text/password secrets**, backed by a **PostgreSQL 16** database [153].
4. **Registration Authority (RA) & Governance Layer:** **OmniTrust Identity Lifecycle Management (ILM)** microservice acting as an automated RA control plane for policy enforcement, ACME enrollment, and certificate lifecycle governance [158, 186, 201].
5. **Certificate Revocation List (CRL) Publisher:** Automated CRL generation and HTTP CRL Distribution Point (CDP) publishing service for active revocation checks across legacy and enterprise clients [138, 187, 190].

---

## Why Upgrade to a 3-Tier otPKI + ILM Architecture?

* **Defense-in-Depth:** Decouples the Root CA from operational Issuing CAs, allowing intermediate revocation or rotation without disturbing client root trust stores [101, 113].
* **OmniTrust ILM as Registration Authority (RA):** Separates issuance policy and client authentication from the core CA signing engine [186, 201]. ILM provides a centralized control plane for ACME/SCEP/CMP protocols, policy compliance checks, and approval workflows [156, 187].
* **Active & Passive Revocation:** Combines short-lived certificate issuance via automated ACME with active **Certificate Revocation List (CRL)** publishing for clients requiring strict CRL Distribution Point (CDP) validation [138, 190].
* **Hardware-Rooted Assurance:** Root private key operations remain strictly locked within physical hardware (YubiHSM 2 / YubiKey / TPM 2.0) [10, 31, 112].
* **Microservices Native on Ubuntu:** Deployed via Docker Compose microservices using standard PostgreSQL data storage and Traefik ingress routing [153, 158].

---

## 3-Tier Architecture & Component Specifications

| PKI Tier / Service | Role & Operational Function | Cryptographic Key / Engine | Deployment Model |
| :--- | :--- | :--- | :--- |
| **Tier 1: Root CA** | Apex Trust Anchor (Offline / Air-Gapped) | **YubiHSM 2** (PKCS#11) | Dedicated Air-Gapped Host |
| **Tier 2: Policy CA** | Intermediate Policy Issuer (Semi-Offline) | **YubiKey 5 (PIV)** or **TPM 2.0** | Admin Workstation / Air-Gapped System |
| **Tier 3: Issuing CA** | Operational Signing Engine (**otPKI**) | Encrypted Password / Text File | **Docker Container** (Ubuntu Host) |
| **RA Layer (ILM)** | Registration Authority & ACME Gateway | **OmniTrust ILM** Control Plane | **Docker Container** (Microservice) |
| **Database** | Shared Persistent Storage | PostgreSQL 16 DB | **Docker Container** (Persistent Volume) |
| **CRL Endpoint** | Public CRL Distribution Point (CDP) | Nginx / Traefik HTTP Endpoint | **Docker Container** (Static File / API) |

---

## System Requirements & Prerequisites

* **Host Platform:** Ubuntu Server 24.04 LTS (x86_64) [33]
* **Container Environment:** Docker 24.0+ and Docker Compose v2.0+ [151]
* **Database:** PostgreSQL 16 Alpine container [153]
* **PKI Software Stack:** OmniTrust otPKI monolith/microservice image and OmniTrust ILM core image [153, 158, 186]
* **Hardware Tokens:** YubiHSM 2 (Root CA), YubiKey 5 NFC / TPM 2.0 (Policy CA) [10, 31, 33]

---

## Part 1: Host Preparation & Network Setup

### 1.1 Install System Dependencies on Ubuntu Server
Log into your Ubuntu Server host and install Docker, system utilities, and hardware driver libraries [33, 35, 151]:

```bash
# Update Ubuntu system packages
sudo apt update && sudo apt upgrade -y

# Install prerequisite tools, PC/SC daemon, and TPM 2.0 utilities
sudo apt install -y \
    curl \
    jq \
    pcscd \
    yubikey-manager \
    tpm2-tools \
    libpcsclite-dev \
    gnupg2 \
    docker.io \
    docker-compose-v2 \
    nginx

# Add administrative user to Docker group
sudo usermod -aG docker $USER
```

### 1.2 Setup Workspace Directories
Create standard host directories for configuration, certificates, secrets, CRL publishing, and database persistence [43, 153]:

```bash
mkdir -p ~/homelab-pki/{config,certs,secrets,crl_www,postgres_data,ilm_config}
cd ~/homelab-pki
```

---

## Part 2: Tier 1 — Bootstrapping the Offline Root CA (YubiHSM 2)

The **Root CA** is the apex trust anchor [101, 112]. Its private key is generated directly inside a **YubiHSM 2** and never leaves the hardware module.

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
On an air-gapped machine, configure the YubiHSM connector and PKCS#11 module configuration [10]:

```bash
sudo tee /etc/yubihsm_pkcs11.conf > /dev/null << 'EOF'
connector = http://localhost:12345
debug
EOF
```

### 2.2 Generate Root Certificate in YubiHSM Slot
Generate an ECC P-256 root keypair inside object slot 1 of the YubiHSM and export the self-signed root certificate `root_ca.crt` [45, 102]:

```bash
# Generate Root CA key pair directly inside YubiHSM
yubihsm-shell -a generate-key \
    --object-id=1 \
    --label="Homelab Root CA Key" \
    --algorithm=ecdsa-p256 \
    --capabilities=sign-attestation,sign-ecdsa

# Copy public Root Certificate to persistent storage
cp /mnt/cold-storage/root_ca.crt ~/homelab-pki/certs/root_ca.crt
```

---

## Part 3: Tier 2 — Policy Tier CA (YubiKey / TPM Protected)

The **Policy CA** defines path length constraints, organizational policy rules, and signs operational Issuing CA certificates [113].

```
 +-------------------------------------------------------+
 |                 TIER 2: POLICY CA                     |
 |  - Subject: "Homelab Policy CA"                       |
 |  - Status: Semi-Offline / Administrative              |
 |  - Key Protection: YubiKey 5 (PIV) OR TPM 2.0        |
 +-------------------------------------------------------+
                            |
                            | Signs Issuing CA Cert
                            v
```

### 3.1 Option A: YubiKey 5 PIV Protection (Slot 9c)
Import the Policy CA certificate and key into YubiKey PIV slot 9c [33, 46]:

```bash
sudo systemctl enable --now pcscd
ykman piv certificates import 9c policy_ca.crt
```

### 3.2 Option B: TPM 2.0 Protection
Lock the Policy CA signing secret to the Ubuntu host's TPM 2.0 module using `systemd-creds` [10, 18]:

```bash
sudo systemctl-creds setup
```

---

## Part 4: Tier 3 — Deploying Containerized otPKI, ILM RA, CRL Publisher & PostgreSQL

Tier 3 consists of three core microservices operating on Ubuntu:
1. **OmniTrust otPKI:** Core certificate authority signing engine [153, 191].
2. **OmniTrust ILM:** Registration Authority (RA) providing policy enforcement, web control plane, and ACME/SCEP/CMP protocols [156, 186, 201].
3. **CRL Publisher:** Exposes HTTP CRL Distribution Points (CDP) for certificate revocation validation [138, 190].

```
                +------------------------------------+
                |       TIER 3: ISSUING CA           |
                |  - OmniTrust otPKI Signing Engine  |
                |  - Text/Password Key Protection    |
                +------------------------------------+
                                  ^
                                  | CA Signing Channel
                                  v
 +-------------------------------------------------------------------+
 |             REGISTRATION AUTHORITY (RA) CONTROL PLANE             |
 |  - OmniTrust ILM (Identity Lifecycle Management)                  |
 |  - Policy Verification, ACME/EST/CMP Endpoints & Web Admin        |
 +-------------------------------------------------------------------+
           |                                             |
           | Serves ACME / TLS Certs                     | Publishes CRL
           v                                             v
 +-------------------+                         +---------------------+
 | Homelab Workloads |                         | HTTP CRL Publisher  |
 | & Microservices   |                         | (CDP Distribution)  |
 +-------------------+                         +---------------------+
```

### 4.1 Create Password Secret for Issuing CA Key
Store the password used to protect the operational Issuing CA private key in `secrets/issuing_ca_password.txt` [18, 48]:

```bash
echo "HomelabPassphrase2026!" > ~/homelab-pki/secrets/issuing_ca_password.txt
chmod 600 ~/homelab-pki/secrets/issuing_ca_password.txt
```

### 4.2 Configure otPKI Engine (`config/otpki.env`)
Define environment parameters for the otPKI backend service [152, 153]:

```ini
APP_NAME=otpki
APP_DOMAIN=pki.homelab.internal
POSTGRES_USER=otpki_user
POSTGRES_PASSWORD=DB_Secret_Password_2026
POSTGRES_DB=otpkidb
POSTGRES_HOST=postgres
CA_ROOT_CERT_PATH=/etc/otpki/certs/root_ca.crt
CA_ISSUING_CERT_PATH=/etc/otpki/certs/issuing_ca.crt
CA_ISSUING_KEY_PATH=/etc/otpki/secrets/issuing_ca.key
CA_KEY_PASSWORD_FILE=/etc/otpki/secrets/issuing_ca_password.txt
CRL_ENABLED=true
CRL_FILE_PATH=/var/www/crl/homelab.crl
CRL_DISTRIBUTION_POINT=http://crl.homelab.internal/crl/homelab.crl
```

### 4.3 Configure OmniTrust ILM as RA (`ilm_config/ilm.env`)
Configure OmniTrust ILM to interface as the Registration Authority (RA) for otPKI [156, 186]:

```ini
ILM_APP_NAME=ot-ilm
ILM_ADMIN_EMAIL=admin@homelab.internal
ILM_DB_HOST=postgres
ILM_DB_USER=otpki_user
ILM_DB_PASSWORD=DB_Secret_Password_2026
ILM_DB_NAME=otpkidb
RA_AUTHORITY_PROVIDER=otpki
RA_ACME_ENABLED=true
RA_ACME_URL=https://pki.homelab.internal/acme/directory
RA_CRL_AUTO_PUBLISH=true
RA_CRL_SYNC_INTERVAL=3600
```

### 4.4 Build Docker Compose Microservices Architecture
Create `~/homelab-pki/docker-compose.yml` to orchestrate PostgreSQL, otPKI, ILM RA, Traefik ingress, and the HTTP CRL Distribution Point [153, 158]:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    container_name: pki-postgres
    restart: always
    environment:
      POSTGRES_USER: otpki_user
      POSTGRES_PASSWORD: DB_Secret_Password_2026
      POSTGRES_DB: otpkidb
    volumes:
      - ./postgres_data:/var/lib/postgresql/data
    networks:
      - pki-network

  otpki-engine:
    image: omnitrust/otpki-engine:latest
    container_name: pki-otpki-engine
    restart: always
    depends_on:
      - postgres
    env_file:
      - ./config/otpki.env
    volumes:
      - ./certs:/etc/otpki/certs:ro
      - ./secrets:/etc/otpki/secrets:ro
      - ./crl_www:/var/www/crl
    networks:
      - pki-network

  ilm-ra:
    image: omnitrust/ilm-core:latest
    container_name: pki-ilm-ra
    restart: always
    depends_on:
      - otpki-engine
    env_file:
      - ./ilm_config/ilm.env
    ports:
      - "8443:8443"
    volumes:
      - ./certs:/etc/ilm/certs:ro
      - ./crl_www:/var/www/crl
    networks:
      - pki-network

  crl-publisher:
    image: nginx:alpine
    container_name: pki-crl-publisher
    restart: always
    ports:
      - "8080:80"
    volumes:
      - ./crl_www:/usr/share/nginx/html/crl:ro
    networks:
      - pki-network

networks:
  pki-network:
    driver: bridge
```

### 4.5 Initialize Stack & Verify Deployment
Start the microservices stack [153]:

```bash
docker compose up -d
docker compose ps
```

---

## Part 5: Configuring ILM RA Workflows & Active CRL Publishing

### 5.1 Access OmniTrust ILM Admin Control Plane
Open your browser to `https://pki.homelab.internal:8443` or `https://<HOST_IP>:8443` to log into the OmniTrust ILM administrator interface [154, 161]:
* **Default Admin User:** `admin@homelab.internal` [154]
* **Initial Actions:** Configure Super Administrator credentials and assign the default RA Profile to the otPKI backend connector [156, 160].

### 5.2 Validate ACME Registration Endpoint
OmniTrust ILM exposes ACME endpoints on behalf of otPKI [127, 157]. Test ACME certificate issuance using `certbot`:

```bash
certbot certonly --standalone \
    --server https://pki.homelab.internal:8443/acme/directory \
    -d service.homelab.internal \
    --no-verify-ssl
```

### 5.3 Verify CRL Generation & CDP Distribution Point
Verify that otPKI generates and updates the Certificate Revocation List (`homelab.crl`) at the HTTP CRL Distribution Point [138, 190]:

```bash
# Fetch CRL from Nginx CRL publisher endpoint
curl -v http://pki.homelab.internal:8080/crl/homelab.crl -o /tmp/homelab.crl

# Inspect CRL contents using OpenSSL
openssl crl -in /tmp/homelab.crl -inform DER -text -noout
```

---

## Maintenance & Operational Checklist

1. **Root CA Offline Security:** Keep the YubiHSM 2 disconnected in a secure location except during annual Policy CA renewals or emergency revocation procedures [112].
2. **PostgreSQL Database Backups:** Automate database backups for audit trails, issuance history, and serial number state tracking [153]:
   ```bash
   docker exec pki-postgres pg_dump -U otpki_user otpkidb > ~/homelab-pki/backups/otpkidb_backup.sql
   ```
3. **CRL Renewal Cron:** Ensure otPKI re-publishes updated CRL files every 24 hours to prevent CRL stale expiration errors on relying clients [138].

---

## Summary & Further Reading

You have successfully constructed an enterprise 3-tier PKI stack using OmniTrust otPKI and ILM:
* **Root CA** locked in hardware inside a **YubiHSM 2** [31, 112].
* **Policy CA** protected via **YubiKey 5 / TPM 2.0** [10, 33, 46].
* **Issuing CA Engine (otPKI)** running as a container microservice [153, 191].
* **Registration Authority (ILM)** managing ACME enrollment, compliance, and governance [158, 186].
* **CRL Publishing** delivering active certificate revocation checks over HTTP [138, 190].

### Related Documents in Notebook:
* *OmniTrust Public Key Infrastructure*
* *Introduction | ILM Documentation*
* *Streamline Identity Lifecycle Management with enterprise PKI*
* *Build a Tiny Certificate Authority For Your Homelab*
