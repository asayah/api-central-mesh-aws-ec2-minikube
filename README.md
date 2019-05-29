

# Introduction

This page illustrate a way for deploying API Central Mesh Governance on AWS for demo purposes. there is multiple ways of installing kubernetes, in this tutorial we will be using minikube.  

⚠️  Disclaimer: installing minikube on AWS is not the best way of creating a  hosted kubernetes cluster( explore kops, GKE, EKS...) ,  the purpose of this tutorial is just to give an alternative. 

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

 ```Shell
sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
sudo yum install -y noip
sudo noip2 -C 
sudo systemctl enable noip.service
sudo systemctl start noip.service
```

After few second your ec2 instance should be reachable using the dns host provided, you can verify that by doing and ssh again and using the host you provided instead of the generated hostname provided by ec2
For example: `ssh -i asayah.pem ec2-user@asayah.ddns.net`

## 3 - Installing docker/kubernetes/minikube
ssh into the ec2 instance and run this commands: 

**Login as root**
```Shell
sudo -i
```

**Install Docker**
```Shell 
yum update -y
amazon-linux-extras install docker
service docker start
```

**Install kubctl**
```shell
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x ./kubectl
mv ./kubectl /usr/bin
```

**Install minikube**
```shell
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 \
  && chmod +x minikube
mv ./minikube /usr/bin
```

**Install helm**
```Shell
curl -o helm.tar.gz -L https://storage.googleapis.com/kubernetes-helm/helm-v2.13.1-linux-amd64.tar.gz
tar -xvf helm.tar.gz
mv linux-amd64/helm /usr/bin
```
Don't forget to run a helm init after starting your minikube. 

**Install socat**
```Shell
yum install socat -y
```

**Install certbot**
```Shell
sudo yum install certbot -y
```

## 4 - Start minikube

Start minikube with 8Go of ram
```Shell
minikube start --memory=8192 --cpus=4 --kubernetes-version=v1.13.0 --vm-driver=none
```

## 5 - Apicentral Mesh governance prerequisites 

 - Go on apicentral https://apicentral.axway.com
 - In the tab services Create an environment, use the host of the ec2 instance and port 443, usage should be publicly accessible.  
![create env](http://i65.tinypic.com/k4wi6f.png)

 - Download the zip containing the overrides.
 - transfer this zip in the /tmp folder of your ec2 instance using SCP
 In the following script change YOURHOST by your no ip host value.  
 ```Shell
 scp -i asayah.pem ******-override.zip  ec2-user@YOURHOST:/tmp
 ```

 - SSH again in your ec2 instance
In the following script change YOURHOST by your no ip host value.  
 ```Shell
 ssh -i yourkey.pem ec2-user@YOURHOST
 ```

⚠️ Run all the remaining commands in this tutorial using the root user
```Shell
 sudo -i 
```

 - Generate a certifcate for istio gateway. 
 Istio uses certificate to secure it gateway, as a prerequisite we have to create a kubernetes secret containing the certifcate. 
⚠️ first locate the name of the secret that you will have to use, by doing 
```Shell
cat istioOverride.yaml | grep secretName
```
In the following command, replace the YOURHOST by your host, and PUT_YOUR_SECRET_NAME by the secret name
```Shell
certbot certonly --standalone -d YOURHOST
kubectl create ns istio-system
kubectl create -n istio-system secret tls PUT_YOUR_SECRET_NAME --cert /etc/letsencrypt/live/YOURHOST/fullchain.pem --key /etc/letsencrypt/live/YOURHOST/privkey.pem -o yaml
```
Now lets create a namespace (apic-control) that will be used by APIC Agents, (Service Discovery Agent, Config Sync Agent)

```Shell
kubectl create namespace apic-control
```
Create RSA  keys  for the API Central agents  authentication
```Shell
mkdir -p /tmp/sda
mkdir -p /tmp/csa
openssl genpkey -algorithm RSA -out /tmp/sda/private_key.pem -pkeyopt rsa_keygen_bits:2048
openssl rsa -pubout -in /tmp/sda/private_key.pem -out /tmp/sda/public_key.der -outform der && base64 /tmp/sda/public_key.der > /tmp/sda/public_key
openssl genpkey -algorithm RSA -out /tmp/csa/private_key.pem -pkeyopt rsa_keygen_bits:2048
openssl rsa -pubout -in /tmp/csa/private_key.pem -out /tmp/csa/public_key.der -outform der && base64 /tmp/csa/public_key.der > /tmp/csa/public_key

```

Now lets create  kubernetes secrets to store them
```Shell
 kubectl create --namespace apic-control secret generic csa-secrets --from-file=publicKey=/tmp/csa/public_key --from-file=privateKey=/tmp/csa/private_key.pem --from-literal=password="" -o yaml
 kubectl create --namespace apic-control secret generic sda-secrets --from-file=publicKey=/tmp/sda/public_key --from-file=privateKey=/tmp/sda/private_key.pem --from-literal=password="" -o yaml
```
We can proceed now to the installation of Istio and the Agents.

# II APIC Mesh governance installation

 Lets extract the API Central overide files 
 ```Shell
 mv /tmp/******-override.zip ./  # replace ******-override.zip by the file you downloaded form API Central
 unzip /******-override.zip
 ```

To add Axway helm chart repo run this command: 
```Shell
helm repo add axway https://charts.axway.com/charts
```
Now we can install Istio: 

```Shell
helm upgrade --install --namespace istio-system istio axway-public/istio -f istioOverride.yaml
```
Its time to install APIC mesh agents
```Shell
helm upgrade --install --namespace apic-control apic-hybrid axway/apicentral-hybrid -f hybridOverride.yaml  --set observer.enabled=true --set observer.filebeat.sslVerification=none
```

Normally we would use nodeport if we want to use Minikube,  instead of modifing the installation lets just map the gateway internal port to 443 so we can acess it.

```Shell
kubectl get svc -n istio-system
apic-e4f77cf86a5b8349016a990e8f821b14-gateway   LoadBalancer   10.106.166.185   <pending>     443:30099/TCP                           23h
```
Identify the gateway port, in my case **30099**, and foward it.
run the cmd screen to keep the forwarding process alive
```Shell
screen
socat TCP-LISTEN:443,fork TCP:0.0.0.0:30099 # replace 30099 by your gateway port
```
Press Ctrl-A then Ctrl-D. This will **detach** your screen session but leave your processes running. You can now log out of the remote box.

Congrats your installed APIC Mesh Governance on a minikube on AWS, no you can go back on API Central, on the environment page, your services should be discoverded. 


