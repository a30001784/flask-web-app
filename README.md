# Brief explanation on how to create and deploy application on Kubernetes cluster

### Section One:  Build the application as a docker image

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

After we have the docker runtime environment up and ready, we need create a Dockerfile as shown in the repository to build the docker images.  I just name the docker image as **simple-flask-app** for this technical test. Run the following command, we can get a docker image for deployment.

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

### Section Two : Design of a container platform



### Section Three: Deploy the application on EKS







