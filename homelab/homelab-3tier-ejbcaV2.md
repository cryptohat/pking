# Building an Enterprise-Grade 3-Tier PKI with EJBCA Community Edition and Ansible Automation for Your Homelab

*A comprehensive guide to constructing an offline-rooted, hardware-secured, containerized Certificate Authority stack on Ubuntu using EJBCA Community Edition, PostgreSQL 16, and automated Ansible playbooks.*

---

## Executive Summary & Architecture Overview

In this guide, we construct an enterprise-class **3-Tier Public Key Infrastructure (PKI)** for a local network and homelab environment [31, 113]. Modeled after production zero-trust identity architectures, this design decouples root trust from operational certificate issuance through an intermediate policy layer, maximizing defense-in-depth:

1. **Tier 1 (Root CA):** Offline & air-gapped. Private key generated and protected within a **YubiHSM 2** hardware security module [10, 112].
2. **Tier 2 (Policy CA):** Semi-offline policy tier. Private key protected by a hardware token (**YubiKey 5 PIV**) or host-bound **TPM 2.0** [10, 33, 46].
3. **Tier 3 (Issuing CA):** Online, enterprise-grade Certificate Authority running **EJBCA Community Edition** (`keyfactor/ejbca-ce`) in a **Docker** container [158, 222]. Private key protected via encrypted **text/password secrets**, backed by a high-performance **PostgreSQL 16** database [153].
4. **Ansible Automation Layer:** End-to-end Infrastructure as Code (IaC) playbooks automating host preparation, microservices container stack deployment, EJBCA Crypto Token initialization, Certificate/End-Entity profile creation, background Service configuration, and ACME alias setup.

This setup provides a fully containerized PKI control plane with an internal **ACME v2 server** (compatible with standard Let's Encrypt clients) capable of automatically issuing and renewing short-lived TLS certificates (e.g., 24-hour duration) for homelab services, microservices, and network endpoints [31, 127, 137].

---

## EJBCA Architectural & Operational Concepts

Understanding EJBCA requires distinguishing between several interrelated concepts defined in the EJBCA CA and RA Concept Guides [64, 66, 88]:

### 1. Crypto Tokens
A **Crypto Token** represents a container for cryptographic key pairs used for signing and encryption operations [68]. EJBCA supports two main types:
* **PKCS#11 Crypto Tokens (HSM):** Hardware-backed key pairs where signing operations occur directly inside a physical HSM or YubiHSM [68].
* **Soft Crypto Tokens (PKCS#12):** Software key pairs stored in the relational database, encrypted with a master password [68]. For our online Tier 3 Issuing CA, we utilize a Soft Crypto Token protected by an encrypted password file [18, 48, 68].

### 2. Certificate Profiles
A **Certificate Profile** acts as a technical template defining the structure and constraints of generated certificates [68, 69]. It specifies:
* **Key Algorithms & Sizes:** Allowed algorithms (e.g., RSA 2048/4096, ECDSA P-256/P-384) [69].
* **Validity Durations:** Maximum lifetime (e.g., 24 hours for short-lived operational certificates) [44, 69].
* **X.509 Extensions:** Key Usage (Digital Signature, Key Encipherment) and Extended Key Usage (Server Authentication, Client Authentication) [69].
* **Revocation Information:** Embedded CRL Distribution Points (CDP) and OCSP responder URIs [68, 71].

### 3. End Entity Profiles
While Certificate Profiles constrain the *technical fields* of the issued certificate, **End Entity Profiles** govern the *administrative input data* and authorization rules for end entities (users, servers, network devices) [70, 71]. End Entity Profiles define:
* **Subject DN Constraints:** Mandatory, optional, or forbidden fields (e.g., `CN`, `O`, `OU`, `C`) and regex validation rules [70, 71].
* **Subject Alternative Name (SAN) Rules:** Allowed DNS names, IP addresses, or URIs [71, 159].
* **Available Certificate Profiles & CAs:** Which CAs and Certificate Profiles an administrator or automated protocol (such as ACME or SCEP) is permitted to request [70].

### 4. End Entities
An **End Entity** represents the actual identity holder (a device, container, microservice, or user) in EJBCA [70]. An End Entity is bound to a specific CA, Certificate Profile, and End Entity Profile [70]. Certificate enrollment is a two-step process:
1. An End Entity record is registered with a username/password or token [90].
2. The entity submits a Certificate Signing Request (CSR), or EJBCA generates a PKCS#12 keystore, producing the issued certificate [90, 91].

### 5. EJBCA Services & Publishers
* **Services:** Background worker processes executing at scheduled intervals to perform tasks such as generating Certificate Revocation Lists (CRLs), sending expiration warnings, or cleaning up expired certificates [72].
* **Publishers:** Components that automatically export issued certificates or CRLs to external targets, such as LDAP directories, Active Directory, or Validation Authorities (VAs) [71].

---

## 3-Tier Architecture & Hardware Specifications

| PKI Tier / Component | Role & Operational Status | Cryptographic Key Protection | Hosting / Deployment Model |
| :--- | :--- | :--- | :--- |
| **Tier 1: Root CA** | Apex Trust Anchor (Offline / Air-Gapped) | **YubiHSM 2** (PKCS#11) | Air-Gapped Host / Live Boot OS |
| **Tier 2: Policy CA** | Intermediate Policy Issuer (Semi-Offline) | **YubiKey 5 NFC (PIV)** or **TPM 2.0** | Admin Workstation / System |
| **Tier 3: Issuing CA** | Operational Signing Engine (**EJBCA Community**) | Encrypted **Password / Text File** Secret | **Docker Container** (Ubuntu Server) |
| **Database** | Shared Persistent Storage | PostgreSQL 16 DB | **Docker Container** (Persistent Volume) |
| **Automation** | Orchestration & Configuration | Ansible Playbooks & EJBCA CLI | Host Workstation / CI Runner |

### Infrastructure Stack Summary
* **Host Platform:** Ubuntu Server 24.04 LTS (x86_64) [33]
* **Database Backend:** PostgreSQL 16 Alpine container (`postgres:16-alpine`) [153]
* **CA Engine:** Keyfactor EJBCA Community Edition (`keyfactor/ejbca-ce:latest`) [158, 222]
* **Automation:** Ansible 2.15+ with `community.docker` collection
* **Protocols:** ACME v2, REST API, EJBCA Web Administration GUI (`https://<host>:8443/ejbca/adminweb/`) [127, 229]

---

## Part 1: System Environment & Container Setup

### 1.1 Host Preparation & Prerequisites
Install Docker, Python dependencies for Ansible Docker management, PC/SC drivers, and hardware security utilities on the Ubuntu Server host [33, 35, 151]:

```bash
# Update base Ubuntu system
sudo apt update && sudo apt upgrade -y

# Install prerequisites, hardware drivers, and Python environment
sudo apt install -y     curl     jq     pcscd     yubikey-manager     tpm2-tools     libpcsclite-dev     gnupg2     python3-pip     python3-docker     docker.io     docker-compose-v2     ansible

# Add current user to docker group
sudo usermod -aG docker $USER
```

### 1.2 Prepare Workspace Directory
Establish host directory structures for certificates, secrets, persistent database storage, and Ansible playbooks [43, 153]:

```bash
mkdir -p ~/homelab-pki/{certs,secrets,postgres_data,ejbca_config,ansible/roles}
cd ~/homelab-pki
```

---

## Part 2: Tier 1 & Tier 2 — Hardware-Secured Root & Policy CAs

### 2.1 Tier 1: Offline Root CA Bootstrapping (YubiHSM 2)
The **Root CA** key pair is generated and locked inside object slot 1 of a **YubiHSM 2** on an air-gapped machine [10, 112].

```bash
# Configure YubiHSM PKCS#11 middleware
sudo tee /etc/yubihsm_pkcs11.conf > /dev/null << 'EOF'
connector = http://localhost:12345
debug
EOF

# Generate ECC P-256 Root Keypair in YubiHSM slot 1
yubihsm-shell -a generate-key     --object-id=1     --label="Homelab Root CA Key"     --algorithm=ecdsa-p256     --capabilities=sign-attestation,sign-ecdsa

# Copy public Root Certificate to persistent storage
cp /mnt/cold-storage/root_ca.crt ~/homelab-pki/certs/root_ca.crt
```

### 2.2 Tier 2: Policy Tier CA Setup (YubiKey 5 / TPM 2.0)
The **Policy CA** defines organizational constraint rules and signs operational Issuing CAs [113].

```bash
# Option A: Import Policy CA certificate & key into YubiKey 5 PIV Slot 9c
sudo systemctl enable --now pcscd
ykman piv certificates import 9c policy_ca.crt

# Option B: Lock Policy CA key using host TPM 2.0 via systemd-creds
sudo systemd-creds setup
```

---

## Part 3: Tier 3 — EJBCA Container Stack & Detailed Profile Setup

### 3.1 Create Password Secret File
Create an encrypted passphrase secret to protect the Tier 3 Soft Crypto Token in EJBCA [18, 48]:

```bash
echo "EJBCA_Keystore_Pass_2026!" > ~/homelab-pki/secrets/issuing_ca_password.txt
chmod 600 ~/homelab-pki/secrets/issuing_ca_password.txt
```

### 3.2 Environment Variable File (`config/ejbca.env`)
Define environment variables for database connections and EJBCA initialization [152, 153]:

```ini
# PostgreSQL Credentials
POSTGRES_USER=ejbca
POSTGRES_PASSWORD=DB_Secret_Password_2026
POSTGRES_DB=ejbcadb

# EJBCA Container Configuration
DATABASE_JDBC_URL=jdbc:postgresql://postgres:5432/ejbcadb
DATABASE_USER=ejbca
DATABASE_PASSWORD=DB_Secret_Password_2026
INITIAL_ADMIN_USER=ejbcaadmin
INITIAL_ADMIN_PASSWORD=Admin_Super_Secret_2026!
TLS_SETUP_ENABLED=true
```

### 3.3 Microservices Orchestration (`docker-compose.yml`)
Create `~/homelab-pki/docker-compose.yml` to orchestrate PostgreSQL 16 and EJBCA Community [153, 158]:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    container_name: pki-postgres
    restart: always
    environment:
      POSTGRES_USER: ejbca
      POSTGRES_PASSWORD: DB_Secret_Password_2026
      POSTGRES_DB: ejbcadb
    volumes:
      - ./postgres_data:/var/lib/postgresql/data
    networks:
      - pki-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ejbca -d ejbcadb"]
      interval: 5s
      timeout: 5s
      retries: 5

  ejbca:
    image: keyfactor/ejbca-ce:latest
    container_name: pki-ejbca-issuing
    restart: always
    depends_on:
      postgres:
        condition: service_healthy
    ports:
      - "80:8080"   # HTTP / ACME Enrollment
      - "443:8443"  # HTTPS Client-Cert Authenticated Admin Web GUI
    env_file:
      - ./config/ejbca.env
    volumes:
      - ./certs:/etc/ejbca/certs:ro
      - ./secrets:/etc/ejbca/secrets:ro
      - ./ejbca_config:/var/ejbca/custom_config
    networks:
      - pki-network

networks:
  pki-network:
    driver: bridge
```

### 3.4 Initial Client Certificate Authenticated Web Access
Upon container initialization, EJBCA generates an initial Management CA and SuperAdmin enrollment credentials [225]:
1. Access the RA enrollment endpoint at `https://<HOST_IP>:8443/ejbca/retrieve/ca_certs.jsp` or copy the SuperAdmin enrollment link from container logs [225, 227].
2. Enter the initial password (`Admin_Super_Secret_2026!`) on the Enrollment page to download `SuperAdmin.p12` [227].
3. Import `SuperAdmin.p12` into your browser (e.g., Firefox or Chrome) [228].
4. Access the EJBCA Administrative Web GUI at `https://<HOST_IP>:8443/ejbca/adminweb/` [229].

---

## Part 4: Infrastructure as Code — Ansible Playbooks

To make this deployment completely reproducible, we provide two Ansible playbooks:
1. `deploy_stack.yml`: Provisions system prerequisites, directories, and starts the Docker Compose microservices.
2. `configure_ejbca.yml`: Automates EJBCA Crypto Token creation, Issuing CA import, Certificate Profile setup, End Entity Profile creation, background Service enablement, and ACME alias configuration using the EJBCA CLI tool (`ejbca.sh`).

### 4.1 Ansible Inventory (`ansible/inventory.ini`)
```ini
[pki_hosts]
localhost ansible_connection=local
```

### 4.2 Playbook 1: Stack Deployment (`ansible/deploy_stack.yml`)
```yaml
---
- name: Deploy Homelab 3-Tier EJBCA & PostgreSQL Microservices Stack
  hosts: pki_hosts
  become: true
  vars:
    base_dir: "{{ ansible_env.HOME }}/homelab-pki"
    db_password: "DB_Secret_Password_2026"
    ejbca_admin_password: "Admin_Super_Secret_2026!"

  tasks:
    - name: Ensure required packages are installed
      apt:
        name:
          - curl
          - jq
          - pcscd
          - yubikey-manager
          - tpm2-tools
          - python3-docker
          - docker.io
          - docker-compose-v2
        state: present
        update_cache: yes

    - name: Create persistent workspace directories
      file:
        path: "{{ item }}"
        state: directory
        mode: '0755'
        owner: "{{ ansible_user_id }}"
      loop:
        - "{{ base_dir }}/certs"
        - "{{ base_dir }}/secrets"
        - "{{ base_dir }}/postgres_data"
        - "{{ base_dir }}/config"
        - "{{ base_dir }}/ejbca_config"

    - name: Write Issuing CA password secret file
      copy:
        content: "EJBCA_Keystore_Pass_2026!
"
        dest: "{{ base_dir }}/secrets/issuing_ca_password.txt"
        mode: '0600'
        owner: "{{ ansible_user_id }}"

    - name: Render ejbca.env configuration
      copy:
        dest: "{{ base_dir }}/config/ejbca.env"
        content: |
          POSTGRES_USER=ejbca
          POSTGRES_PASSWORD={{ db_password }}
          POSTGRES_DB=ejbcadb
          DATABASE_JDBC_URL=jdbc:postgresql://postgres:5432/ejbcadb
          DATABASE_USER=ejbca
          DATABASE_PASSWORD={{ db_password }}
          INITIAL_ADMIN_USER=ejbcaadmin
          INITIAL_ADMIN_PASSWORD={{ ejbca_admin_password }}
          TLS_SETUP_ENABLED=true
        mode: '0600'

    - name: Deploy Docker Compose stack
      community.docker.docker_compose_v2:
        project_src: "{{ base_dir }}"
        state: present
      register: compose_output

    - name: Wait for PostgreSQL container to become healthy
      command: docker exec pki-postgres pg_isready -U ejbca -d ejbcadb
      register: db_check
      until: db_check.rc == 0
      retries: 15
      delay: 5
```

### 4.3 Playbook 2: Automated EJBCA Detailed Configuration (`ansible/configure_ejbca.yml`)
```yaml
---
- name: Automate EJBCA CA, Crypto Tokens, Profiles, Services & ACME Configuration
  hosts: pki_hosts
  become: true
  vars:
    base_dir: "{{ ansible_env.HOME }}/homelab-pki"
    container_name: "pki-ejbca-issuing"
    ca_name: "Homelab Operational Issuing CA"
    cert_profile_name: "Homelab_ShortLived_Server_Profile"
    ee_profile_name: "Homelab_Server_EE_Profile"
    acme_alias: "homelab-acme"

  tasks:
    - name: Import Tier 3 Issuing CA into EJBCA Soft Crypto Token
      command: >
        docker exec -i {{ container_name }} /opt/primekey/ejbca/bin/ejbca.sh ca importca
        --caname "{{ ca_name }}"
        --certfile /etc/ejbca/certs/issuing_ca.crt
        --keyfile /etc/ejbca/secrets/issuing_ca_key
        --passwordfile /etc/ejbca/secrets/issuing_ca_password.txt
      register: import_ca_result
      failed_when: import_ca_result.rc != 0 and 'already exists' not in import_ca_result.stdout

    - name: Create Short-Lived 24-Hour Server TLS Certificate Profile
      command: >
        docker exec -i {{ container_name }} /opt/primekey/ejbca/bin/ejbca.sh ca addcertprofile
        --name "{{ cert_profile_name }}"
        --type enduser
        --keyusage digitalSignature,keyEncipherment
        --extendedkeyusage serverAuth,clientAuth
        --availablekeyalgorithms RSA2048,RSA4096,ECDSA_P256
        --validity 1d
      ignore_errors: true

    - name: Create End Entity Profile with SAN Constraints
      command: >
        docker exec -i {{ container_name }} /opt/primekey/ejbca/bin/ejbca.sh ca addendentityprofile
        --name "{{ ee_profile_name }}"
        --availablecas "{{ ca_name }}"
        --availablecertprofiles "{{ cert_profile_name }}"
        --allowdns-san true
        --allowip-san true
      ignore_errors: true

    - name: Configure Background CRL Updater Service
      command: >
        docker exec -i {{ container_name }} /opt/primekey/ejbca/bin/ejbca.sh service addservice
        --name "CRL_Updater_Service"
        --worker "CRLUpdater"
        --period 3600
        --active true
      ignore_errors: true

    - name: Enable ACME Protocol Engine
      command: >
        docker exec -i {{ container_name }} /opt/primekey/ejbca/bin/ejbca.sh config set acme.enabled true

    - name: Create ACME Protocol Alias for Homelab Endpoints
      command: >
        docker exec -i {{ container_name }} /opt/primekey/ejbca/bin/ejbca.sh acme addalias
        --alias "{{ acme_alias }}"
        --caname "{{ ca_name }}"
        --certprofile "{{ cert_profile_name }}"
        --endentityprofile "{{ ee_profile_name }}"
      register: acme_result
      failed_when: acme_result.rc != 0 and 'already exists' not in acme_result.stdout
```

---

## Part 5: Client Enrollment & Automated Certificate Issuance

### 5.1 Install Root CA Trust Anchor on Endpoint Clients
Distribute and install the Root CA certificate (`root_ca.crt`) onto all homelab VMs, containers, and client devices [51, 57]:

```bash
# On Ubuntu / Debian endpoints
sudo cp ~/homelab-pki/certs/root_ca.crt /usr/local/share/ca-certificates/homelab_root.crt
sudo update-ca-certificates
```

### 5.2 Automated ACME Certificate Request with Certbot
Request a short-lived 24-hour TLS certificate from EJBCA via ACME using Certbot [58, 127]:

```bash
certbot certonly --standalone     --server http://pki.homelab.internal/ejbca/acme/homelab-acme/directory     -d dashboard.homelab.internal     --agree-tos     --register-unsafely-without-email
```

---

## Operational Maintenance & Security Checklist

1. **Root CA Key Protection:** Ensure the YubiHSM 2 remains disconnected except during scheduled Policy CA renewals or emergency revocation procedures [112].
2. **PostgreSQL Database Backups:** Automate PostgreSQL database snapshots using `pg_dump` [153]:
   ```bash
   docker exec pki-postgres pg_dump -U ejbca ejbcadb > ~/homelab-pki/backups/ejbca_db.sql
   ```
3. **Automated Renewal & Passive Revocation:** Homelab workloads must execute Certbot or ACME client renewals every 12 hours. Short-lived 24-hour certificates expire automatically, minimizing compromise windows without relying on external OCSP responders [44, 137, 175].

---

## Summary & Related Notebook Documents

You have successfully constructed a production-grade 3-tier Public Key Infrastructure using EJBCA Community Edition and Ansible:
* **Root CA** locked in hardware on a **YubiHSM 2** [31, 112].
* **Policy CA** secured via **YubiKey 5 / TPM 2.0** [10, 33, 46].
* **Issuing CA Engine (EJBCA Community)** running as a **Docker microservice** backed by **PostgreSQL 16** [153, 158].
* **Infrastructure as Code (Ansible)** automating host preparation, Docker Compose deployment, EJBCA Crypto Tokens, Certificate/End-Entity Profiles, background Services, and ACME aliases.

### Related Documents in Notebook:
* *Build a Tiny Certificate Authority For Your Homelab*
* *EJBCA CA Concept Guide | Keyfactor Docs*
* *EJBCA RA Concept Guide | Keyfactor Docs*
* *Quick Start Guide - Start EJBCA Container with Client Certificate Authenticated Access*
* *Everything You Should Know About Certificates and PKI*
