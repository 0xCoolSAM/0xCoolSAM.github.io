---
title: "Deploy Static Website on AWS"
classes: wide
header:
  teaser: /assets/images/devops/AWS/Gojo.jpg
ribbon: red
description: "I deployed a static webiste via S3 Bucket and then Distribute this Website via CloudFront."
categories:
  - DevOps
toc: false
---

Hi folks,

### I deployed a static webiste via S3 Bucket and then Distribute this Website via CloudFront.

## First: S3 Bucket
#### Create My S3 Bucket to serve webiste
![](/assets\images\devops\AWS\S3.jpg)

## Second: Upload Files
##### Upload my Website files to the Bucket
![](/assets\images\devops\AWS\mybucket.jpg)

## Third: Bucket Permissions
#### Edit my S3 Bucket Permissions as shown below
![](/assets\images\devops\AWS\bucketPerm.jpg)

## Fourth: Static Website
#### Enable Static Website Hosting in my S3 Bucket
![](/assets\images\devops\AWS\staticWebsite.jpg)

## Finally: CloudFront
#### Create CloudFront Distribution linked with my S3 Bucket
![](/assets\images\devops\AWS\CloudFront.jpg)

## Hola! Now our Webiste up and running
![](/assets\images\devops\AWS\OurWebsite.jpg)

### And here is the Domain Name for my Static Website: [https://d1xv1cy0gkgawo.cloudfront.net/](https://)

Thank you for reading ^_^ 
