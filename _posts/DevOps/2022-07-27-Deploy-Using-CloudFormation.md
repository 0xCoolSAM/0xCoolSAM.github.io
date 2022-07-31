---
title: "Deploy High-Availability WebApp Using CloudFormation"
classes: wide
header:
  teaser: /assets/images/devops/AWS/Gojo1.jpg
ribbon: red
description: "I deployed a High-Availability WebApp Using CloudFormation."
categories:
  - DevOps
toc: false
---

# Udacity *Advanced Cloud DevOps Nanodegree Program*
## 2- Deploy Infrastructure as Code (*IAC*)

## Project 2 : Deploy High-Availability WebApp Using CloudFormation

### Description
Create a Launch Configuration in order to deploy four servers, two located in each of your private subnets.  
The launch configuration will be used by an auto-scaling group.  
You'll need two vCPUs and at least 4GB of RAM. The Operating System to be used is Ubuntu 18.  
So, choose an Instance size and Machine Image (AMI) that best fits this spec.  
Be sure to allocate at least 10GB of disk space so that you don't run into issues.

### Architecture
![](/assets/images/devops/AWS/infrastructure-diagram-Lucidchart.png)

### Github Link for the Project Scripts: [https://bit.ly/3znLhpp](https://bit.ly/3znLhpp)

### How to run the project:

1. Network infrastructure stack Usage:

```shell
./create.sh netstackName network.yml network-parameters.json
```

2. After step 1 has been completed successfully, run the services infrastructure stack Usage:

```shell
./create.sh serverstackName server.yml server-parameters.json
 ```

 3. Updating the network infrastructure stack Usage:
 
```shell
./update.sh netstackName network.yml network-parameters.json
```
4. Updating the services infrastructure stack Usage:

```shell
./update.sh serverstackName server.yml server-parameters.json
```

# WebApp URL:
[http://serve-WebAp-1HESZ7MAZT598-158522272.us-east-1.elb.amazonaws.com](http://serve-WebAp-1HESZ7MAZT598-158522272.us-east-1.elb.amazonaws.com)

Thank you for reading ^_^ 
