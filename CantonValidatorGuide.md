# Setting Up a Canton Network Validator Node: A Complete Guide

A step-by-step walkthrough covering everything from prerequisites and network sponsorship, to Docker Compose deployment, authentication, and post-launch health checks everything you need to go from zero to a live Canton validator.

## Table of Contents

1. [What Is the Canton Network?](#what-is-the-canton-network)
2. [Validator vs. Super Validator: Know the Difference](#validator-vs-super-validator)
3. [Prerequisites](#prerequisites)
4. [Step 1: Get a Sponsor & Join the Allowlist](#step-1-get-a-sponsor--join-the-allowlist)
5. [Step 2: Set Up Your Infrastructure](#step-2-set-up-your-infrastructure)
6. [Step 3: Configure OIDC Authentication](#step-3-configure-oidc-authentication)
7. [Step 4: Obtain Your Onboarding Secret](#step-4-obtain-your-onboarding-secret)
8. [Step 5: Deploy via Docker Compose (DevNet)](#step-5-deploy-via-docker-compose-devnet)
9. [Step 6: Kubernetes Deployment (Production)](#step-6-kubernetes-deployment-production)
10. [Step 7: Verify Connectivity](#step-7-verify-connectivity)
11. [Step 8: Access Your Wallet UI](#step-8-access-your-wallet-ui)
12. [Step 9: Promote to TestNet & MainNet](#step-9-promote-to-testnet--mainnet)
13. [Rewards, Traffic & Canton Coin](#rewards-traffic--canton-coin)
14. [Backups & Disaster Recovery](#backups--disaster-recovery)
15. [Monitoring & Observability](#monitoring--observability)
16. [Common Errors & Troubleshooting](#common-errors--troubleshooting)
17. [Summary](#summary)
18. [About Us](#about-us)
19. [FAQ](#faq)



## What Is the Canton Network?

Canton is an enterprise-grade blockchain network built by Digital Asset on top of the Daml smart contract language. Unlike traditional public blockchains that broadcast all data to all nodes, Canton uses a **privacy-by-default architecture**: your validator node only receives and stores data belonging to transactions you are a party to. Every other participant's data remains invisible to you by design.

The network's backbone is the **Global Synchronizer** a decentralized coordination layer operated by a set of Super Validator (SV) nodes. Validators connect to this synchronizer to participate in the ecosystem, submit transactions, and interact with applications such as tokenization platforms, payment rails, and settlement systems.

The native utility token is **Canton Coin (CC)**, used to pay for network traffic fees and earned as liveness rewards for running an active validator node.



## Validator vs. Super Validator

Before diving in, it's important to understand the two node tiers on Canton:

| Feature | Validator Node | Super Validator Node |
|---|---|---|
| Role | Participant in the network | Core infrastructure of the Global Synchronizer |
| Availability | Open to approved applicants | By invitation only |
| Responsibilities | Validate own transactions, run wallet/app UIs | Validate all CC transfers, run sequencer & mediator |
| Rewards | Liveness rewards in Canton Coin | Higher rewards + governance participation |
| Setup Complexity | Moderate | High |
| Slashing | No | No |

This guide covers **validator nodes only**. Super Validator setup is a separate, invite-only process.



## Prerequisites

Before you touch any config file, make sure you have the following in place.

### Software Requirements

| Tool | Minimum Version | Notes |
|---|---|---|
| Docker | 24.0+ | Engine + CLI |
| Docker Compose | 2.26.0+ | Run `docker compose version` |
| `kubectl` | v1.26.1+ | For Kubernetes path only |
| `helm` | v3.x | For Kubernetes path only |
| `curl` | Any recent | For connectivity checks |
| `jq` | Any recent | For parsing JSON outputs |
| PostgreSQL | 14+ | Managed by Docker Compose; external for K8s |

### Hardware Requirements (Minimum for DevNet/TestNet)

| Resource | Minimum | Recommended (Production) |
|---|---|---|
| vCPUs | 4 | 8 |
| RAM | 8 GB | 16 GB |
| Disk | 100 GB SSD | 500 GB SSD |
| OS | Ubuntu 22.04 LTS | Ubuntu 22.04 LTS / Debian 12 |
| Architecture | AMD64 or ARM64 | AMD64 |

### Network Requirements

- A **static egress IP address** is mandatory. The Canton network uses IP allowlisting at the firewall level; dynamic or NAT'd IPs will not work unless tunneled through a VPN that has been whitelisted.
- Alternatively, you can connect your validator through a **VPN operated by your sponsor SV**.
- Outbound HTTPS (port 443) to Super Validator sequencer and Scan endpoints.
- Inbound ports for the wallet and CNS UIs (typically 80/443 behind a reverse proxy).

### Account Requirements

- An **OIDC-compatible Identity Provider** (Auth0, Keycloak, Azure AD, Okta, or similar)
- An existing **Super Validator sponsor** to whitelist your IP and issue an onboarding secret
- Access to the [Splice release bundle](https://docs.dev.sync.global) (open-source, hosted on GitHub)



## Step 1: Get a Sponsor & Join the Allowlist

Canton MainNet operates in an **invite-only** model. To join, you must be sponsored by an existing Super Validator (SV), an existing validator, an application provider, or the Canton Foundation itself.

### How to Find a Sponsor

- Request access through the **Canton Foundation form** at [sync.global/validator-request](https://sync.global/validator-request)
- Reach out directly to a listed Super Validator operator (Digital Asset, Tradeweb, Cumberland, etc.)
- For DevNet, any SV can sponsor you and the process is largely self-service

### Submit Your Egress IP for Whitelisting

Once you have a sponsor, provide them with the **static egress IP** of the machine or cluster that will run your validator. Each network (DevNet, TestNet, MainNet) requires a **distinct** IP address you cannot reuse the same IP across networks.

Your sponsor will submit your IP to the Super Validators for inclusion in their firewall allowlist. This covers both the Scan endpoint and the sequencer endpoint connectivity your validator needs.

```bash
# Confirm your egress IP before sending it to your sponsor
curl -s https://api.ipify.org
```

> ⚠️ **Important:** Run this command from the exact machine or cluster that will host your validator. If you run it from your laptop, you will send the wrong IP.



## Step 2: Set Up Your Infrastructure

### Docker Compose Path (DevNet / Small Production)

Provision a Linux VM (on AWS, GCP, Azure, or bare metal) meeting the hardware specs above. Install Docker and Docker Compose:

```bash
# Install Docker Engine
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add your user to the docker group
sudo usermod -aG docker $USER
newgrp docker

# Verify
docker --version
docker compose version
```

Both AMD64 and ARM64 architectures are supported. The output should show Docker Compose **2.26.0 or newer**; older versions will fail silently on certain config features.

### Kubernetes Path (Production)

For production deployments, a Kubernetes-based setup using Helm charts is strongly recommended. It provides better scalability, built-in health probes, and supports Grafana-based monitoring out of the box.

Requirements:

- A running Kubernetes cluster with **admin access** to create and manage namespaces
- `kubectl` (v1.26.1+) and `helm` (v3.x) installed on your workstation
- An external PostgreSQL database (RDS, Cloud SQL, or self-managed)
- An ingress controller (nginx-ingress recommended) with TLS termination

Download the Helm chart bundle from the official Splice release page and extract it:

```bash
# Download and extract the release bundle
curl -L https://github.com/hyperledger-labs/splice/releases/latest/download/splice-node.tar.gz \
  -o splice-node.tar.gz
tar -xzf splice-node.tar.gz
cd splice-node/
```



## Step 3: Configure OIDC Authentication

This is the step that trips up most new operators. Canton validator nodes use **OAuth 2.0 / OIDC** for two distinct authentication flows, and both need to be configured before you start the node.

### Flow 1: Machine-to-Machine (Client Credentials Grant)

Used internally: the **validator app backend** authenticates to the **Canton participant node**. Your OIDC provider must support the OAuth 2.0 Client Credentials flow, and you need to create a `(CLIENT_ID, CLIENT_SECRET)` pair for this.

The `sub` field of JWTs issued through this flow must match the `ledger-api-user` you configure later. Most providers (Auth0, Keycloak) form this as `CLIENT_ID@clients` by default.

### Flow 2: User-Facing (Authorization Code Grant)

Used by humans accessing the **Wallet UI** and **Canton Name Service (CNS) UI**. Your OIDC provider must support the Authorization Code Grant flow and expose `sub` as a stable, unique user identifier.

### Required Configuration Values

Export these variables you will need them during deployment:

```bash
# OIDC Provider configuration
export OIDC_AUTHORITY="https://YOUR_TENANT.auth0.com/"
export OIDC_CLIENT_ID="your-validator-client-id"
export OIDC_CLIENT_SECRET="your-validator-client-secret"
export VALIDATOR_WALLET_ADMIN_USER="auth0|your-admin-user-sub"

# JWT Audience start with this default, harden later
export LEDGER_API_AUTH_AUDIENCE="https://canton.network.global"
export VALIDATOR_AUTH_AUDIENCE="https://canton.network.global"
```

> 💡 **Tip:** When starting out, setting both audiences to `https://canton.network.global` is fine and simplifies debugging. Once your node is running, configure **dedicated per-deployment audience values** to prevent tokens from one network being usable on another.

### Callback URLs to Register in Your OIDC Provider

```
http://wallet.localhost      ← Wallet UI
http://ans.localhost         ← Canton Name Service UI
```

For Kubernetes deployments using real domain names, replace `localhost` with your actual domain (e.g., `https://wallet.mycompany.com`).



## Step 4: Obtain Your Onboarding Secret

Your onboarding secret is a **one-time-use token** provided by your sponsor SV that authorizes your node to join the network. On MainNet and TestNet, your sponsor must generate this for you manually. On DevNet, you can self-generate it.

### Self-Generate on DevNet

```bash
# Replace SPONSOR_SV_URL with your sponsor's SV app URL
# Example for GSF (Global Synchronizer Foundation):
SPONSOR_SV_URL="https://sv.sv-1.dev.global.canton.network.sync.global"

curl -X POST "${SPONSOR_SV_URL}/api/sv/v0/devnet/onboard/validator/prepare"
```

The response contains your `onboardingSecret`. Copy it immediately it is only valid for **1 hour** on DevNet (48 hours for manually issued secrets on TestNet/MainNet).

> ⚠️ Secrets are **single-use and time-limited**. If your deployment fails and you need to retry, request a new secret from your sponsor.



## Step 5: Deploy via Docker Compose (DevNet)

This is the fastest path to a running validator. The Docker Compose setup bundles the participant node, validator backend, wallet UI, and CNS UI into a single orchestrated deployment.

### Download and Extract the Bundle

```bash
# Download the latest Splice release bundle
curl -L https://github.com/hyperledger-labs/splice/releases/latest/download/splice-node.tar.gz \
  -o splice-node.tar.gz
tar -xzf splice-node.tar.gz
cd splice-node/docker-compose/
```

### Configure Your Environment

Create your `.env` file from the provided template and populate it:

```bash
cp .env.example .env
```

Edit `.env` with the values prepared in the previous steps:

```bash
# .env file key variables

# Network
MIGRATION_ID=9                            # Get current value from your sponsor or docs
SPONSOR_SV_URL=https://sv.sv-1.dev.global.canton.network.sync.global

# Validator Identity
PARTY_HINT=myOrg-myValidator-1            # Format: <org>-<function>-<enumerator>

# OIDC Auth (for production skip for unauthenticated DevNet quickstart)
AUTH_OIDC_AUTHORITY=https://YOUR_TENANT.auth0.com/
AUTH_OIDC_CLIENT_ID=your-client-id
AUTH_OIDC_CLIENT_SECRET=your-client-secret
WALLET_ADMIN_USER=auth0|your-admin-sub
```

### Start the Validator Node

```bash
./start.sh \
  -s "<SPONSOR_SV_URL>" \
  -o "<ONBOARDING_SECRET>" \
  -p "<PARTY_HINT>" \
  -m "<MIGRATION_ID>"
```

A real example for DevNet:

```bash
./start.sh \
  -s "https://sv.sv-1.dev.global.canton.network.sync.global" \
  -o "abc123...your-secret..." \
  -p "acmeCorp-primaryValidator-1" \
  -m "9"
```

The node will pull Docker images, initialize its PostgreSQL database, connect to the Global Synchronizer, and complete onboarding. This typically takes **3-10 minutes** on first boot. Subsequent restarts are much faster.

### Stop and Restart

```bash
# Stop the node (data is preserved in Docker volumes)
./stop.sh

# Restart using the same parameters
# The -o flag is still required but the secret value can be empty after initial onboarding
./start.sh -s "<SPONSOR_SV_URL>" -o "" -p "<PARTY_HINT>" -m "<MIGRATION_ID>"
```

### Systemd Integration (Optional)

If you want the validator to survive reboots automatically:

```ini
# /etc/systemd/system/canton-validator.service
[Unit]
Description=Canton Validator Node
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=true
WorkingDirectory=/home/ubuntu/splice-node/docker-compose
ExecStart=/bin/bash start.sh -s "SPONSOR_URL" -o "" -p "PARTY_HINT" -m "MIGRATION_ID"
ExecStop=/bin/bash stop.sh

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable canton-validator
sudo systemctl start canton-validator
```

## Step 6: Kubernetes Deployment (Production)

For operators running on Kubernetes, the official Helm charts deploy the validator node along with wallet and CNS UIs and wire up all the service interconnects automatically.

### Prerequisites Check

```bash
kubectl version --client
helm version
```

### Create a Dedicated Namespace

```bash
kubectl create namespace canton-validator
kubectl config set-context --current --namespace=canton-validator
```

### Create Kubernetes Secrets

```bash
# OIDC credentials
kubectl create secret generic oidc-credentials \
  --from-literal=clientId="YOUR_CLIENT_ID" \
  --from-literal=clientSecret="YOUR_CLIENT_SECRET" \
  -n canton-validator

# Docker registry credentials (for pulling private images)
kubectl create secret docker-registry canton-registry \
  --docker-server=ghcr.io \
  --docker-username=YOUR_GITHUB_USER \
  --docker-password=YOUR_GITHUB_PAT \
  -n canton-validator
```

### Configure Your Helm Values File

Create a `validator-values.yaml` with your deployment-specific config:

```yaml
# validator-values.yaml

validator:
  name: "acmeCorp-primaryValidator-1"
  onboardingSecret: "YOUR_ONBOARDING_SECRET"
  migrationId: 9
  sponsorSvUrl: "https://sv.sv-1.dev.global.canton.network.sync.global"

  auth:
    oidcAuthority: "https://YOUR_TENANT.auth0.com/"
    clientId: "YOUR_CLIENT_ID"
    ledgerApiAudience: "https://canton.network.global"
    validatorApiAudience: "https://canton.network.global"
    walletAdminUser: "auth0|your-admin-user-sub"

  participant:
    adminApiPort: 5002
    ledgerApiPort: 5001

  topUp:
    enabled: true
    targetThroughput: 2048       # bytes/sec; adjust based on expected traffic
    minTopupIntervalSeconds: 60

  healthProbes:
    enabled: true

postgresql:
  host: "your-postgres-host.rds.amazonaws.com"
  port: 5432
  database: "canton_validator"
  username: "canton_user"
  existingSecret: "postgres-credentials"

ingress:
  enabled: true
  className: "nginx"
  walletHostname: "wallet.mycompany.com"
  cnsHostname: "cns.mycompany.com"
  tls: true
```

### Deploy the Helm Chart

```bash
helm upgrade --install canton-validator ./helm/validator \
  -f validator-values.yaml \
  -n canton-validator \
  --wait \
  --timeout 10m
```

### Check Pod Status

```bash
kubectl get pods -n canton-validator

# Expected output (all pods should be Running):
# NAME                                     READY   STATUS    RESTARTS   AGE
# participant-xxxxxxxxx-xxxxx              1/1     Running   0          5m
# validator-backend-xxxxxxxxx-xxxxx        1/1     Running   0          5m
# wallet-web-ui-xxxxxxxxx-xxxxx            1/1     Running   0          5m
# cns-web-ui-xxxxxxxxx-xxxxx               1/1     Running   0          5m
# postgres-xxxxxxxxx-xxxxx                 1/1     Running   0          5m
```

## Step 7: Verify Connectivity

Before declaring your node live, verify it can actually reach the Global Synchronizer.

### Check Your Egress IP

```bash
# Run this from your validator machine/cluster
curl -s https://api.ipify.org
```

Confirm this IP matches what you submitted to your sponsor for whitelisting.

### Test Scan Connectivity

```bash
# Install jq if not present
sudo apt-get install -y jq

# Test connectivity to SV Scan endpoints
(set -o pipefail
CURL='curl -fsS -m 5 --connect-timeout 5'
for url in $($CURL https://scan.sv-1.dev.global.canton.network.sync.global/api/scan/v0/scans \
  | jq -r '.scans[].scans[].publicUrl'); do
  echo -n "$url: "
  $CURL "$url/api/scan/version" | jq -r '.version'
done)
```

A healthy output looks like:

```
https://scan.sv-2.dev.global.canton.network.digitalasset.com: 0.3.6
https://scan.sv.dev.global.canton.network.tradeweb.com: 0.3.6
https://scan.sv-1.dev.global.canton.network.cumberland.io: 0.3.6
https://scan.sv-1.dev.global.canton.network.sync.global: 0.3.6
```

If lines return errors instead of version numbers, your IP has not been added to the allowlist yet, or the SV you're checking is momentarily unreachable. You need **at least 2/3 of SVs** reachable for your validator to function.

### Test Sequencer Connectivity

```bash
# Check that SV sequencer endpoints are reachable
curl -s https://sequencer-1.sv-1.dev.global.canton.network.sync.global/health
# Expected: { "status": "SERVING" }
```

### Get Console Access

For Docker Compose deployments, you can drop into the Canton Admin Console:

```bash
# Create a console config
cat > console.conf <<EOF
canton {
  remote-participants {
    participant {
      admin-api { port = 5002; address = participant }
      ledger-api { port = 5001; address = participant }
    }
  }
  features.enable-preview-commands = yes
  features.enable-testing-commands = yes
  features.enable-repair-commands = yes
}
EOF

# Launch the console
docker run -it --rm \
  --network splice-validator \
  -v $(pwd)/console.conf:/app/app.conf \
  ghcr.io/digital-asset/decentralized-canton-sync/docker/canton:latest \
  --console
```

For Kubernetes, use a debug pod:

```bash
POD_NAME=$(kubectl get pods -n canton-validator -l app=participant -o name | head -1)
kubectl debug "${POD_NAME}" \
  --image "$(kubectl get pod "${POD_NAME}" -o json | jq -re '.spec.containers[0].image')" \
  -i -t -- bash
```

## Step 8: Access Your Wallet UI

Once the node is running and connected, the wallet UI is your primary interface for managing parties, canton coin balances, and validator operations.

**Docker Compose:** Navigate to `http://wallet.localhost` in your browser.

**Kubernetes:** Navigate to `https://wallet.mycompany.com` (or whatever hostname you configured in Helm values).

### First Login

On first access, you will be prompted to set a password via Keycloak. Save these credentials they are your validator operator credentials.

If you need to update the wallet user password later via Keycloak:

1. Log into the Keycloak admin console
2. Switch to the **validator** realm from the top-left dropdown
3. Navigate to **Users** → search for `<VALIDATOR_NAME>_walletuser`
4. Open **Credentials** tab → set new password → toggle **Temporary** to **OFF**
5. Click **Reset Password**

### Self-Feature on DevNet

On DevNet, you can self-feature your validator operator party to start receiving liveness rewards:

1. Log into the Wallet UI as the validator operator user
2. Tap to airdrop yourself **20 CC** (DevNet test coins)
3. In the validator settings, feature your operator party as the exchange party

On TestNet/MainNet, featuring requires SV approval contact your sponsor.

## Step 9: Promote to TestNet & MainNet

Once your validator is stable on DevNet, you can apply to connect to TestNet and eventually MainNet.

### TestNet Requirements

- Your validator must have been **approved for MainNet** by the Tokenomics Committee of the Global Synchronizer Foundation first TestNet approval is bundled with MainNet approval.
- Submit your application at [sync.global/validator-request](https://sync.global/validator-request)
- Provide a **separate static egress IP** for TestNet (distinct from your DevNet IP)
- Your sponsor will submit the IP whitelisting request to the SV operators

### MainNet

MainNet access is granted after the Tokenomics Committee review, which typically takes around **two weeks**. Once approved:

1. Your sponsor submits the MainNet IP to the SV operators for whitelisting
2. You receive a MainNet onboarding secret from your sponsor
3. Deploy a separate validator instance (do not reuse your DevNet/TestNet deployment) using the MainNet bundle and secret
4. Your node connects to the MainNet Global Synchronizer and begins earning **live Canton Coin rewards**

> 🔑 **Never reuse the same PostgreSQL database or Docker volumes across networks.** Each network (DevNet, TestNet, MainNet) must be a fully isolated deployment with its own persistent storage.

## Rewards, Traffic & Canton Coin

Understanding Canton Coin economics helps you configure your node correctly from day one.

### Liveness Rewards

Validators earn CC for simply keeping their nodes **online and connected** to the Global Synchronizer. There is no staking, no slashing, and no minimum uptime threshold but significant downtime means you miss out on rewards and may need to catch up on ledger state.

### Traffic Fees

Every transaction you submit to the Global Synchronizer consumes **network traffic**, which must be paid for in CC. The recommended starting configuration is:

```yaml
topUp:
  enabled: true
  targetThroughput: 2048     # 2 kB/s sufficient for ~1 tx/10 seconds
  minTopupIntervalSeconds: 60
```

This tells the validator backend to automatically purchase traffic using your operator party's CC balance when it falls below the calculated threshold. Adjust `targetThroughput` based on your actual transaction volume.

### Recommended Setup for Self-Funding Traffic

Use your **validator operator party** as both the exchange party and the traffic funding source. This creates a self-sustaining loop:

1. Node earns liveness rewards in CC → deposited to operator party
2. Auto-topup purchases traffic using that CC balance
3. Traffic is consumed by your transactions
4. More transactions → more usage → more rewards over time

## Backups & Disaster Recovery

Regular backups are not strictly required to keep the node running but they are critical for recovering your Canton Coin in the event of a catastrophic failure.

### What to Back Up

| Component | What It Contains | Priority |
|---|---|---|
| PostgreSQL databases | All ledger state, contract data, transaction history | Critical |
| Identities dump | Party keys and cryptographic identities | Critical |
| `.env` / Helm values | Configuration | High |
| Keycloak realm export | User credentials and OIDC config | High |

### Identities Backup (Most Important)

Super Validators retain enough information to help you recover your **Canton Coin** from an identities backup. They do **not** retain transaction details from non-SV applications. If you run third-party apps on your validator, only your own backups can recover that data.

For Docker Compose deployments:

```bash
# Trigger an identities dump via the wallet UI
# Settings → Backup → Create Dump → enter wallet user password

# Or via API
curl -X POST http://localhost:5003/api/validator/v0/admin/backup/identity \
  -H "Authorization: Bearer <ADMIN_TOKEN>" \
  -o identities-backup-$(date +%Y%m%d).json
```

Back up your PostgreSQL data with standard `pg_dump`:

```bash
docker exec canton-postgres pg_dump -U splice splice_validator \
  > backup-$(date +%Y%m%d-%H%M%S).sql
```

Store backups in a separate location from your validator host an S3 bucket, GCS bucket, or off-site storage.

## Monitoring & Observability

### Docker Compose (Basic)

The Docker Compose deployment does not include a monitoring stack. For basic health checks, use:

```bash
# Check all container health
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Follow validator backend logs
docker logs -f validator-backend

# Follow participant logs
docker logs -f participant
```

### Kubernetes (Full Stack)

The Kubernetes Helm deployment supports Grafana dashboards out of the box. Key metrics available:

- **Traffic usage** per party and per transaction type
- **CC balances** for local parties (operator party, exchange party, treasury party)
- **Ledger offset lag** how far behind the validator is from the tip of the chain
- **Container health** and resource utilization

Enable monitoring in your Helm values:

```yaml
monitoring:
  enabled: true
  grafana:
    enabled: true
    adminPassword: "your-grafana-password"
  prometheus:
    enabled: true
```

### Health Check Endpoints

```bash
# Validator backend health
curl http://localhost:5003/api/validator/v0/readyz

# Participant node health
curl http://localhost:5002/health

# Expected: HTTP 200 with {"status":"SERVING"} or similar
```

## Common Errors & Troubleshooting

### Node fails to connect to sequencer

**Symptom:** Logs show repeated connection failures to SV sequencer endpoints.

**Cause:** Your egress IP has not been added to the SV firewall allowlist, or you are running the check from a different IP than what you submitted.

**Fix:** Confirm your egress IP with `curl -s https://api.ipify.org` from the validator host, then ask your sponsor to verify whitelisting.

### Onboarding secret expired or already used

**Symptom:** Start script fails with `invalid onboarding secret` or `secret already consumed`.

**Cause:** Onboarding secrets are one-time use and expire after 48 hours (1 hour for self-generated DevNet secrets).

**Fix:** Request a new secret from your sponsor. On DevNet, re-run the `curl -X POST .../devnet/onboard/validator/prepare` call.

### Wallet UI shows "Unauthorized"

**Symptom:** Accessing `http://wallet.localhost` redirects to a login error.

**Cause:** OIDC configuration mismatch the `sub` in the issued JWT does not match the `WALLET_ADMIN_USER` configured in `.env`.

**Fix:** Check the `sub` field of the JWT your OIDC provider is issuing (you can decode it at jwt.io), and set `WALLET_ADMIN_USER` to exactly that value.

### Docker Compose containers keep restarting

**Symptom:** `docker ps` shows containers in a restart loop.

**Cause:** Usually a PostgreSQL initialization failure or a misconfigured environment variable.

**Fix:**

```bash
# Check logs for the specific failing container
docker logs participant --tail 100
docker logs validator-backend --tail 100

# Most common fix: wipe and re-initialize the database
./stop.sh
docker volume rm compose_postgres-splice
./start.sh -s "..." -o "NEW_SECRET" -p "..." -m "..."
```

> ⚠️ Wiping the volume means losing all existing data. Only do this on a fresh install or if you have a backup.

### Less than 2/3 of Scan endpoints reachable

**Symptom:** Connectivity check shows only 1-2 out of 10 SVs responding.

**Cause:** Your IP may be partially whitelisted, or some SVs are temporarily down.

**Fix:** Wait 10-15 minutes and re-run the connectivity check. If it persists, contact your sponsor to confirm full whitelisting across all SV operators.

## Summary

Setting up a Canton validator node follows a clear sequence: secure a sponsor, set a static egress IP, configure OIDC authentication, obtain an onboarding secret, deploy via Docker Compose or Kubernetes, verify sequencer connectivity, and access the wallet UI. From there you can apply to graduate to TestNet and MainNet, where your node becomes eligible to earn live Canton Coin liveness rewards.

The key operational principles are: keep your identities backed up, use a dedicated IP per network, never share database volumes across networks, and configure auto-topup so your traffic costs are funded automatically by your own rewards.

The software is fully open-source under the Splice project on GitHub. There is no slashing, no minimum stake, and no penalty for downtime though staying online maximizes your reward accrual and keeps your ledger state current.

**Canton Network Docs:** [docs.dev.sync.global](https://docs.dev.sync.global)

**Splice Source Code:** [github.com/hyperledger-labs/splice](https://github.com/hyperledger-labs/splice)

**Validator Application Form:** [sync.global/validator-request](https://sync.global/validator-request)

**Canton Foundation:** [canton.foundation/validators](https://canton.foundation/validators)

## About Us

At SC Audit Studio, we specialize in protocols security assessments.
Our team of experts has worked with companies like Aave, 1Inch and several more to conduct security assessments.
Partner with us to enhance your project's security and gain peace of mind.

[Reach out to us](https://x.com/SCAuditStudio) for queries and security assessments!

## Tags
["Canton Network", "Validator Node", "DeFi Infrastructure", "Daml", "Blockchain", "DevOps", "Docker", "Kubernetes", "Canton Coin"]

## FAQ
[
  {
    "question": "Do I need to stake Canton Coin to become a validator?",
    "answer": "No. Canton validators do not require staking. You simply need infrastructure that meets the hardware requirements and approval from a sponsor Super Validator. Rewards are earned purely through liveness, keeping your node online."
  },
  {
    "question": "What happens if my validator goes offline?",
    "answer": "There is no slashing. You will simply miss liveness rewards while offline and may need to catch up on ledger state when you come back online. Prolonged downtime can make the catch-up process slow, which is why production operators target high availability."
  },
  {
    "question": "Can I run validator nodes on multiple networks simultaneously?",
    "answer": "Yes, but each network (DevNet, TestNet, MainNet) requires a completely separate deployment with its own static egress IP, PostgreSQL instance, and Docker/Kubernetes environment. Sharing infrastructure between networks is not supported."
  },
  {
    "question": "How long does MainNet approval take?",
    "answer": "The Tokenomics Committee review typically takes around two weeks. Submit your application at [sync.global/validator-request](https://sync.global/validator-request) and your sponsor SV will be notified of the outcome."
  },
  {
    "question": "Is the Docker Compose setup suitable for production?",
    "answer": "It is acceptable for small production deployments and is the easiest path to get started. However, the official recommendation for serious production usage is the Kubernetes/Helm path because it includes monitoring, better scalability, and more robust restart handling than Docker Compose alone."
  },
  {
    "question": "What OIDC providers are officially supported?",
    "answer": "Auth0 has the most detailed official configuration guide in the Splice docs. Any OIDC-compliant provider that supports both Client Credentials Grant (for machine-to-machine) and Authorization Code Grant (for user-facing UIs) will work, including Keycloak, Azure AD, and Okta."
  }
]
