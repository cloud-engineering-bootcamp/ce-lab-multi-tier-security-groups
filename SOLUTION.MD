# Configure Multi-Tier Security Groups Lab - Solution

**Student Name:** Pragash KMARAVEL 
**Date Completed:** 15-07-2026

---

## Environment Details

| Tier | Instance ID | Private IP | Public IP | Security Group |
|------|-------------|------------|-----------|----------------|
| Bastion | [i-05c1364023f152c9b] | [172.31.11.77] | [15.237.246.158 — redact if repo is public] | sg-0dff75e8358a07424 (bastion-sg)  |
| Web | [i-0f499a5f8422e4e91] | [172.31.1.168] | [x.x.x.x] | sg-0b7dbf300a79e6066 (web-tier-sg) |
| App | [i-0c26f8e4d26065d50] | [172.31.9.231] | [none] | sg-06ad7522e2717c29d (app-tier-sg) |
| Database | [i-0e1b531b6ce77b261] | [172.31.0.196] | [none] | sg-02d6f34870e5fd101 (database-sg)  |

- **Region / VPC:** [eu-west-3 / vpc-034e913d2933ce958]
- **Key Pair:** [bootcamp-week2-key.pem]
- **My IP (SSH source):** [81.220.52.58/32]

---

## Step 1: Create Security Groups

![Security Groups List](Screenshots/01-security-groups-list.png)

- [x] `sg-bastion` created
- [x] `sg-web-tier` created
- [x] `sg-app-tier` created
- [x] `sg-database` created
- [x] Rules reference other **security groups**, not IP ranges (except port 22 on the bastion and 80/443 on the web tier)

**sg-bastion**

| Direction | Port | Source / Destination | Purpose |
|-----------|------|----------------------|---------|
| Inbound | [22] | [YOUR_IP/32] | [SSH from my laptop] |
| Outbound | [22] | [sg-web-tier, sg-app-tier, sg-database] | [Hop to private tiers] |

**sg-web-tier**

| Direction | Port | Source / Destination | Purpose |
|-----------|------|----------------------|---------|
| Inbound | [80] | [0.0.0.0/0] | [Public HTTP] |
| Inbound | [443] | [0.0.0.0/0] | [Public HTTPS] |
| Inbound | [22] | [sg-bastion] | [SSH via bastion only] |
| Outbound | [8080] | [sg-app-tier] | [Proxy to app tier] |
| Outbound | [443] | [0.0.0.0/0] | [Package downloads] |

**sg-app-tier**

| Direction | Port | Source / Destination | Purpose |
|-----------|------|----------------------|---------|
| Inbound | [8080] | [sg-web-tier] | [App traffic from web tier] |
| Inbound | [22] | [sg-bastion] | [SSH via bastion only] |
| Outbound | [3306] | [sg-database] | [MySQL queries] |
| Outbound | [443] | [0.0.0.0/0] | [Package downloads] |

**sg-database**

| Direction | Port | Source / Destination | Purpose |
|-----------|------|----------------------|---------|
| Inbound | [3306] | [sg-app-tier] | [MySQL from app tier only] |
| Inbound | [22] | [sg-bastion] | [SSH via bastion only] |
| Outbound | [443] | [0.0.0.0/0] | [Package downloads] |

**Why reference a security group instead of a private IP or CIDR?**

Referencing a security group means the rule automatically applies to any instance that's a member of that group regardless of what IP address it has.

---

## Step 2: Launch Instances and Connect Through the Bastion

![Web Tier Rules](Screenshots/02-web-tier-rules.png)

- [x] 4 instances launched, one security group each
- [x] Connected to the bastion from my laptop
- [x] Reached web / app / db by **private IP** through the bastion

**Which connection method did you use?** [ProxyJump ]

**My `~/.ssh/config` (or the commands I ran):**
```
Host bastion
    HostName 15.237.246.158
    User ec2-user
    IdentityFile ~/.ssh/bootcamp-week2-key.pem

Host web
    HostName 172.31.1.168
    User ec2-user
    IdentityFile ~/.ssh/bootcamp-week2-key.pem
    ProxyJump bastion

Host app
    HostName 172.31.9.231
    User ec2-user
    IdentityFile ~/.ssh/bootcamp-week2-key.pem
    ProxyJump bastion

Host db
    HostName 172.31.0.196
    User ec2-user
    IdentityFile ~/.ssh/bootcamp-week2-key.pem
    ProxyJump bastion
```

**Why must you target the private tiers by private IP, not public IP?**

Web, app, and database instances have no public IP — they were launched without one. Even if they did, their security groups only allow SSH from sg-bastion, not from the internet. So the only way in is through the bastion, over the private network, using each instance's private IP.

---

## Step 3: Simulate Application Traffic

- [x] Database: `nc -l 3306` loop running (`nmap-ncat` installed)
- [x] App: `server.js` listening on 8080 (`nodejs` installed)
- [x] Web: Nginx installed, proxying `/` to `http://APP_PRIVATE_IP:8080`, enabled on boot

**Anything that did not start cleanly?**

[Your answer, or "Nothing — all three services came up first try"]

Database, app, and web tier installs initially failed — those instances had no public IP, so no outbound internet access to reach the package repos, even though security groups allowed HTTPS out. I attached temporary Elastic IPs to all three to complete the installs.
---

## Step 4: Test Traffic Flow

![Traffic Flow Test](Screenshots/03-traffic-flow-test.png)

**Reading results:** *connection refused* = the security group **allowed** the packet but nothing was listening. *Timeout / hang* = the security group **blocked** it.

Tick the box if the test did what the "Expected" column says.

| # | Test | Run from | Expected | Got it |
|---|------|----------|----------|--------|
| 1 | `curl http://WEB_PUBLIC_IP` | Laptop | ✅ HTTP 200 | [x] |
| 2 | `ssh bastion`, then `ssh WEB_PRIVATE_IP` | Laptop → bastion | ✅ Connects | [x] |
| 3 | `ssh ec2-user@WEB_PUBLIC_IP` | Laptop | ❌ Timeout | [x] |
| 4 | `nc -zv APP_PRIVATE_IP 8080` | Web | ✅ Succeeds | [x] |
| 5 | `nc -zv DB_PRIVATE_IP 3306` | App | ✅ Succeeds | [x] |
| 6 | `nc -zv DB_PRIVATE_IP 3306` | Web | ❌ Timeout | [x] |
| 7 | `curl http://WEB_PUBLIC_IP` | Laptop | ✅ "Hello from App Tier" | [x] |

**Test 6 — how did it fail?** (Timeout is the correct answer. "Refused" means the web tier *can* reach the database and a rule is too open.)

- [x] Timed out / hung ✅ correct
- [ ] Connection refused ⚠️ rules too permissive

**Did every test pass first try?**

- [ ] Yes
- [x] No — what I had to fix:  installs failed initially since db/app/web had no public IP and no NAT gateway; I attached temporary Elastic IPs to complete installs, then all traffic tests passed first try.

---

## Step 5: Architecture

![Architecture in Console](Screenshots/04-architecture-console.png)

**Confirm each hop works and nothing else does:**

- [x] Internet → Web on port **80** (allowed by `sg-web-tier`: inbound 80 from `0.0.0.0/0`)
- [x] Web → App on port **8080** (allowed by `sg-app-tier`: inbound 8080 from `sg-web-tier`)
- [x] App → Database on port **3306** (allowed by `sg-database`: inbound 3306 from `sg-app-tier`)
- [x] Me → private tiers on port **22**, but only via the bastion (`sg-bastion` is the only source)
- [x] Web → Database is **blocked** (no rule allows it)
- [x] Internet → App / Database is **blocked** (no public route in)

**Instances with no public IP:** [app, db]

**Instances reachable from the internet:** [bastion (22 from my IP), web (80/443 from anywhere)]

---

## Bonus Challenges (Optional)

- [ ] **Challenge 1:** Remove the `443 → 0.0.0.0/0` egress rule from one tier — what broke? [Your answer]
- [ ] **Challenge 2:** Attach two security groups to one instance — how do the rules combine? [Your answer]
- [ ] **Challenge 3:** Tighten the bastion's inbound SSH to a single `/32` and re-test from a different network. [Your answer]
- [ ] **Challenge 4:** Reproduce the four groups with AWS CLI (`aws ec2 authorize-security-group-ingress`). [Your answer]

---

## Reflection

**1. Why can the app tier reach the database, but the web tier cannot?**

Because sg-database only allows inbound traffic from sg-app-tier, not sg-web-tier — enforcing that only the app tier can talk to the database.

**2. Why do all four instances only accept SSH from the bastion?**

So there's one single, tightly controlled entry point for SSH — reducing the attack surface instead of exposing every instance to the internet.

**3. When a test hung instead of failing fast, what was blocking it?**

The security group — it silently dropped the packet before it ever reached the instance, so the connection just timed out.

**4. Which single rule would you never write, and why?**

Allowing SSH (22) from 0.0.0.0/0 — it exposes every instance to brute-force attacks from anywhere on the internet

---

## Key Learnings

**Hardest part of the lab:**
[Your answer]

**One thing I'll do differently from now on:**
[Your answer]

---

## Checklist

- [x] 4 security groups created with SG-to-SG references
- [x] 4 instances launched, one per tier
- [x] SSH to private tiers works only through the bastion
- [x] Services listening on all three tiers (nc, Node.js, Nginx)
- [x] All 7 traffic tests run and documented
- [x] Blocked tests confirmed as **timeout**, not refused
- [x] End-to-end curl returns "Hello from App Tier"
- [x] Architecture diagram included
- [x] All screenshots captured
- [x] Reflection questions answered
- [x] Work committed to Git
- [x] Pull request created

---

**Completed By:** Pragash KUMARAVEL  
**Date:** 15-07-2026
