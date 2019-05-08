
# Introduction

This page illustrate a way for deploying API Central Mesh Governance on AWS for demo purposes. there is multiple ways of installing kubernetes, in this tutorial we will be using minikube.  


# I Prerequisites

## 1 - Create an ec2 instance on AWS

In this  tutorial we will use a ec2 instance to create an ec2 instance. 

 - On your AWS Console, go to EC2, then click on **Launch Instance**
 - Create an EC2 instance of type **Amazon Linux 2 AMI (HVM)**
 - Select at least **t2.xlarge** for the image size (yes, Istio needs resources) 
 - Select a key (or create it)  then launch the instance (save the key locally, it is important for the rest of the tutorial). 
 - Go back to EC2, on the list of the instances, identify the instance have just created, in its description, identify the **security group** attached to it and update it to **allow all inbound traffic** on port 443.  
 ![list of ec2 instances, identify security group in the instance description](http://i68.tinypic.com/a88yf.png)
 - Edit the security group

![edit security group](http://i65.tinypic.com/2e1cd8n.png)

 - allow all traffic 
 
![allow all inbound traffic](http://i67.tinypic.com/2cikadi.png)

## 2 - Setup the dynamic DNS

 - Create a free account for a dynamic DNS provider  https://www.noip.com/ and create a free dns. in my example I created a host asayah.ddns.net
 -  Connect to use ec2 instance, the list of instruction to do that are described when you click on **Connect** button while selecting the ec2 instance. 
![enter image description here](http://i65.tinypic.com/4hr50y.png)
⚠️ You will have to use **PuTTY** if you are on windows.

 - Then setup the free DNS on you console  

 `sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm`
`sudo yum install -y noip`
`sudo noip2 -C `
`sudo systemctl enable noip.service`
`sudo systemctl start noip.service`

After few second your ec2 instance should be reachable using the dns host provided, you can verify that by doing and ssh again and using the host you provided instead of the generated hostname provided by ec2
For example: `ssh -i asayah.pem ec2-user@asayah.ddns.net`

## 3 - Installing docker/kubernetes/minikube
ssh into the ec2 instance and run this commands: 

**Login as root**
`sudo -i`

**Install Docker**
`yum update -y`
`amazon-linux-extras install docker`
`service docker start`

**Install kubctl**
```shell
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
```

`chmod +x ./kubectl`
`mv ./kubectl /usr/bin`

**Install minikube**
```shell
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 \
  && chmod +x minikube
```
`mv ./minikube /usr/bin`

**Install helm**
```Shell
curl -o helm.tar.gz -L https://storage.googleapis.com/kubernetes-helm/helm-v2.13.1-linux-amd64.tar.gz
```
`tar -xvf helm.tar.gz`
`mv linux-amd64/helm /usr/bin`

**Install socat**
`yum install socat -y`

## 4 - Start minikube

Start minikube with 8Go of ram
`minikube start --memory=8192 --cpus=4 --kubernetes-version=v1.13.0 --vm-driver=none`

# II APIC Mesh governance installation


