# Procedure for migrating DELIS Cloud platform services into the K3S environment of the FLUIDOMOS project

This procedure describes how the DELIS Cloud services migration was performed to obtain the service images, which were containerized and deployed in FLUIDOMS's cluster infrastructure.

The main steps of the migration process are the following:

1. How to build  the docker images  
2. Build multi-platform docker image (to enable services execution on Edge architecture)  
3. Migrating to Kubernetes  
4. Vemar Sas  repository 

## 1  **Build Docker  Image** 

A **Dockerfile** will be used, which is a text file containing the necessary instructions to create the Docker image.

### 1.1  **Preparing the environment**

Ensure that Docker is installed on the machine. Follow the specific installation instructions depending on the operating system. To check if Docker is installed, open a terminal and type:  
```bash
docker \--version
```
### 1.2  **Organizing the application files**

Create a well-structured directory architecture for the application.

### 1.3  **Creating the Dockerfile**

The Dockerfile describes how to create the application’s image. Below is a basic example for different types of Python applications: 

1. Create the Dockerfile in the root directory of the project.   
2. Write the following instructions:
```bash
   \# Use an official Python base image  
   FROM python:3.9  
     
   \# Set the working directory  
   WORKDIR /usr/src/app  
     
   \# Copy the dependency file  
   COPY requirements.txt ./  
     
   \# Install dependencies  
   RUN pip install \--no-cache-dir \-r requirements.txt  
     
   \# Copy the source files  
   COPY . .  
     
   \# Expose the port if necessary (e.g., 5000 for Flask)  
   EXPOSE 5000

   \# Command to start the application

   CMD \["python", "app.py"\]
```
For other platforms or languages: The Dockerfile varies depending on the language and framework used. Generally, the logic is:

* Choose a base image (e.g., Java, PHP, Go).  
* Install dependencies.  
* Copy the files.  
* Expose the necessary ports.  
* Define the startup command.

### 1.4 **Creating the .dockerignore file (optional)**

The **.dockerignore** file works similarly to **.gitignore**, excluding certain files or directories from the Docker build process. Create a **.dockerignore** file in the root directory of the project and add lines like:  
```bash
node\_modules  
\*.log  
.git  
.env
```

### 1.5 **Building the Docker image**

When the Dockerfile is ready, it is possible to build the image. Open a terminal in the main application directory and run the command:  
```bash
docker build \-t your-app-name .
```
Explanation:

* **\-t your-app-name**: Specifies the name of the image.  
* **.**: Indicates that Docker should use the Dockerfile and the build context in the current directory.

### 1.6 **Running the container**

After creating the image,  run a container based on that image with the following command:
```bash
docker run \-p 3000:3000 your-app-name
```

Explanation:

* **\-p 3000:3000**: Maps the container’s port 3000 to port 3000 of the host system (adjust the numbers according to the application).  
* **your-app-name**: The name of the Docker image.

### 1.7  **Verify the application is running**

Open a browser and navigate to: 

http://localhost:3000 (or the exposed port). 

The application should be running.

### 1.8  **Publishing the image to Docker Hub (optional)**

The image can be uploaded to Docker Hub to make it available to others. Log in from the terminal using the command:  
docker login

Tag the image:
```bash
docker tag your-app-name your-dockerhub-username/your-app-name
```
Push the image to Docker Hub:
```bash
docker push your-dockerhub-username/your-app-name
```
### 1.9 **Summary Chapter 1**

* Prepare the files and create a Dockerfile.  
* Use a suitable base image for application.  
* Install dependencies and copy source files.  
* Build the Docker image.  
* Run the app in a container.  
* Optionally publish to Docker Hub.

 ## 2 **Build multi-platform docker image (to enable services execution on Edge architecture)**

Creating Docker images that support both ARM and AMD (x86\_64) platforms requires a few steps to ensure compatibility across architectures. Using Docker Buildx, it is possible to create multi-architecture images that work on both ARM devices (like Raspberry Pi) and AMD/Intel processors.  
**Following the steps to create separate Docker images for ARM and AMD.**

### 2.1  **Create a Dockerfile**

First, it  needs a common Dockerfile that defines the application. This file should be written to be architecture-agnostic, without depending on a specific architecture. (See the example in the previous paragraph) This Dockerfile works on both ARM and AMD architectures, as long as the base image supports both architectures.

### 2.2  **Build the image separately for each architecture**

 **A. For AMD architecture (x86\_64):**

Working on an AMD machine, simply build the image with the standard command:  
```bash
docker build \-t your-app-name:amd64 .
``` 
Working on an ARM machine and want to force the build for AMD, specify the platform:  
```bash
docker build \--platform linux/amd64 \-t your-app-name:amd64 .
```

 **B. For ARM architecture (arm64):**

Using an ARM machine or want to build for ARM: Build the image for ARM:  
```
docker build \--platform linux/arm64 \-t your-app-name:arm64 .
```

### 2.3  **Use Docker Buildx to build multi-architecture images simultaneously**

Docker Buildx is a Docker extension that allows you to build images for multiple platforms at the same time. 
This is useful for creating multi-architecture images that can run on both ARM and AMD.

 **A. Enable Docker Buildx**

Docker Buildx is included in Docker Desktop and Docker for Linux, but it may need to be activated. To check if Buildx is active, run:
```bash
docker buildx version
```
With Docker Desktop or an updated Docker on Linux, Buildx will already be enabled. Otherwise, follow the installation instructions.

 **B. Create and use a multi-architecture builder**

Create a new builder that supports multi-architecture builds: 
```bash
docker buildx create \--name mybuilder \--use
```
Start the builder:  
```bash
docker buildx ls
```
Verify the platforms supported by the builder:  
```bash
docker buildx ls
```
This will show the platforms that the builder currently supports.

**C. Build the multi-architecture image**

Now that Buildx is set up, build the image for multiple architectures simultaneously.  
Run the `docker buildx build` command with the desired platforms: 
```
docker buildx build \--platform linux/amd64,linux/arm64 \-t your-username/your-app-name:latest \--push .
```

* **\--platform linux/amd64,linux/arm64**: Specifies the platforms desired to build the image for.  
* **\-t your-username/your-app-name**  
  : Tags the image (e.g., latest).  
* **\--push**: This directly uploads the image to a Docker registry (e.g., Docker Hub).

To build the image locally without pushing it to a registry, omit the **\--push** option:  
```bash
docker buildx build \--platform linux/amd64,linux/arm64 \-t your-username/your-app-name:latest \--output type=docker .
```
This command will allow the multi-architecture image available locally.

**D. Verify the multi-architecture image**

Once the image is built and pushed to Docker Hub or a private registry, verify that the image supports multiple architectures with the following command:  
```bash
docker buildx imagetools inspect your-username/your-app-name:latest
```
This command will display the different architectures supported by the image (something like  linux/amd64 and linux/arm64).

### 2.4  **Running the image on different platforms**

With a multi-architecture Docker image available on Docker Hub or another registry, It is possible to run the container on different platforms.

**A. On an ARM device (e.g., Raspberry Pi):**
```bash
docker run your-username/your-app-name:latest
```
 **B. On an AMD machine (e.g., a laptop or x86\_64 server):**
```bash
docker run your-username/your-app-name:latest
```
### 2.5  **Summary of steps with Docker Buildx:**

* Create an architecture-agnostic Dockerfile.  
* Enable Buildx with the `docker buildx create` command.  
* Build a multi-architecture image with `docker buildx build --platform linux/amd64,linux/arm64`.  
* Push the image to a registry with **\--push**.  
* Verify the multi-architecture image with `docker buildx imagetools inspect`.

With this approach, it is possible to create a single Docker image that works on both ARM and AMD platforms, optimizing application deployment across a wide range of devices.

## 3 **Migrating  To Kubernetes**

Migrating a Docker image for use in a Kubernetes environment requires several steps to integrate the image into a Kubernetes cluster. Below is a step-by-step guide on how to perform the migration.

### 3.1 **Push the Docker image to a registry**

For Kubernetes to access the Docker image, it must be uploaded to a container registry (such as Docker Hub or a private registry like registry.vemarsas.it).

##### **A. Using Docker Hub:**

If needed, log in to Docker Hub:  
```bash
docker login
```
Then tag the image:  
```bash
docker tag your-app-name your-dockerhub-username/your-app-name:v1
``` 
Finally, push the image to Docker Hub:  
```bash
docker push your-dockerhub-username/your-app-name:v1
```

##### **B. Using a private registry:**

Within Kubernetes, the repository of usable Docker images is the public Docker Hub. To use our private registry containing the **Delis** project images prepared for Kubernetes, it needs to integrate a secret into Kubernetes to be specified during image deployment. The following command was used to work with the private registry (**registry.vemarsas.it**) that contains the images (the_username and the_password are placeholder) :  
```bash
kubectl create secret docker-registry regcred \--docker-server=registry.vemarsas.it \--docker-username=the_username \--docker-password=the_password \--docker-email=monitoraggio@vemarsas.it  
```
During deployment, add the following lines to the **yaml** files:  
```yaml
imagePullSecrets:  
  name: regcred
```
The `kubectl` command to create the secret only needs to be run once within the control plane of the consumer cluster.

### 3.2 **Preparing the Kubernetes configuration files**

   Once the image is uploaded to a registry, it needs to create YAML manifests to manage the deployment of the application on Kubernetes. The essential components are:  
* **Deployment**: Defines the number of application replicas and the Docker image to use.  
* **Service**: Exposes the application within or outside the Kubernetes cluster by mapping ports.

  ##### **A. Create the Deployment**

  Create a YAML file, for example, **deployment.yaml**, with the following content:
```yaml
  apiVersion: apps/v1  
  kind: Deployment  
  metadata:  
    name: your-app-name  
  spec:  
    replicas: 3  \# Number of replicas  
    selector:  
      matchLabels:  
        app: your-app-name  
    template:  
      metadata:  
        labels:  
          app: your-app-name  
      spec:  
        containers:  
        \- name: your-app-name  
          image: your-dockerhub-username/your-app-name:v1  
          ports:  
          \- containerPort: 3000  \# Change if app uses a different port
```
  Explanation:  
* **replicas**: Specifies how many instances of the app intended to deploy.  
* **image**: The Docker image uploaded to the registry.  
* **containerPort**: The port exposed by the app inside the container (used previously in the Dockerfile).

  ##### **B. Create the Service**

  The Service maps the Kubernetes cluster’s ports to those exposed by the application. Create a file called **service.yaml** with this content:
```yaml
  apiVersion: v1  
  kind: Service  
  metadata:  
    name: your-app-name-service  
  spec:  
    type: LoadBalancer  \# Use "NodePort" if testing on minikube or a local cluster  
    ports:  
    \- port: 80  \# Port exposed by the service  
      targetPort: 3000  \# Port inside the container  
    selector:  
      app: your-app-name
```
    
  Explanation:  
* **type: LoadBalancer**: Distributes external traffic to the container. Use **NodePort** for local or test clusters.  
* **port**: The port accessible from outside.  
* **targetPort**: The port inside the container (as in the Dockerfile).

### 3.3 **Deploying to the Kubernetes cluster**

   Once the **deployment.yaml** and **service.yaml** files are created, it is ready to deploy the application to the Kubernetes cluster.

    **A. Connect to the Kubernetes cluster**

   Ensure it is connected to the correct Kubernetes cluster.

    **B. Apply the Deployment and the Service**

   Once connected to the cluster, apply the YAML manifests using the `kubectl` command:
```bash
   kubectl apply \-f deployment.yaml  
   kubectl apply \-f service.yaml
```
### 3.4 **Checking the Deployment status**

   To check if the deployment was successful and if the pods are running correctly, use the following commands:
```bash
   kubectl get deployments  
   kubectl get pods
```

   To verify the service and obtain the public IP address (if using a LoadBalancer type service):
```bash
   kubectl get services
```

### 3.5 **Summary of steps:**

* Push the Docker image to Docker Hub or a private registry.  
* Create the YAML manifests for the Deployment and the Service.  
* Apply the manifests to the Kubernetes cluster.  
* Verify the status of the pods and the service.  
* Access the application through the Kubernetes service IP.

## 4 **VEMARSAS Repository**

To use our private repository  containing the **Delis** project images, it is possible to reach it at the following link 

[**https://registry.vemarsas.it/home**](https://registry.vemarsas.it/home)

This registry contains the Delis services images for Fluidomos:


<p align="center">
    <img src="https://github.com/vemarsas/FLUIDOMOS/blob/main/resources/vemarsas-01.png"><br>
<p>
<p align="center">
    <img src="https://github.com/vemarsas/FLUIDOMOS/blob/main/resources/vemarsas-02a.png"><br>
<p>
<p align="center">
    <img src="https://github.com/vemarsas/FLUIDOMOS/blob/main/resources/vemarsas-02b.png"><br>
<p>
<p align="center">
    <img src="https://github.com/vemarsas/FLUIDOMOS/blob/main/resources/vemarsas-02c.png"><br>
<p>
<p align="center">
    <img src="https://github.com/vemarsas/FLUIDOMOS/blob/main/resources/vemarsas-02d.png"><br>
<p>
<p align="center">
    <img src="https://github.com/vemarsas/FLUIDOMOS/blob/main/resources/vemarsas-02e.png"><br>
<p>
<p align="center">
    <img src="https://github.com/vemarsas/FLUIDOMOS/blob/main/resources/vemarsas-02f.png"><br>
<p>
<p align="center">
    <img src="https://github.com/vemarsas/FLUIDOMOS/blob/main/resources/vemarsas-02g.png"><br>
<p>
<p align="center">
    <img src="https://github.com/vemarsas/FLUIDOMOS/blob/main/resources/vemarsas-02h.png"><br>
<p>
