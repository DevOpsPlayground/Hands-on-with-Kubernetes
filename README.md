### devopsplayground7-kubernetes
# Introduction to Kubernetes

## Overview
This page covers the required information for the DevOps Playground on kubernetes.
`Kubernetes is an open-source platform for automating deployment, scaling, and operations of application containers across clusters of hosts, providing container-centric infrastructure.`
It enables better control over your container ecosystem. Along this page we will deploy containers, duplicate them, load balance them and perform a rolling update on these container.  We will work with two versions of the same docker container `nginx:1.10` and `nginx:latest` (nginx:1.11.4). 

All the work will be performed on virtual machines you will be provided. Only a basic installlation of Kubernetes  exists on this image.

## Requirements

1. The IP of your dedicated box
2. A way to ssh into it (putty for windows) 
3. The Pem key to connect to the box

## Step 0 : Accessing the AWS instance
In your CLI, please enter : 
`ssh -i devops-playground.pem admin@<IP>`

## Step 1 : Installing **kubectl**

Altough the Kubernetes server is already installed on the box, we need to install the _client_ to communicate with it.
Here we simply download the binary, give it permission and move it where we can access it.
```
sudo wget https://storage.googleapis.com/kubernetes-release/release/v0.20.1/bin/linux/amd64/kubectl
sudo chmod u+x kubectl
sudo  mv kubectl /usr/local/bin/
```

To initialize the cluster : 

`sudo ./kube-up.sh` 
(custom script)

## Step 2 : Run our first container from CLI

Using the newly installed _kubectl_, let's run our first docker container: 

`sudo kubectl run nginx-old --image=nginx:1.10`

Here we download and run an instance of the version 1.10 of Nginx.
To verify the container is running inside the Kubernetes cluster : 

`sudo kubectl get pods`

Now, since this is a Nginx server running in the container, we should be able to curl it. 
First we need to get its IP : 

`sudo kubectl describe pod <your container name>`
Grab the IP from the output and :

`curl -I <IP>`
The output should look like this : 
```
root@ip-172-31-0-248:/home/admin# curl -I 10.0.0.34
HTTP/1.1 200 OK
Server: nginx/1.10.1
Date: Thu, 29 Sep 2016 14:24:43 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 31 May 2016 14:17:02 GMT
Connection: keep-alive
ETag: "574d9cde-264"
Accept-Ranges: bytes
```

notice the version of the nginx server : `Server: nginx/1.10.1`

## Step 3 : Scale and expose the containers

We currently have one container, in a real case we would have many, to showcase kubernetes capabilities let's extend our cluster. 
By running the command below, we will tell kubernetes that we want a total of 3 _nginx-old_ containers.

`sudo kubectl scale rc nginx-old --replicas=3`

To verify that the operation was successful, let's check the list of pods.
`sudo kubectl get pods`

you should see a total of three nginx-old-x containers.

How would we do to have curl access to all of these ? 
Well, instead of having to describe all the images one by one, we can simply tell kubernetes to expose the containers :

`sudo kubectl expose rc nginx-old --port=80`

You'll notince no IP in the output of this command. 
to get it you will need to query the kubernetes service : 

`sudo kubectl get svc`

the output should look like that :

```
root@ip-172-31-0-248:/home/admin# kubectl get svc
NAME         LABELS                                    SELECTOR         IP(S)       PORT(S)
kubernetes   component=apiserver,provider=kubernetes   <none>           10.0.0.1    443/TCP
nginx-base   run=nginx-base                            run=nginx-base   10.0.0.34   80/TCP
```

In my case i can curl _10.0.0.34_ 
`curl -I  10.0.0.34`


## Step 4 : Perform a rolling update

We've just realized that the nginx version we are using (1.10) is old and unsafe, and wish to upgrade to a later one. 
We choose to update to nginx:latest

let's try to run one: 

`sudo kubectl run nginx-latest --image=nginx:latest`

Once again if we want to curl it, we need to 

`sudo kubectl get pods`

`sudo kubectl get svc`

`curl -I <IP>`

This time, your output should be like :
```
root@ip-172-31-0-248:/home/admin# curl -I <ip>
HTTP/1.1 200 OK
Server: nginx/1.11.4
Date: Thu, 29 Sep 2016 14:18:48 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 13 Sep 2016 16:18:28 GMT
Connection: keep-alive
ETag: "57d826d4-264"
Accept-Ranges: bytes
```

if we are satifisfied with this container/version we can now do the upgrade.

Kubernetes has a `rolling-update` feature that enable a smooth transparent upgrade, with no downtime from one version to another. 
Kubernetes would simply replace the conatiners one by one, until they all are in the desired state.

Here, we want to replace the `nginx-old` with the latest version.

`sudo kubectl rolling update nginx-old --image=nginx:latest`

After a few minutes, the updat eshould be successful.

Now, to confirm it, if you try to curl again the "load balancer" you should get the new version :

`curl -I  <IP>`

If you can see the nginx version as 1.11.4, it was successful.
One thing to understand is that Kubernetes maintained the load balancer alive even after the upgrade.

## Step 3 : Create a  Nginx *pod* template file and run a container from it
nginx-pod.yml:
```
apiVersion: v1
kind: Pod
metadata:
 name: nginx-web
 labels:
   app: nginx-web
spec:
 containers:
   - name: nginx-web
     image: nginx
```
`sudo kubectl create -f nginx-pod.yml`

`sudo kubectl run nginx-web --image=nginx`


## Step 4 : Create a Tomcat *pod* template file
tomcat-pod.yml
```
apiVersion: v1
kind: Pod
metadata:
 name: tomcat
 labels:
   app: tomcat-web
spec:
 containers:
   - name: tomcat-web
     image: tomcat
```
`kubectl create -f tomcat-pod.yml`

`kubectl get pods`

## Step 5 : Do a Rolling Update of the whole pod
`sudo kubectl rolling-update nginx-web tomcat-web --image=tomcat-web`

`sudo kubectl get pods`


