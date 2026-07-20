hqxx@xx::~/Ironhack/Week2/LM2_03/ce-lab-multi-tier-security-groups$ curl http://98.81.106.182~

~Hello from App Tier
hqxx@xx:~/Ironhack/Week2/LM2_03/ce-lab-multi-tier-security-groups$ ssh web
   ,     #_
   ~\_  ####_        Amazon Linux 2023
  ~~  \_#####
  ~~     \###|
  ~~       \#/ ___   https://aws.amazon.com/linux/amazon-linux-2023
   ~~       V~' '->
    ~~~         /
      ~~._.   _/
         _/ _/
       _/m/'
Last login: Mon Jul 20 10:16:08 2026 from 172.31.17.52
[ec2-user@ip-172-31-13-68 ~]$ nc -zv 172.31.18.56 8080
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Connected to 172.31.18.56:8080.
Ncat: 0 bytes sent, 0 bytes received in 0.01 seconds.
[ec2-user@ip-172-31-13-68 ~]$ exit
logout
Connection to 172.31.13.68 closed.
hqxx@xx:~/Ironhack/Week2/LM2_03/ce-lab-multi-tier-security-groups$ ssh app
   ,     #_
   ~\_  ####_        Amazon Linux 2023
  ~~  \_#####
  ~~     \###|
  ~~       \#/ ___   https://aws.amazon.com/linux/amazon-linux-2023
   ~~       V~' '->
    ~~~         /
      ~~._.   _/
         _/ _/
       _/m/'
Last login: Mon Jul 20 10:11:58 2026 from 172.31.17.52
[ec2-user@ip-172-31-18-56 ~]$ nc -zv 172.31.20.44 3306
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Connected to 172.31.20.44:3306.
Ncat: 0 bytes sent, 0 bytes received in 0.01 seconds.
[ec2-user@ip-172-31-18-56 ~]$ exit
logout
Connection to 172.31.18.56 closed.
hqxx@xx::~/Ironhack/Week2/LM2_03/ce-lab-multi-tier-security-groups$ ssh web
   ,     #_
   ~\_  ####_        Amazon Linux 2023
  ~~  \_#####
  ~~     \###|
  ~~       \#/ ___   https://aws.amazon.com/linux/amazon-linux-2023
   ~~       V~' '->
    ~~~         /
      ~~._.   _/
         _/ _/
       _/m/'
Last login: Mon Jul 20 10:19:53 2026 from 172.31.17.52
[ec2-user@ip-172-31-13-68 ~]$ nc -zv 172.31.20.44 3306
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: TIMEOUT.
[ec2-user@ip-172-31-13-68 ~]$
