# Solution on how to create and deploy application into kubernetes cluster

### **Section One:  Build the application as a docker image**

In oder to build a docker image for our flask python application, we need to install some libraries and dependencies such docker runtime for our implementation. Since we use ubuntu as our Operating System, we can run the following command to install docker. 

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

After we have the docker runtime environment up and ready, we need to create a Dockerfile as shown in the repository to build the docker images.  I just name the docker image as **simple-flask-app** for this technical test. Run the following command in the directory of the source code, we can get a docker image for deployment.

**Step 4: Build a docker image**

```
docker build -t simple-flask-app .
```

We can run `docker images` command to check if we have successfully created the image. After we get the image, then we can test it on ubuntu server by running the following command.

**Step 5: Run the docker image on a bare linux system**

```
docker run —name=webapp -d -p 5000:5000 simple-flask-app:latest
```

We can then jump over to Chrome web browser and type in the link: http://localhost:5000 to see if the application has been deployed correctly. Below is a snapshot I captured from the web browser for the outcome of the running application.

![snapshot](https://tva1.sinaimg.cn/large/00831rSTly1gck4l01sufj30tv0btjrn.jpg)

**Step 6: Login into docker hub**

We can push the newly created docker image to the docker hub so that when we want to deploy the application into a container, we can easily download and run the image.

```
docker login --username markwu100 --password *******
```

**Step 7:push the docker image**

```
docker push markwu100/simple-flask-app:latest
```

By now, a docker image that could be deployed to the kubernetes cluster is ready in the docker hub repository. If you like, you can also use the below command to pull the image and deploy it on you local machine. 

```
docker pull markwu100/simple-flask-app
```



### **Section Two : Design of a container platform**

Since the obejective of this test is to create and deploy the web application in a reliable and scalable containerized platform, I intend to deploy the application by using kubernetes cluster.  Kubernetes is an open source software that allows us to deploy and manage containerized applications at scale. It also provides portability and extensibility, enabling us to scale containers seamlessly.  However, the cons of using kubernetes is that it usually takes a lot of time to deploy the kubernetes master and worker nodes. Therefore, in order to facilitate our work, we decide to turn to EKS service. 

Amazon EKS is a managed service that makes it easy for users to run Kubernetes on AWS without needing to stand up or maintain our own Kubernetes control plane. By using EKS, it could save us a lot of time in managing,updating and patching the control plane module.

#### **1. Architectural diagram on production-grade EKS cluster**

 Figure1 shows the diagram for deploying a kubernetes cluster in a production-grade environment as a reliable and stable service. The key points for designing this architecutre are as follows:  

![diagram](https://tva1.sinaimg.cn/large/00831rSTly1gcl2r7tcutj30oh0ehmyz.jpg)

	<center>Figure1: Architecture diagram on the entire kubernetes platform</center>

1. The entire network of the kubernetes platform is sitting on AWS VPC which spans two availability zones and  is divided into two subnets: public subnets and private subnets.  By setting up the service in two availability zones, the platform can provide low-latency performace to the end user and more reliability in case of one of the availability zones freezing. 
2. Public subnet is internet-facing while providing the environment where bastion host is serving. Engineers could only log into kubernetes worker nodes via SSH protocol on port 22 from bastion host to make sure the entire EKS cluster is  secured from the external access. The bastion host is also configured with the Kubernetes kubectl command line interface for managing the Kubernetes cluster.
3. Inside the public subnets, managed NAT gateways to allow outbound internet access for resources in the private subnets. If we need to patch the kubenetes cluster and modifiy the applicaiton running in the kubernetes pods, the internal request could access internet via the NAT Gataways.
4. The ALB is also hosting in the public subnet.  It exposes port 80 to the public and efficiently forward the traffic to the backend fleet of kubenetes worker nodes. Whenever a user is accessing the web application, user request will hit the ALB first. Then the ALB will then forward the http request to backend kubernetes workder nodes and the web applicaitons running inside kubernetes pods will then provide certain respond back to the end user. 
5. Private subnets host a group of kubernetes worker nodes. These worker nodes are also in a autoscaling group which makes the service more reliable and flexible. 

#### **2. Some important decisions**

In oder to further secure the entire architecture, we use a combination of AWS Security group and IAM user/IAM roles to provide fine-grained access control. 

**Security Group Consideration**

We create several security groups for ALB, bastion host and kubernetes worker nodes respectively. With regard to the SG for ALB,port 80 is accessible for all the source ip address.

The security group of worker nodes and the security group that handles the communication of the control plane with the worker nodes have been constructed in a way to avoid communication via the privileged ports in the worker nodes. 

**IAM Consideration**

IAM role is used to authenticate users to be admitted into kubernetes cluster. We use an mature aws-iam-authenticator tool to create and assign IAM Roles for kubernetes cluster. When users are admitted into the cluster, the native Kubernetes RBAC system will be responsible for interacting with Amazon EKS cluster’s Kubernetes API. 

### **Section Three: Deploy the application in EKS cluster**

Before we start our hands-on work to setup the kubernetes cluster and deploy our web application into EKS, **there are two prerequisites:** 

1. **AWS account access is ready**
2. **Python and pip3 are already installed** 

AWS has provided us an easy way to install EKS cluster by using eksctl tool. The following article describes the major steps to install eskctl and then spin up the kubernetes cluster with eksctl.  We will be albe to deploy the web application into kubernetes pods after the EKS cluster is up and running.

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

Kubernetes uses a command line utility called **kubectl** for communicating with the cluster API server. The following part describes how to install the kubectl tool on the bastion host.

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

In order to modify our kubectl configuration file to use it for authentication, we need to install aws-iam-authenticator. 

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

To deploy an app in Kubernetes, we need to apply a deployment file and a service file. I have created and uploaded the deployment.yaml file and service.yaml file to the code repository.  After copy these two files into the bastion host, we can run the following command to launch the deployment.

```
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

After we run `kubectl get svc` command, we can get the external-ip address of the web service which is the DNS name of the ALB.  Jump over to the Chrome web browser and type in the http link:http://a89e2e85b5f8c11eaa0f806c8c5284ab-996220484.ap-southeast-2.elb.amazonaws.com/,  it will show our hello world service.

![](https://tva1.sinaimg.cn/large/00831rSTly1gckej14zfkj30tq0dcta5.jpg)

By now, we have successfully created our web application in AWS EKS cluster:)



### **Section Four: Deliverables**

In the end of this article, I would like to attach serveral links where I  uploaded my code and docker images: 

**Github repository:**  https://github.com/a30001784/flask-web-app 

**Docker hub repository:** 	https://hub.docker.com/repository/docker/markwu100/simple-flask-app

**Command to pull the docker image:** `docker pull markwu100/simple-flask-app`