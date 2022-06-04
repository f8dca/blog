+++
title = "Running OSTicket on Azure #1"
slug = "OSticket1"
date = "2020-01-03"
description = "Osticket setup and configuration for the entrprise."
tags = ["Support" , "OSTicket", "Technology", "Azure"]
+++

I recently deployed and configured the excellent [OSTicket](https://osticket.com/) support ticketing system at my work. This was a fun project where I learned a lot that warrants a **series of articles**.   

1.  Architecture: Setting up OSticket for speed and reliability in Azure  
2.  Prepare the database: Cost, performance and security considerations
3.  Prepare the web server: Azure app service, startup script and dealing with cronjobs.  
4.  Deployment and slots : Leveraging Git and the Azure deployment slot model
5.  Configuration management: Environmental variables and the Azure key vault 
6.  Wishlist.
- - -

# Architecture: Setting up OSticket for speed and reliability in Azure

## Why not a single machine setup?
First thing first: OSticket perfectly runs on a small single server installation, on a RaspberryPI, on a cheap virtual machine or even on a free Kubernetes instance. If you are setting it up for yourself to learn, for personal projects or for a small team then you do not need more.  

In my case we needed to support 180+ internal users plus an unknown number of external users in a highly regulated industry with high reliability and low downtime.  

## Why SaaS ?
One of the lessons that I learned from our previous commercially licensed ticketing system was systems that per user licensing is evil in many levels:
  - It forces IT to constantly assign / revoke user licenses. Except sometimw IT does do not necessarily know all of those users because they might be external parties.
  - IT does not need to manage user level policies.
  - If user licenses cost money managers will be hesitant assigning everyone fearing cost. If not all user have access to the ticketing system, then they will circumvent the process and just pester IT using different channels which will create a shadow docket.
  - Full company wide access to the ticketing system eliminates politics and excuses. Same simple policy applies to everyone.  


*Cool ...  but if this is an open system and anybody can just create a ticket then isn't that a big security risk ?*

Yes, It is.  --  My company is in a highly regulated industry with a lot more regulatory, cyber-security and privacy requirements than and average company of the same size.  None of our systems are actually exposed to the internet in order to minimise our attack surface. We require all users to be either on our LAN or connect via VPN.

Using Azure SaaS we were able to completly isolate our OSticket instance from the rest of our network infrastructure. In the event of a breach the attackers would gain access to a single isolated server that has no link to the core infrastructure.

## Our architecture:

|Role | SaaS service                                                   | Monthly Cost|
|-----|----------------------------------------------------------------|-------------|
|DataBase Server | Azure Database for MySQL -Basic, 2 vCore(s), 50 GB  | ~ 78 CAD    |
|WEB Server| Azure App service plan - Linux - Premium V2 (P1v2: 1)     |~ 100 CAD    |

Monthly cost per user : 2 CAD
Monthly cost per user for our previous commercial ticketing system :10 USD

The following diagram shows OSticket is separated from the production environment:
```goat
+--------------------------------------------------------+
| Azure tennant                                          | 
| +---------------------------------------------------+  |
| | Azure Subscription                                |  |        
| | +--------------------+ +-------------------+      |  |
| | | Resource Group 1   | |Resource Group 2   |      |  |
| | |  +--------------+  | |                   |      |  |
| | |  |Web Server    |  | |                   |      |  |
| | |  +--------------+  | |                   |      |  |
| | |  +--------------+  | | Production servers|      |  |
| | |  |DB Server     |  | |                   |      |  |
| | |  +--------------+  | |                   |      |  |
| | +--------------------+ +-------------------+      |  |
| +---------------------------------------------------+  |
+--------------------------------------------------------+

```

