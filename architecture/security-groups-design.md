1) app-tier-sg
2) bastion-sg
3) database-sg
4) web-tier-sg




    sg-bastion
                      |
          SSH (22)    |    SSH (22)
                      |
        +-------------+-------------+
        |                                        |
        ↓                                	 ↓

   sg-web-tier                     sg-app-tier
        |                                         |
        | TCP 8080                  | TCP 3306
        |                                        |
        ↓                                       ↓

   sg-app-tier                  sg-database

Allowed References:

sg-bastion  → sg-web-tier      : SSH 22
sg-bastion  → sg-app-tier      : SSH 22
sg-bastion  → sg-database      : SSH 22

sg-web-tier → sg-app-tier      : TCP 8080

sg-app-tier → sg-database      : TCP 3306
