### devopsplayground7-kubernetes
# Introduction to Kubernetes

## Overview
This page covers the required information for the DevOps Playground on kubernetes.
`Kubernetes is an open-source platform for automating deployment, scaling, and operations of application containers across clusters of hosts, providing container-centric infrastructure. `
It enables better control over your container ecosystem. Along this page we will deploy containers, duplicate them, load balance them and perform a rolling update on these containers.  We will work with two versions of the same docker container `nginx:1.10` and `nginx:latest` (nginx:1.11.4). 

All the work will be performed on virtual machines you will be provided. Only a basic installation of Kubernetes exists on this image.

## Requirements

1. The IP of your dedicated box
2. A way to ssh into it (putty for windows) 
3. The Pem key to connect to the box

## Step 0: Accessing the AWS instance
Linux/Mac:
`ssh -i devops-playground.pem admin@<IP>`
Windows:
Use Putty and ask for help connecting.

## Step 1 : Installing **kubectl**

Although the Kubernetes server is already installed on the box, we need to install the _client_ to communicate with it.
Here we simply download the binary, give it permission and move it where we can access it.
```
sudo wget https://storage.googleapis.com/kubernetes-release/release/v0.20.1/bin/linux/amd64/kubectl
sudo chmod u+x kubectl
sudo  mv kubectl /usr/local/bin/
```

To initialize the cluster: 

`sudo ./kube-up.sh` 
(custom script)

## Step 2: Run our first container from CLI

Using the newly installed _kubectl_, let's run our first docker container: 

`sudo kubectl run nginx-old --image=nginx:1.10`

Here we download and run an instance of the version 1.10 of Nginx.
To verify the container is running inside the Kubernetes cluster: 

`sudo kubectl get pods`

Now, since this is a Nginx server running in the container, we should be able to curl it. 
First we need to get its IP: 

`sudo kubectl describe pod <your container name>`
Grab the IP from the output and:

`curl -I <IP>`
The output should look like this: 
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

notice the version of the nginx server: `Server: nginx/1.10.1`

## Step 3: Scale and expose the containers

We currently have one container, in a real case we would have many, to showcase kubernetes capabilities let's extend our cluster. 
By running the command below, we will tell kubernetes that we want a total of 3 _nginx-old_ containers.

`sudo kubectl scale rc nginx-old --replicas=3`

To verify that the operation was successful, let's check the list of pods.
`sudo kubectl get pods`

you should see a total of three nginx-old-x containers.

How would we do to have curl access to all of these? 
Well, instead of having to describe all the images one by one, we can simply tell kubernetes to expose the containers:

`sudo kubectl expose rc nginx-old --port=80`

You'll notice no IP in the output of this command. 
to get it you will need to query the kubernetes service: 

`sudo kubectl get svc`

the output should look like that:

```
root@ip-172-31-0-248:/home/admin# kubectl get svc
NAME         LABELS                                    SELECTOR         IP(S)       PORT(S)
kubernetes   component=apiserver,provider=kubernetes   <none>           10.0.0.1    443/TCP
nginx-base   run=nginx-base                            run=nginx-base   10.0.0.34   80/TCP
```

In my case i can curl _10.0.0.34_ 
`curl -I  10.0.0.34`


## Step 4: Perform a rolling update

We've just realized that the nginx version we are using (1.10) is old and unsafe, and wish to upgrade to a later one. 
We choose to update to nginx:latest

let's try to run one: 

`sudo kubectl run nginx-latest --image=nginx:latest`

Once again if we want to curl it, we need to 

`sudo kubectl get pods`

`sudo kubectl describe pod nginx-latest-x`

`curl -I <IP>`

This time, your output should be like:
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

If we are satisfied with this container/version, we can now do the upgrade.

Kubernetes has a `rolling-update` feature that enable a smooth transparent upgrade, with no downtime from one version to another. 
Kubernetes would simply replace the containers one by one, until they all are in the desired state.

Here, we want to replace the `nginx-old` with the latest version.

`sudo kubectl rolling-update nginx-old --image=nginx:latest`

After a few minutes, the update should be successful.

Now, to confirm it, if you try to curl again the "load balancer" you should get the new version:

`curl -I  <IP>`

If you can see the nginx version as 1.11.4, it was successful.
One thing to understand is that Kubernetes maintained the load balancer alive even after the upgrade.

## To go Further - Step 5 : Create a  Nginx  template file and run a containers from it
nginx-old.yml:
```
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    app: nginx
  template:
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.10
        ports:
        - containerPort: 80
```

This file sums up step 1-2, when created it will launch 3 instances of _nginx:1.10_
`sudo kubectl create -f nginx-old.yml`

let's verify all the containers are there :
`sudo kubectl get pods`


## To Go Further - Step 6 : Create another more up-to-date template file
nginx-new.yml
```
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-new
spec:
  replicas: 3
  selector:
    app: nginx-new
  template:
    metadata:
      name: nginx-new
      labels:
        app: nginx-new
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

This time we don't want to create it, we will directly do the rolling update.

`sudo kubectl rolling-update nginx -f nginx-new.yml`

Using this command, we tell kubernetes to replace all nginx containers by their newest version.


#Further Reading :

[Kubernetes Kubectl documentation](http://kubernetes.io/docs/user-guide/kubectl/)

