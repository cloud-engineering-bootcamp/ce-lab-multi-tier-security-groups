# Lab M2.03 - Configure Multi-Tier Security Groups

**Week 2 - Core Services & Secure Deployment**  
**Day 2 - Advanced Security and HTTP Access**

**Repository:** [https://github.com/cloud-engineering-bootcamp/ce-lab-multi-tier-security-groups](https://github.com/cloud-engineering-bootcamp/ce-lab-multi-tier-security-groups)

**Activity Type:** Individual (Mandatory)  
**Estimated Time:** 45-60 minutes

## Skills

- **AWS:** Security Groups
- **Security:** Defense in Depth
- **AWS:** Multi-Tier Architecture
- **Development Workflow:** Problem Solving

## Learning Objectives

- [ ] Create multiple security groups for different tiers
- [ ] Configure security group references
- [ ] Test traffic flow between tiers
- [ ] Implement defense-in-depth security
- [ ] Document security architecture

## Prerequisites

- [ ] EC2 instance from previous labs (or launch new ones)
- [ ] Understanding of security group concepts
- [ ] AWS CLI configured (optional but helpful)

---

## Introduction

Move beyond simple security groups to production-ready, layered security. You'll create a simulated 3-tier architecture with proper security isolation.

## Scenario

Your team is deploying a web application with these requirements:
- Public-facing web server (Nginx)
- Application server (Node.js on port 8080)
- Database server (simulated with netcat)
- Bastion host for secure SSH access

Your task: Design and implement security groups so only necessary traffic flows between tiers.

---

## Your Task

**What you'll create:**
- 4 security groups (web, app, database, bastion)
- 4 EC2 instances (bastion, web, app, database)
- Security group references between tiers
- Traffic flow tests and documentation

**Success criteria:**
- [ ] Web tier accessible from internet (port 80)
- [ ] Application tier accessible only from web tier
- [ ] Database tier accessible only from app tier
- [ ] SSH only via bastion host
- [ ] All traffic flows tested and verified

**Time limit:** 45-60 minutes

---

## Step-by-Step Instructions

### Step 1: Create Security Groups

Create 4 security groups with these configurations:

**sg-bastion:**
```
Inbound:
  - SSH (22) from YOUR_IP/32

Outbound:
  - SSH (22) to sg-web-tier
  - SSH (22) to sg-app-tier
  - SSH (22) to sg-database
```

**sg-web-tier:**
```
Inbound:
  - HTTP (80) from 0.0.0.0/0
  - HTTPS (443) from 0.0.0.0/0
  - SSH (22) from sg-bastion

Outbound:
  - Custom TCP (8080) to sg-app-tier
  - HTTPS (443) to 0.0.0.0/0
```

**sg-app-tier:**
```
Inbound:
  - Custom TCP (8080) from sg-web-tier
  - SSH (22) from sg-bastion

Outbound:
  - MySQL (3306) to sg-database
  - HTTPS (443) to 0.0.0.0/0
```

**sg-database:**
```
Inbound:
  - MySQL (3306) from sg-app-tier
  - SSH (22) from sg-bastion

Outbound:
  - HTTPS (443) to 0.0.0.0/0
```

**Create via console or CLI**

---

### 📸 Screenshot Required

**Filename**

```
screenshots/01-security-groups-list.png
```

**Capture**

Capture the **EC2 → Security Groups** page showing all four security groups:

- `sg-bastion`
- `sg-web-tier`
- `sg-app-tier`
- `sg-database`

Ensure all four security groups are visible in the list.

**Purpose**

This screenshot verifies that all required security groups have been successfully created before proceeding to launch the EC2 instances.

---

### Step 2: Launch Instances

Launch 4 instances, each with a single security group:
- Bastion instance: sg-bastion
- Web instance: sg-web-tier
- App instance: sg-app-tier
- DB instance: sg-database

#### Connect Through the Bastion

Only the bastion accepts SSH from your laptop. The web, app, and DB instances accept SSH **only from `sg-bastion`**, so every connection to them must go through it. Always target them by their **private IP** - security group references only match traffic inside the VPC.

Pick one of the two approaches below. You'll use it for the rest of the lab.

**Option 1: ProxyJump (key stays on your laptop)**

Add this to `~/.ssh/config`, replacing `YOUR_KEY.pem` with your key file:

```text
Host bastion
    HostName BASTION_PUBLIC_IP
    User ec2-user
    IdentityFile ~/.ssh/YOUR_KEY.pem

Host web
    HostName WEB_PRIVATE_IP
    User ec2-user
    IdentityFile ~/.ssh/YOUR_KEY.pem
    ProxyJump bastion

Host app
    HostName APP_PRIVATE_IP
    User ec2-user
    IdentityFile ~/.ssh/YOUR_KEY.pem
    ProxyJump bastion

Host db
    HostName DB_PRIVATE_IP
    User ec2-user
    IdentityFile ~/.ssh/YOUR_KEY.pem
    ProxyJump bastion
```

Now you can connect to any tier in one command - SSH hops through the bastion for you:

```bash
ssh bastion
ssh web
ssh app
ssh db
```

**One-off `-J` without editing the config:**

`ssh -J` does **not** pass your `-i` flag along to the jump host - the bastion hop would be offered no key at all, and you'd get `Permission denied (publickey)` from the *bastion* before the second hop is even attempted. Load the key into the **ssh-agent** instead, which both hops can reach:

```bash
# Start the agent and register it with your shell
eval "$(ssh-agent -s)"

# Hand your key to the agent (held in memory, not written to disk)
ssh-add ~/.ssh/YOUR_KEY.pem

# Now -J works: the agent supplies the key for BOTH hops
ssh -J ec2-user@BASTION_PUBLIC_IP ec2-user@APP_PRIVATE_IP
```

The agent lives in memory for this shell only - open a new terminal and you'll need to `ssh-add` again.

**Option 2: Copy the key to the bastion, then hop manually**

```bash
# Copy your private key to the bastion
scp -i ~/.ssh/YOUR_KEY.pem \
    ~/.ssh/YOUR_KEY.pem \
    ec2-user@BASTION_PUBLIC_IP:~/.ssh/

# SSH into the bastion
ssh -i ~/.ssh/YOUR_KEY.pem ec2-user@BASTION_PUBLIC_IP

# From the bastion, fix the key permissions (once)
chmod 400 ~/.ssh/YOUR_KEY.pem

# From the bastion, SSH to any tier by private IP
ssh -i ~/.ssh/YOUR_KEY.pem ec2-user@WEB_PRIVATE_IP
ssh -i ~/.ssh/YOUR_KEY.pem ec2-user@APP_PRIVATE_IP
ssh -i ~/.ssh/YOUR_KEY.pem ec2-user@DB_PRIVATE_IP
```

Option 2 is simpler to follow, but it puts a copy of your private key on a shared host - which is why Option 1 is what you'd use in production.

---

### 📸 Screenshot Required

**Filename**

```text
screenshots/02-web-tier-rules.png
```

**Capture**

Open the **Inbound Rules** for the **sg-web-tier** security group and ensure the following rules are visible:

- HTTP (80) from `0.0.0.0/0`
- HTTPS (443) from `0.0.0.0/0`
- SSH (22) from `sg-bastion`

**Purpose**

This screenshot verifies that the web tier has been configured with the correct inbound security rules before application traffic is tested.

---
### Step 3: Simulate Application Traffic

Before you can test whether traffic is *allowed*, each tier needs a service actually listening. Set these up first — otherwise every test in Step 4 fails no matter how correct your security groups are.

> **Note:** Amazon Linux 2023 is a minimal image - it ships with neither `nc` (netcat) nor Node.js. Each tier installs what it needs below. `nc` goes on **all three** instances: the database uses it to listen, and the web and app tiers use it for the connectivity tests in Step 4.

**On "database" instance:**
```bash
# netcat is not installed on Amazon Linux 2023
sudo dnf install -y nmap-ncat

# Simulate MySQL listening
while true; do
  echo "MySQL Connection Accepted" | nc -l 3306
done &
```

**On "app" instance:**
```bash
# Node.js and netcat are not installed on Amazon Linux 2023
sudo dnf install -y nodejs nmap-ncat

# Simulate app server
cat > server.js <<EOF
const http = require('http');
const server = http.createServer((req, res) => {
  res.writeHead(200);
  res.end('Hello from App Tier\\n');
});
server.listen(8080);
console.log('App listening on port 8080');
EOF

node server.js &
```

**On "web" instance:**
```bash
# Configure Nginx to proxy to app (netcat is needed for the Step 4 tests)
sudo dnf install -y nginx nmap-ncat

sudo tee /etc/nginx/conf.d/app.conf <<EOF
server {
    listen 80;
    location / {
        proxy_pass http://APP_INSTANCE_IP:8080;
    }
}
EOF

# Start Nginx now, and have it start automatically on every boot
sudo systemctl start nginx
sudo systemctl enable nginx
```

---

### 📸 Screenshot Required

**Filename**

```
screenshots/03-services-running.png
```

**Capture**

Capture terminal windows showing:

- Database instance listening on port **3306**
- App instance running the Node.js server on port **8080**
- Web instance with **Nginx** installed and running

You may use one combined screenshot or multiple terminal panes if everything is clearly visible.

**Purpose**

This screenshot verifies that each application tier is running the required service before testing the security groups.

---

### Step 4: Test Traffic Flow

Now that each tier is listening, the tests below actually exercise your security groups.

**Reading your results — this distinction is the whole point of the lab:**

| Result | What it means |
|--------|---------------|
| **Connection refused** | The security group **allowed** the packet, but nothing is listening on that port |
| **Timeout / hangs** | The security group **blocked** the packet - it never reached the host |

A "refused" on a test you expected to block means your rules are too permissive.

**Test 1: Web tier from internet**
```bash
# Should work
curl http://WEB_INSTANCE_PUBLIC_IP
```

**Test 2: SSH via bastion**
```bash
# Should work
ssh bastion-instance
ssh web-instance-private-ip  # From bastion
```

**Test 3: Direct SSH (should fail)**
```bash
# Should timeout/refuse
ssh web-instance-public-ip  # From your computer (not bastion)
```

**Test 4: App tier from web tier**
```bash
# SSH to web instance
ssh web-instance

# Test connection to app tier on port 8080
nc -zv app-instance 8080  # Should work
```

**Test 5: Database from app tier**
```bash
# SSH to app instance
ssh app-instance

# Test connection to database on port 3306
nc -zv db-instance 3306  # Should work
```

**Test 6: Database from web tier (should fail)**
```bash
# SSH to web instance
ssh web-instance

# Try to reach database directly
nc -zv db-instance 3306  # Should timeout (not allowed)
```

**Test 7: End-to-end through all tiers**
```bash
curl http://WEB_INSTANCE_PUBLIC_IP
# Should return: "Hello from App Tier"
```

---

### 📸 Screenshot Required

**Filename**

```
screenshots/04-traffic-flow-test.png
```

**Capture**

Capture your terminal showing evidence of the traffic flow tests, including:

- Successful `curl` request to the web tier
- Successful `nc` connection from the web tier to the app tier
- Successful `nc` connection from the app tier to the database tier
- At least one failed connection (timeout) demonstrating that unauthorized traffic was blocked by the security groups

**Purpose**

This screenshot demonstrates that the multi-tier security group configuration correctly allows authorized traffic while blocking unauthorized access.

---
## 📤 What to Submit

**Submission Type:** GitHub Repository

Create a **public** GitHub repository named `ce-lab-multi-tier-security`:

**Structure:**
```
ce-lab-multi-tier-security/
├── README.md
├── architecture/
│   ├── architecture-diagram.png
│   ├── security-groups-design.md
│   └── traffic-flow.md
├── security-groups/
│   ├── sg-bastion-rules.txt
│   ├── sg-web-tier-rules.txt
│   ├── sg-app-tier-rules.txt
│   └── sg-database-rules.txt
├── tests/
│   ├── security-test-results.md
│   └── test-commands.sh
└── screenshots/
    ├── 01-security-groups-list.png
    ├── 02-web-tier-rules.png
    ├── 03-services-running.png
    ├── 04-traffic-flow-test.png
    └── 05-architecture-console.png
```

---

### 📸 Screenshot Required

**Filename**

```
screenshots/05-architecture-console.png
```

**Capture**

Capture the **EC2 Instances** page showing:

- Bastion instance
- Web instance
- App instance
- Database instance

Ensure the **Security Groups** column is visible so it is clear which security group is attached to each instance.

**Purpose**

This screenshot documents the completed multi-tier architecture and verifies that each instance has been associated with the correct security group.

---

## Screenshot Checklist

Before submitting your lab, verify that the following screenshots are present in the `screenshots/` folder:

- [ ] `screenshots/01-security-groups-list.png`
- [ ] `screenshots/02-web-tier-rules.png`
- [ ] `screenshots/03-services-running.png`
- [ ] `screenshots/04-traffic-flow-test.png`
- [ ] `screenshots/05-architecture-console.png`

---

## Grading Rubric

| Criteria | Points |
|----------|--------|
| **Security Group Design** | 30 |
| **Configuration Correctness** | 25 |
| **Testing & Verification** | 20 |
| **Documentation** | 15 |
| **Architecture Diagram** | 10 |
| **Total** | **100** |

---

**Great work on implementing production-ready security!** 🛡️

