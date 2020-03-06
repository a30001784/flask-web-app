# Brief explanation on how to create and deploy application on Kubernetes cluster

### **Section One:  Build the application as a docker image**

In oder to build a docker image for out flask python application, we need to install some libraries and dependencies such a docker runtime for our implementation. Since we use ubuntu as our Operating System, I can use following command to install docker. 

**Step 1: Update Software Repositories**

```
sudo apt-get update
```

**Step 2: Install Docker**

```
sudo apt install docker.io
```

**Step 3: Start and Automate Docker**

```
sudo systemctl start docker
sudo systemctl enable docker
```

After we have the docker runtime environment up and ready, we need to create a Dockerfile as shown in the repository to build the docker images.  I just name the docker image as **simple-flask-app** for this technical test. Run the following command, we can get a docker image for deployment.

**Step 4: Build a docker image**

```
docker build -t simple-flask-app .
```

We can test the docker image on the ubuntu server by running the following command.

**Step 5: Run the docker image on a bare linux system**

```
docker run —name=webapp -d -p 5000:5000 simple-flask-app:latest
```

We can then jump over over to Chrome web browser and type in the link: http://localhost:5000 to see if the application has been deployed correctly. Below is a snapshot I captured from the web browser.

![snapshot](https://tva1.sinaimg.cn/large/00831rSTly1gck4l01sufj30tv0btjrn.jpg)

**Step 6: Login into docker hub**

```
docker login --username markwu100 --password *******
```

**Step 7:push the docker image**

```
docker push markwu100/simple-flask-app:latest
```

By now a docker image that could be deployed to the kubernetes cluster has been ready in the docker hub repository.

### **Section Two : Design of a container platform**

Considering to improve the scalability and reliability of the web application, we prefer to deploy the app to a Kubernetes cluster on AWS. So in this part of the article, I would put forward a diagram on how to set up multi-tier secure and reliable EKS cluster on AWS.

#### **1. Architecture diagram on production-grade EKS cluster**

Figure1 is the diagram for deploying the kubernetes cluster in a production environment as a reliable and stable service. 

![Diagram3](https://tva1.sinaimg.cn/large/00831rSTly1gckcy31z3kj30oh0eiabw.jpg)

1. The entire network of the system is base on AWS VPC which spans two availability zone and  is seperate into two subnets: public subnets and private subnets. 
2. Public subnet is internet-facing and is the place where bastion host is serving. Engineer could only log into the kubernetes worker nodes via ssh protocol(port 22) from bastion host so that the eks cluster is  secured from the external access. The bastion host is also configured with the Kubernetes kubectl command line interface for managing the Kubernetes cluster.
3. Inside the public subnets, managed NAT gateways to allow outbound internet access for resources in the private subnets.
4. The ALB is also hosting in the public subnet. It expose port 80 to the public and efficiently forward the traffic to the backend kubenetes worker nodes.
5. Private subnets host a group of kubernetes worker nodes. These worker nodes are also in a autoscaling group which make the service more reliable and flexible. 

#### **2. Some important decisions**

In oder to further secure the entire architecture, we use a combination of AWS Security group, IAM user/IAM roles to provide fine-grained access control. 

**Security Group consideration**

We create several security groups for ALB, bastion host and kubernetes worker nodes respectively. With regard to the SG for ALB,port 80 is accessible for all the source ip address.

The security group of worker nodes and the security group that handles the communication of the control plane with the worker nodes have been constructed in a way to avoid communication via the privileged ports in the worker nodes. 

#### **3. Major interactions and traffic flows**





### **Section Three: Deploy the application in EKS cluster**

Before we start our hands-on work to deploy the kubernetes cluster and deploy our web application into EKS, there are two prerequisite: 

1. **AWS account access**
2. **Python and pip3 are already installed on your ubuntu server**

AWS has provided us an easy way to install EKS Cluster by using eksctl tool. The following article describes the major steps to install eskctl and then spin up the kubernetes cluster with eksctl. And then it will help you to deploy the web application in the EKS cluster. 

**Step 1: Install the Latest AWS CLI**

Log into one of the bastion host and run the following commands one by one.

```
pip install awscli --upgrade --user
```

**Step 2: Configure AWS credential**

```
aws configure
```

**Step 3: Install eksctl**

```
curl --silent --location"https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
```

```
sudo mv /tmp/eksctl /usr/local/bin
```

```
eksctl version
```

**Step 4: Install kubectl**

```
curl -o kubectl https://amazon-eks.s3-us-west-2.amazonaws.com/1.14.6/2019-08-22/bin/linux/amd64/kubectl
```

```
chmod +x ./kubectl
```

```
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
```

```
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
```

```
kubectl version --short --client
```

**Step 5: Install aws-iam-authenticator**

```
curl -o aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.14.6/2019-08-22/bin/linux/amd64/aws-iam-authenticator
```

```
chmod +x ./aws-iam-authenticator
```

```
mkdir -p $HOME/bin && cp ./aws-iam-authenticator $HOME/bin/aws-iam-authenticator && export PATH=$PATH:$HOME/bin
```

```
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
```

```
aws-iam-authenticator help
```

**Step 6: Create a EKS Cluster**

By running the following command, you can easily spin up an kubernetes cluster on AWS with 2 worker nodes in the private subnet.

```
eksctl create cluster --name prod --version 1.14 --region ap-southeast-2 --nodegroup-name standard-workers --node-type t2.medium --nodes 2 --nodes-min 1 --nodes-max 2 --ssh-access --ssh-public-key postgre --node-private-networking
```

**Step 7: Create a kubeconfig for Amazon EKS**

After the eks cluster is up and running, use the AWS CLI **update-kubeconfig** command to create or update your kubeconfig for your cluster.

```
aws sts get-caller-identity
```

```
aws eks --region ap-southeast-2 update-kubeconfig --name prod
```

Test the configuration, we can get the initial output for our eks cluster

```
kubectl get svc
```

```
NAME             TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
svc/kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   1m
```

**Step 8: Deploy the application in EKS cluster**

To deploy an app in Kubernetes, you need to deploy a deployment file and a service file. I have created and uploaded the deployment.yaml file and service.yaml file.  After copy these two files into the bastion host, we can run the following command to launch the deployment.

```
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

After we run `kubectl get svc` command, we can get the external-ip address of the web service which is the DNS name of the ALB.  Jump over to the Chrome web browser and type in the http link below,  it will show our hello world service.

![](https://tva1.sinaimg.cn/large/00831rSTly1gckej14zfkj30tq0dcta5.jpg)

By now, we have successfully created our web application in AWS EKS.



### **Section Four: Deliverables**

In the end of this article, I would like to attach serveral links where I  uploaded my code and docker images: 

**Github repository:**  https://github.com/a30001784/flask-web-app 

**Docker hub repository:** 	https://hub.docker.com/repository/docker/markwu100/simple-flask-app

**Command to pull the docker image:** `docker pull markwu100/simple-flask-app`