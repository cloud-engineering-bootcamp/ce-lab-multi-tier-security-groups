# Configure Multi-Tier Security Groups Lab - Solution

**Student Name:** Adnan Nooruddin
**Date Completed:** 15.07.2026

---

# Environment Details

| Tier | Instance ID | Private IP | Public IP | Security Group |
|------|-------------|------------|-----------|----------------|
| Bastion | [i-0226a51e9528f785d] | [172.31.45.96] | [16.171.182.11] | [sg-bastion] |
| Web | [i-0caaa0a72b0d3ca1c] | [172.31.45.124] | [13.60.58.27] | [sg-web-tier] |
| App | [i-047884464a1210b8f] | [172.31.42.114] | [16.192.57.2] | [sg-app-tier] |
| Database | [i-037924631850eb4c7] | [172.31.36.213] | [13.60.171.49] | [sg-database] |

- **Region / VPC:** [eu-west-1 / vpc-xxxxxxxx]
- **Key Pair:** [bootcamp-week2-key.pem]
- **My IP (SSH source):** [x.x.x.x/32]

---

# Step 1: Create Security Groups

## Screenshot 1 – Security Groups List

Save your screenshot as:

```
screenshots/01-security-groups-list.png
```

![Security Groups List](screenshots/01-security-groups-list.png)

---

- [x] `sg-bastion` created
- [x] `sg-web-tier` created
- [x] `sg-app-tier` created
- [x] `sg-database` created
- [x] Rules reference other **security groups**, not IP ranges (except port 22 on the bastion and 80/443 on the web tier)

### sg-bastion

| Direction | Port | Source / Destination | Purpose |
|-----------|------|----------------------|---------|
| Inbound | [22] | [YOUR_IP/32] | [SSH from my laptop] |
| Outbound | [22] | [sg-web-tier, sg-app-tier, sg-database] | [Hop to private tiers] |

### sg-web-tier

| Direction | Port | Source / Destination | Purpose |
|-----------|------|----------------------|---------|
| Inbound | [80] | [0.0.0.0/0] | [Public HTTP] |
| Inbound | [443] | [0.0.0.0/0] | [Public HTTPS] |
| Inbound | [22] | [sg-bastion] | [SSH via bastion only] |
| Outbound | [8080] | [sg-app-tier] | [Proxy to app tier] |
| Outbound | [443] | [0.0.0.0/0] | [Package downloads] |

### sg-app-tier

| Direction | Port | Source / Destination | Purpose |
|-----------|------|----------------------|---------|
| Inbound | [8080] | [sg-web-tier] | [App traffic from web tier] |
| Inbound | [22] | [sg-bastion] | [SSH via bastion only] |
| Outbound | [3306] | [sg-database] | [MySQL queries] |
| Outbound | [443] | [0.0.0.0/0] | [Package downloads] |

### sg-database

| Direction | Port | Source / Destination | Purpose |
|-----------|------|----------------------|---------|
| Inbound | [3306] | [sg-app-tier] | [MySQL from app tier only] |
| Inbound | [22] | [sg-bastion] | [SSH via bastion only] |
| Outbound | [443] | [0.0.0.0/0] | [Package downloads] |

---

### Why reference a security group instead of a private IP or CIDR?

**Your Answer**

```
We reference a Security Group instead of a private IP or CIDR because Security Groups are dynamic. If an EC2 instance is replaced or its private IP changes, access continues to work without updating firewall rules. This is more secure, easier to manage, and follows the principle of least privilege by allowing only instances with the specified Security Group to connect.
```

---

# Step 2: Launch Instances and Connect Through the Bastion

## Screenshot 2 – Web Tier Security Group Rules

Save your screenshot as:

```
screenshots/02-web-tier-rules.png
```

![Web Tier Rules](screenshots/02-web-tier-rules.png)

---

- [x] 4 instances launched, one security group each
- [x] Connected to the bastion from my laptop
- [x] Reached web / app / db by **private IP** through the bastion

---

### Which connection method did you use?

- [x] ProxyJump
- [ ] Copy key to bastion

---

### My `~/.ssh/config` (or the commands I ran)

```text
Host bastion
    HostName 16.171.182.11
    User ec2-user
    IdentityFile ~/.ssh/bastion-key.pem

Host web
    HostName 172.31.45.124
    User ec2-user
    IdentityFile ~/.ssh/web-key.pem
    ProxyJump bastion

Host app
    HostName 172.31.42.114
    User ec2-user
    IdentityFile ~/.ssh/app-key.pem
    ProxyJump bastion

Host db
    HostName 172.31.36.213
    User ec2-user
    IdentityFile ~/.ssh/db-key.pem
    ProxyJump bastion

```

---

### Why must you target the private tiers by private IP instead of public IP?

**Your Answer**

```
Private tiers should be accessed using private IP addresses because they are intended to remain inside the VPC. Using private IPs keeps traffic on AWS's internal network, improves security, reduces exposure to the internet, can improve performance, and supports a layered architecture where only the web tier is publicly accessible.
```

---
# Step 3: Simulate Application Traffic

## Screenshot 3 – Services Running

Save your screenshot as:

```
screenshots/03-services-running.png
```

![Services Running](screenshots/03-services-running.png)

---

- [x] Database: `nc -l 3306` loop running (`nmap-ncat` installed)
- [x] App: `server.js` listening on 8080 (`nodejs` installed)
- [x] Web: Nginx installed, proxying `/` to `http://APP_PRIVATE_IP:8080`, enabled on boot

### Anything that did not start cleanly?

**Your Answer**

```

```

---

# Step 4: Test Traffic Flow

## Screenshot 4 – Traffic Flow Test

Save your screenshot as:

```
screenshots/04-traffic-flow-test.png
```

![Traffic Flow Test](screenshots/04-traffic-flow-test.png)

---

**Reading results:** *connection refused* = the security group **allowed** the packet but nothing was listening. *Timeout / hang* = the security group **blocked** the packet.

Tick the box if the test did what the **Expected** column says.

| # | Test | Run from | Expected | Got it |
|---|------|----------|----------|--------|
| 1 | `curl http://WEB_PUBLIC_IP` | Laptop | ✅ HTTP 200 | [x] |
| 2 | `ssh bastion`, then `ssh WEB_PRIVATE_IP` | Laptop → bastion | ✅ Connects | [ ] |
| 3 | `ssh ec2-user@WEB_PUBLIC_IP` | Laptop | ❌ Timeout | [x] |
| 4 | `nc -zv APP_PRIVATE_IP 8080` | Web | ✅ Succeeds | [x] |
| 5 | `nc -zv DB_PRIVATE_IP 3306` | App | ✅ Succeeds | [x] |
| 6 | `nc -zv DB_PRIVATE_IP 3306` | Web | ❌ Timeout | [x] |
| 7 | `curl http://WEB_PUBLIC_IP` | Laptop | ✅ "Hello from App Tier" | [x] |

---

### Test 6 — How did it fail?

(Timeout is the correct answer. "Connection refused" means the web tier can reach the database and a rule is too permissive.)

- [ ] Timed out / hung ✅ Correct
- [ ] Connection refused ⚠️ Rules too permissive

---

### Did every test pass on the first attempt?

- [ ] Yes
- [x] No

If no, what did you have to fix?

```
because of the wrong nginx config file
```

---

# Step 5: Architecture

## Screenshot 5 – Completed Architecture

Save your screenshot as:

```
screenshots/05-architecture-console.png
```

![Architecture Console](screenshots/05-architecture-console.png)

---

Confirm each communication path.

- [x] Internet → Web on port **80**
- [x] Web → App on port **8080**
- [x] App → Database on port **3306**
- [x] SSH access only through the bastion host
- [x] Web → Database blocked
- [x] Internet → App blocked
- [x] Internet → Database blocked

---

### Instances with no public IP

```
_______________________________________________________________
```

---

### Instances reachable from the Internet

```
_______________________________________________________________
```

---

# Bonus Challenges (Optional)

- [ ] **Challenge 1:** Remove the `443 → 0.0.0.0/0` egress rule from one tier. What broke?

```
_______________________________________________________________
```

- [ ] **Challenge 2:** Attach two security groups to one instance. How do the rules combine?

```
_______________________________________________________________
```

- [ ] **Challenge 3:** Tighten the bastion's inbound SSH to a single `/32` and re-test from another network.

```
_______________________________________________________________
```

- [ ] **Challenge 4:** Recreate the four security groups using the AWS CLI.

```
_______________________________________________________________
```

---

# Reflection

## 1. Why can the application tier reach the database, but the web tier cannot?

```
The database security group only allows inbound traffic on port 3306 from the
Application tier's security group. The Web tier is not included in this rule,
so it cannot connect directly to the database. This follows the principle of
least privilege and protects the database from unnecessary access.

```

---

## 2. Why do all private instances only accept SSH from the bastion?

```
The Bastion host is the only instance with a public IP that allows SSH from
trusted administrators. The Web, Application, and Database instances remain
private and only accept SSH from the Bastion's security group, reducing the
attack surface and improving security.
```

---

## 3. When a connection timed out instead of failing immediately, what was blocking it?

```
A connection timeout usually indicated that a Security Group or Network ACL
was blocking the traffic. The packets were dropped instead of rejected, so the
client waited until the connection attempt timed out.
```

---

## 4. Which security group rule would you never configure in a production environment, and why?

```
I would never allow SSH (port 22) from 0.0.0.0/0 in a production environment.
Doing so exposes the server to the entire internet and increases the risk of
unauthorized access and brute-force attacks. SSH should only be allowed from
trusted IP addresses or a Bastion host.
```

---

# Key Learnings

### Hardest part of this lab

```
The hardest part was configuring the Security Groups and Nginx reverse proxy
correctly and troubleshooting connectivity issues between the Web and
Application tiers.
```

---

### One thing I will do differently in future AWS deployments

```
I will plan the network architecture and Security Group rules before deploying
instances, use Security Group references instead of IP addresses where
possible, and verify connectivity between each tier step by step.
```

---

# Checklist

- [x] 4 security groups created with security group references
- [x] 4 EC2 instances launched
- [x] SSH works only through the bastion
- [x] Required services running on all tiers
- [x] All 7 connectivity tests completed
- [x] Blocked traffic confirmed with timeouts
- [x] End-to-end application test successful
- [x] Architecture documented
- [x] All required screenshots captured
- [x] Reflection questions completed
- [x] Changes committed to Git
- [x] Pull Request created


