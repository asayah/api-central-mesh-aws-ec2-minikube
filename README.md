
# Introduction

This page illustrate a way for deploying API Central Mesh Governance on AWS for demo purposes. there is multiple ways of installing kubernetes, in this tutorial we will be using minikube.  


# Prerequisites

## 1 - Create an ec2 instance on AWS

In this  tutorial we will use a ec2 instance to create an ec2 instance. 

 - On your AWS Console, go to EC2, then click on **Launch Instance**
 - Create an EC2 instance of type **Amazon Linux 2 AMI (HVM)**
 - Select at least **t2.xlarge** for the image size (yes, Istio needs resources) 
 - Select a key (or create it)  then launch the instance (save the key locally, it is important for the rest of the tutorial). 
 - Go back to EC2, on the list of the instances, identify the instance have just created, in its description, identify the **security group** attached to it and update it to **allow all inbound traffic** on port 443.  
 ![list of ec2 instances, identify security group in the instance description](http://i68.tinypic.com/a88yf.png)

![edit security group](http://i65.tinypic.com/2e1cd8n.png)

![allow all inbound traffic](http://i67.tinypic.com/2cikadi.png)

## 2 - Setup the dynamic DNS

 - Create a free account for a dynamic DNS provider  https://www.noip.com/ and create a free dns. in my example I created a host asayah.ddns.net
 -  Connect to use ec2 instance, the list of instruction to do that are described when you click on **Connect** button while selecting the ec2 instance. 
![enter image description here](http://i65.tinypic.com/4hr50y.png)
⚠️ You will have to use **PuTTY** if you are on windows.
 - Then setup the free DNS on you console 
`
sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
sudo yum install -y noip
sudo noip2 -C
sudo systemctl enable noip.service
sudo systemctl start noip.service
`


