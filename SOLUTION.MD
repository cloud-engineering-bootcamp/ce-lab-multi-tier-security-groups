Private tiers use private IP addresses to ensure internal VPC communication, enhance security, and prevent direct internet access.

# Configure Multi-Tier Security Groups Lab - Solution

**Student Name:** Hafiz Abdul Quddus
**Date Completed:** 19-07-2026

---

# Environment Details

| Tier     | Instance ID         | Private IP   | Public IP      | Security Group |
| -------- | ------------------- | ------------ | -------------- | -------------- |
| Bastion  | i-0dd872a8f7f56a6c9 | 172.31.17.52 | 100.55.63.226  | bastion-sg     |
| Web      | i-09ea566ebc4c002f7 | 172.31.13.68 | 100.53.98.27   | web-tier-sg    |
| App      | i-02ceb7f27addc7e4d | 172.31.18.56 | 54.234.106.103 | app-tier-sg    |
| Database | i-0c5a2a1320a1bbff9 | 172.31.20.44 | 54.144.50.151  | database-sg    |

- **Region / VPC:** eu-east-1 / vpc-07f871bexxxxxf
- **Key Pair:** bootcamp-week2-key.pem
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

- [X] `sg-bastion` created
- [X] `sg-web-tier` created
- [X] `sg-app-tier` created
- [X] `sg-database` created
- [X] Rules reference other **security groups**, not IP ranges (except port 22 on the bastion and 80/443 on the web tier)

### sg-bastion

| Direction | Port | Source / Destination                      | Purpose                                                |
| --------- | ---- | ----------------------------------------- | ------------------------------------------------------ |
| Inbound   | 22   | x.x.x.x/32                                | Allow SSH from my-pc                                   |
| Outbound  | 22   | [app-tier-sg, database-sg, web-tier-sg] | SSH (22) to app-tier-sg, database-sg and web-tier-sg |

### sg-web-tier

| Direction | Port | Source / Destination | Purpose                         |
| --------- | ---- | -------------------- | ------------------------------- |
| Inbound   | 80   | 0.0.0.0/0            | Allow HTTP from Internet        |
| Inbound   | 443  | 0.0.0.0/0            | Allow HTTPS from Internet       |
| Inbound   | 22   | bastion-sg           | SSH via bastion only            |
| Outbound  | 8080 | app-tier-sg          | Cutom TCP (8080) to app-tier-sg |
| Outbound  | 443  | [0.0.0.0/0]          | Allow HTTPS (443) to Internet   |

### sg-app-tier

| Direction | Port | Source / Destination | Purpose                                  |
| --------- | ---- | -------------------- | ---------------------------------------- |
| Inbound   | 8080 | web-tier-sg          | Allow custom TCP (8080) from web-tier-sg |
| Inbound   | 22   | bastion-sg           | Allow SSH (22) from bastion-sg           |
| Outbound  | 3306 | database-sg          | MySQL (3306) to database-sg              |
| Outbound  | 443  | 0.0.0.0/0            | HTTPS (443) to 0.0.0.0/0                 |

### sg-database

| Direction | Port | Source / Destination | Purpose                             |
| --------- | ---- | -------------------- | ----------------------------------- |
| Inbound   | 3306 | app-tier-sg          | Allow MySQL (3306) from app-tier-sg |
| Inbound   | [22] | bastion-sg           | Allow SSH (22) from bastion-sg      |
| Outbound  | 443  | 0.0.0.0/0            | HTTPS to Internet                   |

---

### Why reference a security group instead of a private IP or CIDR?

**Your Answer**

```
Security group references provide dynamic and secure access control. They automatically permit traffic only from instances 
in the specified security group, eliminating the need to manage changing private IP addresses or CIDR ranges.
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

- [X] 4 instances launched, one security group each
- [X] Connected to the bastion from my laptop
- [X] Reached web / app / db by **private IP** through the bastion

---

### Which connection method did you use?

- [X] ProxyJump
- [ ] Copy key to bastion

---

### My `~/.ssh/config` (or the commands I ran)

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

---

### Why must you target the private tiers by private IP instead of public IP?

**Your Answer**

```
Private tiers use private IP addresses to ensure internal VPC communication, enhance security, 
and prevent direct internet access.
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

- [X] Database: `nc -l 3306` loop running (`nmap-ncat` installed)
- [X] App: `server.js` listening on 8080 (`nodejs` installed)
- [X] Web: Nginx installed, proxying `/` to `http://APP_PRIVATE_IP:8080`, enabled on boot

### Anything that did not start cleanly?

**Your Answer**

```
No major issues. The only problem was a malformed curl command caused by hidden Unicode characters, 
which was resolved by typing the command manually.
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

| # | Test                                         | Run from          | Expected                 | Got it                                                 |
| - | -------------------------------------------- | ----------------- | ------------------------ | ------------------------------------------------------ |
| 1 | `curl http://WEB_PUBLIC_IP`                | Laptop            | ✅ HTTP 200              | HTTP 200 (OK and Successful) (`Hello from App Tier`) |
| 2 | `ssh bastion`, then `ssh WEB_PRIVATE_IP` | Laptop → bastion | ✅ Connects              | [Connects (using ssh agent authentication)             |
| 3 | `ssh ec2-user@WEB_PUBLIC_IP`               | Laptop            | ❌ Timeout               | Timeout (Connection time out)                          |
| 4 | `nc -zv APP_PRIVATE_IP 8080`               | Web               | ✅ Succeeds              | Successed (Connection with Ncat)                       |
| 5 | `nc -zv DB_PRIVATE_IP 3306`                | App               | ✅ Succeeds              | Successed (Connection with Ncat)                       |
| 6 | `nc -zv DB_PRIVATE_IP 3306`                | Web               | ❌ Timeout               | Timeout (Ncat: TIMEOUT)                                |
| 7 | `curl http://WEB_PUBLIC_IP`                | Laptop            | ✅ "Hello from App Tier" | "Hello from App Tier)"                                 |

---

### Test 6 — How did it fail?

(Timeout is the correct answer. "Connection refused" means the web tier can reach the database and a rule is too permissive.)

- [X] Timed out / hung ✅ Correct
- [ ] Connection refused ⚠️ Rules too permissive

---

### Did every test pass on the first attempt?

- [ ] Yes
- [X] No

If no, what did you have to fix?

```
No. I fixed Nginx configuration issues, replaced the placeholder app IP with the correct private IP, 
resolved duplicate Node.js port usage, and updated the Nginx default server configuration to enable proper proxying.
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

- [X] Internet → Web on port **80**
- [X] Web → App on port **8080**
- [X] App → Database on port **3306**
- [X] SSH access only through the bastion host
- [X] Web → Database blocked
- [X] Internet → App blocked
- [X] Internet → Database blocked

---

### Instances with no public IP

```
- App Tier EC2 Instance (Private IP: 172.31.18.56)
  Access: Only through Bastion Host / Web Tier (as allowed by Security Groups)

- Database Tier EC2 Instance (Private IP: 172.31.20.44)
  Access: Only through Bastion Host / App Tier (as allowed by Security Groups)
```

---

### Instances reachable from the Internet

```
- Bastion Host EC2 Instance
  Access: SSH (Port 22) from MY_IP/32 only

- Web Tier EC2 Instance (Nginx)
  Access:
    HTTP  (Port 80)  from 0.0.0.0/0
    HTTPS (Port 443) from 0.0.0.0/0
```

---

# Bonus Challenges (Optional) : NA

- [ ] **Challenge 1:** Remove the `443 → 0.0.0.0/0` egress rule from one tier. What broke?

```
```

- [ ] **Challenge 2:** Attach two security groups to one instance. How do the rules combine?

```
```

- [ ] **Challenge 3:** Tighten the bastion's inbound SSH to a single `/32` and re-test from another network.

```
```

- [ ] **Challenge 4:** Recreate the four security groups using the AWS CLI.

```
```

---

# Reflection

## 1. Why can the application tier reach the database, but the web tier cannot?

```
Because the database security group allows traffic only from the app tier security group on port 3306.
```

---

## 2. Why do all private instances only accept SSH from the bastion?

```
To prevent direct internet access and provide a secure controlled entry point for administration.
```

---

## 3. When a connection timed out instead of failing immediately, what was blocking it?

```
The AWS Security Group rules were silently dropping unauthorized traffic.
```

---

## 4. Which security group rule would you never configure in a production environment, and why?

```
Allowing SSH (22) from 0.0.0.0/0 because it exposes servers to unauthorized access attempts.
```

---

# Key Learnings

### Hardest part of this lab

```
Configuring Nginx reverse proxy and troubleshooting security group connectivity between tiers.
```

---

### One thing I will do differently in future AWS deployments

```
I will design security groups and network access rules before launching instances.
```

---

# Checklist

- [X] 4 security groups created with security group references
- [X] 4 EC2 instances launched
- [X] SSH works only through the bastion
- [X] Required services running on all tiers
- [X] All 7 connectivity tests completed
- [X] Blocked traffic confirmed with timeouts
- [X] End-to-end application test successful
- [X] Architecture documented
- [X] All required screenshots captured
- [X] Reflection questions completed
- [X] Changes committed to Git
- [X] Pull Request created
