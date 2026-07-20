
    Internet
                            |
                            |
                    HTTP/HTTPS (80/443)
                            |
                            ↓
                 +----------------------+
                 |   Bastion Security   |
                 |       Group          |
                 +----------------------+
                 |                      |
                 | Bastion Host (EC2)   |
                 | Public Subnet        |
                 | SSH Port 22          |
                 | Source: YOUR_IP/32   |
                 +----------------------+
                            |
                            |
                     SSH Port 22
                  (Private IP Access)

|                                                     |
| --------------------------------------------------- |
|                                                     |
| ↓                      ↓                       ↓ |

    +----------------------+
                 |   Web Tier EC2       |
                 |  (Nginx Server)      |
                 | Private Subnet       |
                 | 172.31.13.68         |
                 +----------------------+
                 | Security Group:      |
                 | sg-web-tier          |
                 |                      |
                 | INBOUND:             |
                 | 80  <- Internet      |
                 | 443 <- Internet      |
                 | 22  <- Bastion SG    |
                 |                      |
                 | OUTBOUND:            |
                 | 8080 -> App Tier SG  |
                 +----------------------+
                            |
                            |
                    TCP Port 8080
                (Only Web → App Allowed)
                            |
                            ↓

    +----------------------+
                 |   App Tier EC2       |
                 |   Node.js Server     |
                 | Private Subnet       |
                 | 172.31.18.56         |
                 +----------------------+
                 | Security Group:      |
                 | sg-app-tier          |
                 |                      |
                 | INBOUND:             |
                 | 8080 <- Web Tier SG  |
                 | 22   <- Bastion SG   |
                 |                      |
                 | OUTBOUND:            |
                 | 3306 -> Database SG  |
                 +----------------------+
                            |
                            |
                    TCP Port 3306
              (Only App → Database Allowed)
                            |
                            ↓

    +----------------------+
                 |   Database Tier EC2  |
                 |  Simulated MySQL     |
                 |      (Netcat)        |
                 | Private Subnet       |
                 | 172.31.20.44         |
                 +----------------------+
                 | Security Group:      |
                 | sg-database          |
                 |                      |
                 | INBOUND:             |
                 | 3306 <- App Tier SG  |
                 | 22   <- Bastion SG   |
                 |                      |
                 | OUTBOUND:            |
                 | HTTPS 443 Internet   |
                 +----------------------+

    SECURITY FLOW SUMMARY
                 ====================

Internet
   |
   | HTTP :80 / HTTPS :443
   ↓
Web Tier (Nginx)
   |
   | TCP :8080
   ↓
App Tier (Node.js)
   |
   | TCP :3306
   ↓
Database Tier (Netcat Simulation)

SSH ADMIN ACCESS FLOW
=====================

Your Laptop
     |
     | SSH :22
     | Source YOUR_IP/32
     ↓
Bastion Host
     |
     | SSH :22
     | Private IP only
     ↓
Web / App / Database Instances

BLOCKED TRAFFIC EXAMPLES
========================

❌ Internet → App Tier :8080
   Blocked by sg-app-tier

❌ Internet → Database :3306
   Blocked by sg-database

❌ Web Tier → Database :3306
   Blocked because database accepts traffic
   only from sg-app-tier

❌ App Tier → Web Tier :80
   Not allowed by security group rules
