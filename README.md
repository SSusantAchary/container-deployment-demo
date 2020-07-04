# Containerizing & Deploying a Web App on the Cloud

This tutorial shows you how to containerize using Docker an existing Python Django application and manually deploy it on Google Cloud Platform.

Author: MV Karan | https://karan.be | mv at karan.be

---

### Prerequisites

This tutorial assumes you have a basic knowledge of the following:

- Linux OS & Commands
- Git
- Python



The required system/tools are:

- Debian-based OS (Like Ubuntu, above 16.04)
- Git

  - Installation & Setup Instructions: https://www.digitalocean.com/community/tutorials/how-to-install-git-on-ubuntu-18-04-quickstart



Accounts you would need:

- GitHub (https://github.com)
- DockerHub (https://hub.docker.com) 
- Google Cloud (https://cloud.google.com)

#### Previous Tutorial

This tutorial is a continuation to a [previous tutorial](https://github.com/mvkaran/webapp-deployment-demo) showing you how to deploy a Django app on the Cloud. This tutorial assumes that you have your own version of the django-dashboard-adminator repo on your GitHub account.



#### Demo Application

The demo application that will be used in this tutorial is https://github.com/app-generator/django-dashboard-adminator which is an open-source dashboard admin panel built using Django framework on Python



---



## Step-by-step Tutorial

Contents:

- [Containerizing & Deploying a Web App on the Cloud](#containerizing--deploying-a-web-app-on-the-cloud)
    - [Prerequisites](#prerequisites)
      - [Previous Tutorial](#previous-tutorial)
      - [Demo Application](#demo-application)
  - [Step-by-step Tutorial](#step-by-step-tutorial)
    - [1. Setting up Docker](#1-setting-up-docker)
    - [2. Setting up Dockerhub](#2-setting-up-dockerhub)
    - [3. Dockerizing the application](#3-dockerizing-the-application)
      - [A. Preparing the Dockerfiles](#a-preparing-the-dockerfiles)
      - [B. Preparing docker-compose files](#b-preparing-docker-compose-files)
      - [C. Building the images and running the containers](#c-building-the-images-and-running-the-containers)
    - [4. Deploying the application](#4-deploying-the-application)
      - [A. Commit to Git](#a-commit-to-git)
      - [B. Push images to Dockerhub](#b-push-images-to-dockerhub)
      - [C. Create cloud infrastructure on GCP](#c-create-cloud-infrastructure-on-gcp)
      - [D. Login to the VM instance](#d-login-to-the-vm-instance)
      - [E. Deploy and run the Docker images](#e-deploy-and-run-the-docker-images) 

---

### 1. Setting up Docker

Firstly, you would have to install Docker CE engine and Docker Compose on your development machine. The instructions for this are available on the documentation site of Docker. However, since we need to setup Docker on the cloud servers as well, it would be helpful to create a shell script with the installation commands.

Change your current working directory to the root of your django-dashboard-adminator repo on your development machine.

```bash
cd /path/to/your/django-dashboard-adminator
```

Open your favorite text editor and create a shell script named ``docker-install.sh`` 

```bash
nano docker-install.sh
```

In this file, let us add the installation commands for Docker CE engine and Docker Compose. We are assuming that the development machine and the cloud server will be running a Debian-based operating system like Ubuntu.

```bash
#!/bin/bash

# Update and upgrade sources and packages
sudo apt-get -y update
sudo apt-get -y upgrade

# Remove older versions of docker, if any
sudo apt-get -y remove docker docker-engine docker.io containerd runc

# Get some dependencies

sudo apt-get -y install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

# Add Docker’s official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

# Add docker's repo to apt sources
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

# Update sources
sudo apt-get -y update

# Install docker
sudo apt-get -y install docker-ce docker-ce-cli containerd.io

# Get docker-compose
sudo curl -L "https://github.com/docker/compose/releases/download/1.26.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# Add execute perms and group perms
sudo chmod +x /usr/local/bin/docker-compose
```



Save this file and execute the script to install Docker CE engine and Docker compose. This will install all the dependencies for Docker and Docker Compose.

```bash
bash docker-install.sh
```

_Note: The above script will upgrade some existing packages to newer versions, if available. Depending on your development environment, this can cause some of your other applications to break. If you want to skip this, remove the line ``sudo apt-get -y upgrade`` on Line 5 of the script._



Let's check whether Docker and Docker Compose have installed correctly, by running a version-check command on each of them individually.

```bash
docker --version
```

```bash
docker-compose --version
```



To test whether Docker engine is able to pull images and run containers properly, let's pull and run the 'hello-world' image.

```bash
docker run hello-world
```

If everything is setup well, you should see a Hello, World! text on your terminal.

---

### 2. Setting up Dockerhub

Dockerhub is one of the most popular and widely-used registries for Docker images. We will be using this to push, pull and deploy our images. 

But first, we need to create an account on Dockerhub. Go ahead and create an account on https://hub.docker.com

Once done, let's create a new repository to store our images.

1. Navigate to the [Create Repository](https://hub.docker.com/repository/create) page. 
2. Type a name for your repository. For our current application, let's just use  ``django-dashboard-adminator``
3. You can add a description if you like, or leave it blank.
4. Select the visibility as ``Public`` for now. You can change this to a private repository later, if you like.
5. Leave the ``Build Settings`` section as it is.
6. Click on ``Create``. This will create your repository

Your repository name will be in the format of ``<docker-id>/django-dashboard-adminator`` where ``<docker-id>`` is the ID you chose while creating your account on Docker Hub and ``django-dashboard-adminator`` is the name of the repository you created. Make a note of this repository format, since we will be using this later.



### 3. Dockerizing the application

---

#### A. Preparing the Dockerfiles

Now that we have Docker engine and Docker Compose setup, we can go ahead to containerize our application.

We will be using 2 containers for our application - one that will have the application source code, its dependencies and the Gunicorn server; and the other that will have an Nginx server that will act as a proxy that passes traffic to the Gunicorn server.

In order to keep the demo and deployment simple, we will be building the images for both the containers ourselves and package all code and configuration within it.

For this, we will be using two separate Dockerfiles - one for the app image and the other for the nginx image.

In the application's source code, you can see that there is already a ``Dockerfile`` that exists with the below instructions and arguments

```dockerfile
FROM python:3.6

ENV FLASK_APP run.py

COPY manage.py gunicorn-cfg.py requirements.txt .env ./
COPY app app
COPY authentication authentication
COPY core core

RUN pip install -r requirements.txt

RUN python manage.py makemigrations
RUN python manage.py migrate

EXPOSE 5005
CMD ["gunicorn", "--config", "gunicorn-cfg.py", "core.wsgi"]
```

Let's see what each of these instructions do and if we need to modify it for our application.



```dockerfile
FROM python:3.6
```

This indicates that the base image we will be using for the build will be the official python image with 3.6 version. We need to change this, since the ``python:3.6`` image is nearly 950 MB, which would lead to large image sizes and would impact the deployment speed. Instead, we will be using the ``python:3.6-slim`` image which is just around 150 MB and would be sufficient for our requirements. 

The modified statement would be this

```dockerfile
FROM python:3.6-slim
```

Next, we see that there is a ``ENV FLASK_APP run.py`` instruction. This would be unnecessary for our application, since our app uses Django and this instruction is intended for a Flask app. This line can be removed.

Following it are multiple lines of ``COPY`` instructions which copy the application source code from the working directory into the Docker image. We need to reorder some of these instructions for better layer caching and leaner intermediate containers. 

For this, we need to first copy only the ``requirements.txt`` file from our app source code and run a ``pip install``. This can be accomplished by the below couple of instructions.

```dockerfile
COPY requirements.txt ./
RUN pip install -r requirements.txt
```

Our Dockerfile will now look like this:

```dockerfile
FROM python:3.6-slim

COPY requirements.txt ./
RUN pip install -r requirements.txt

COPY manage.py gunicorn-cfg.py .env db.sqlite3 ./
COPY app app
COPY authentication authentication
COPY core core

RUN python manage.py makemigrations
RUN python manage.py migrate

EXPOSE 5005
CMD ["gunicorn", "--config", "gunicorn-cfg.py", "core.wsgi"]
```

 Notice that in the line ``COPY manage.py gunicorn-cfg.py .env db.sqlite3 ./`` , ``requirements.txt`` is removed since it was copied earlier itself. Also, our SQLite database file ``db.sqlite3`` was added.

_Note: Containers are meant to be stateless. Including the database file storage within a container defeats this purpose and also not an accepted good practice. However, we are doing this only for the sake of demonstration and illustration. In real use cases, the database engine can be containerized whereas the database file storage should be on a persistent storage device._

The next 4 ``COPY`` instructions copy all the necessary app source code into the container. Outside of the two changes mentioned above, we will leave this as it is.

The ``RUN`` instructions mentioned below initialise the database migrations and also apply them.

```dockerfile
RUN python manage.py makemigrations
RUN python manage.py migrate
```

To make this more efficient, we will combine these two instructions into a single one like below

```dockerfile
RUN python manage.py makemigrations && python manage.py migrate
```

Next, the ``EXPOSE`` instruction informs Docker that the container listens on port ``5005`` at runtime. This does not publish the port, which we will be doing later in Docker Compose. We will leave this instruction as it is.

The final instruction in the Dockerfile is the ``CMD`` instruction, which instructs Docker to execute the command in the arguments. In this case, it instructs Docker to start and run the Gunicorn server with the specified ``gunicorn-cfg.py`` configuration file. We will leave this as it is.

Now, your final Dockerfile should be this:

```dockerfile
FROM python:3.6-slim
COPY requirements.txt ./

RUN pip install -r requirements.txt

COPY manage.py gunicorn-cfg.py .env db.sqlite3 ./
COPY app app
COPY authentication authentication
COPY core core

RUN python manage.py makemigrations && python manage.py migrate

EXPOSE 5005
CMD ["gunicorn", "--config", "gunicorn-cfg.py", "core.wsgi"]
```

Finally, let us rename our dockerfile to ``Dockerfile-app`` to indicate that this is for our app image.

```bash
mv Dockerfile Dockerfile-app
```

Now, we have our Dockerfile for our app image ready. 

Next, we need to write a Dockerfile for our Nginx server. This does not exist, and we need to create it from scratch. Open up your favorite text editor.

```bash
nano Dockerfile-nginx
```

Paste the below 3 instructions into the Dockerfile and save it.

```dockerfile
FROM nginx:latest
RUN rm /etc/nginx/conf.d/default.conf
COPY ./nginx /etc/nginx/conf.d
```

This Dockerfile instructs Docker to use the latest official Nginx image as the base image, remove the existing default Nginx configuration, and copy our own configuration into it.



#### B. Preparing docker-compose files

We have our 2 Dockerfiles ready, which when built and run, will yield 2 Docker containers with our app and nginx server. In order to run these multiple containers, we will be using Docker Compose. With Docker Compose, we can use a YAML file to configure our application services and how they would interact with each other and the host.

In the app's repo, you can see that there is already a ``docker-compose.yml`` file that looks like this.

```yaml
version: '3'
services:
  appseed-app:
    restart: always
    env_file: .env
    build: .
    ports:
      - "5005:5005"
    networks:
      - db_network
      - web_network
  nginx:
    restart: always
    image: "nginx:latest"
    ports:
      - "85:85"
    volumes:
      - ./nginx:/etc/nginx/conf.d
    networks:
      - web_network
    depends_on: 
      - appseed-app
networks:
  db_network:
    driver: bridge
  web_network:
    driver: bridge
```

We need to make a few changes to this file to better suit our application.

Firstly, by the structure of the YAML file, we can see that there are 2 services, named ``appseed-app`` and ``nginx``, and 2 networks named ``db_network`` and ``web_network``. 

Let's look at the service definition for the ``appseed-app`` service, which is as below

```yaml
appseed-app:
    restart: always
    env_file: .env
    build: .
    ports:
      - "5005:5005"
    networks:
      - db_network
      - web_network
```

Rename the service from ``appseed-app`` to just ``app`` for the sake of simplicity.

``restart`` indicates the restart policy for the container. Since this is our app server which we want to be always available, we retain it as ``always``.

``env_file`` configuration states which file to use for environment variables. In this case, it is ``.env`` and we will retain it.

``build`` configuration states details needed to build an image for the container. Here, we see the value is ``.`` which indicates that the build context is the current directory (from where docker-compose.yml is being run) and the file to be used is the default ``Dockerfile``.

However, we have created a separate ``Dockerfile-app`` for our app container and we need to use that to build our image and run the container. Hence, modify the build settings to the below configuration:

```yaml
build:
	context: .
	dockerfile: Dockerfile-app
```

Once the image is built, we want to name it such that it can readily be pushed to Docker Hub. Hence, right after the ``build`` configuration, add an ``image`` configuration as below. After building the image using the mentioned Dockerfile-app, the image would be renamed to what we mention below.

```yaml
image: <docker-id>/django-dashboard-adminator:app-1.0
```

Replace ``<docker-id>`` with your own Docker ID. 

``ports`` configuration mentions the host and container port (and mapping) that needs to be published. Since we will be using our Nginx server as a reverse proxy to our Gunicorn server and we don't want to expose our app server directly to the host, we can remove the ``ports`` configuration.

``networks`` configuration mentions which networks should the container connect to. Since we are not using a separate container for our database engine, we wouldn't need a separate ``db_network`` and can use only the ``web_network`` for our app container and nginx container to talk to each other.

After all the above changes are made, the service would look like this:

```yaml
app:
    restart: always
    env_file: .env
    build:
      context: .
      dockerfile: Dockerfile-app
    image: <docker-id>/django-dashboard-adminator:app-1.0
    networks:
      - web_network
```

For our next ``nginx`` service, we have the below already in our docker-compose.yml file:

```yaml
nginx:
    restart: always
    image: "nginx:latest"
    ports:
      - "85:85"
    volumes:
      - ./nginx:/etc/nginx/conf.d
    networks:
      - web_network
    depends_on: 
      - appseed-app
```

Let's retain the ``restart`` policy. 

Since we are not using the default Nginx Docker image, but would be using our own image that we defined in ``Dockerfile-nginx`` , we need to replace this with the build settings and image name that we need, as follows:

```yaml
build:
    context: .
    dockerfile: Dockerfile-nginx
image: <docker-id>/django-dashboard-adminator:nginx-1.0
```

Change the ports that are published from ``85`` to ``80`` so that we can access our application easily through our DNS.

Since we have already packaged our configuration within our nginx image, we wouldn't need a volume mount for the nginx configuration. Hence, the ``volumes`` instruction can be removed.

We want our nginx server to talk to our app server privately, hence we want to include it as a part of the ``web_network`` that our ``app`` service also is a part of. Hence, we can leave the ``networks`` configuration as it is.

The ``depends_on`` configuration is used to mention the order in which services need to be brought up. Since we want our app server to come up before the nginx server, we need to mention this explicitly here. 

The final ``nginx`` service should look like this:

```yaml
nginx:
    restart: always
    build:
      context: .
      dockerfile: Dockerfile-nginx
    image: <docker-id>/django-dashboard-adminator:nginx-1.0
    ports:
      - "80:80"
    networks:
      - web_network
    depends_on: 
      - app
```

Lastly, in the ``networks`` configuration, remove the ``db_network`` since we are not using it anymore.

The final ``docker-compose.yml`` should look like this:

```yaml
version: '3'
services:
  app:
    restart: always
    env_file: .env
    build:
      context: .
      dockerfile: Dockerfile-app
    image: <docker-id>/django-dashboard-adminator:app-1.0
    networks:
      - web_network
  nginx:
    restart: always
    build:
      context: .
      dockerfile: Dockerfile-nginx
    image: <docker-id>/django-dashboard-adminator:nginx-1.0
    ports:
      - "80:80"
    networks:
      - web_network
    depends_on: 
      - app
networks:
  web_network:
    driver: bridge
```

Let us rename this docker-compose file:

```bash
mv docker-compose.yml docker-compose.dev.yml
```

We will be using this file in our dev environment to build the images and push the images to Dockerhub.

In our production environment (i.e, on the cloud server), we would have to just pull the image from Dockerhub and run it without having to rebuild again. For this, we would need to create a separate docker-compose file. 

Let's start by copying our existing docker-compose.yml file for the dev environment.

```bash
cp docker-compose.dev.yml docker-compose.prod.yml
```

Open up the ``docker-compose.prod.yml`` file in your favourite editor, and remove the ``build`` configuration in both the ``app`` and ``nginx`` services.

Now, the ``docker-compose.prod.yml`` should look like this:

```yaml
version: '3'
services:
  app:
    restart: always
    env_file: .env
    image: <docker-id>/django-dashboard-adminator:app-1.0
    networks:
      - web_network
  nginx:
    restart: always
    image: <docker-id>/django-dashboard-adminator:nginx-1.0
    ports:
      - "80:80"
    networks:
      - web_network
    depends_on: 
      - app
networks:
  web_network:
    driver: bridge
```

We are now ready to build and run our Docker containers.



#### C. Building the images and running the containers

In the terminal, let's bring up the containers using our docker-compose file. If the images we specified already don't exist, Docker will automatically build it for us. Run the below command in your terminal:

```bash
docker-compose -f docker-compose.dev.yml up
```

Navigate to ``localhost`` in your browser, and the app should now open up.



### 4. Deploying the application

---

We are now ready to deploy your application through Docker images to our server.

#### A. Commit to Git

Firstly, commit all the changes in your app and push it to your GitHub repo.

```bash
git add .
git commit -m "Dockerized app"
git push
```



#### B. Push images to Dockerhub

We need to push the Docker images that we have built into our Docker hub repository that we configured in [2. Setting up Dockerhub](#2-setting-up-dockerhub)

For this, we first need to login to our Dockerhub account through our terminal

```bash
docker login
```

This will ask for your username and password for your Dockerhub account. Type in your username and password.

Once successfully logged-in, we need to push our images. This can be done by docker-compose:

```bash
docker-compose -f docker-compose.dev.yml push
```

This will push both of our ``app`` and ``nginx`` Docker images to Dockerhub, with the relevant tags, under our repository. This will take some time since the image sizes could be in a few hundreds of MBs.

Once the push has been complete, open up Dockerhub, navigate to your repository and check that both the ``app-1.0`` and ``nginx-1.0`` tags have been pushed to the repository.



#### C. Create cloud infrastructure on GCP

Firstly, create an account on Google Cloud Platform by heading over to https://cloud.google.com, if you haven't already done so. You will get a certain amount of free credits to start with.

Next, create a project named ``Django Dashboard Adminator`` .

After creating a project, let's create a VM Instance on Google Compute Engine and prepare it to deploy our application.

1. From the navigation bar in the left, under ``Compute Engine`` select ``VM instances`. 
2. Click on ``Create`` button to create a new VM.
3. Let's name our instance as ``dda`` as short form for our application name (django-dashboard-adminator)
4. Leave the region and zone as default.
5. In the machine configuration, select the family as ``General-purpose``, series as ``N1`` and the machine type as ``f1-micro``. This machine configuration will ensure that we are within the free tier of our usage.
6. Leave the other options as default and skip ahead to the ``Boot disk`` section.
7. In the ``Boot disk`` section, click on ``Change``. 
8. Under the ``Operating system`` dropdown, select ``Ubuntu``. In the ``Version``, select ``Ubuntu 20.04 LTS`` which is the latest Ubuntu OS that we will be using.
9. In the ``Size (GB)`` type ``20``. Click on ``Select`` button to finalize the Boot disk selection.
10. In the ``Firewall`` section, check ``Allow HTTP traffic`` since we want to access our app through port 80.
11. Expand the ``Management, security, disks, networking, sole tenancy`` options.
12. In the ``Automation`` and ``Startup script``  section, copy and past the content of the ``docker-install.sh`` script that we had written earlier to install Docker CE and Docker Compose. This will ensure that once our VM instances comes online, it would be ready with Docker.
13. Leave everything else default and skip to the end. Click on the ``Create`` button to create the VM instance.



#### D. Login to the VM instance

Once the VM instance is ready (indicated by the green check mark next to the instance name), we are now ready to manually deploy our Docker images and bring up the containers.

SSH to the VM instance by clicking on the ``SSH`` button. This will open up a web console and logs you into your VM.

Let's check whether Docker and Docker Compose have installed correctly, by running a version-check command on each of them individually.

```bash
docker --version
```

```bash
docker-compose --version
```

To test whether Docker engine is able to pull images and run containers properly, let's pull and run the 'hello-world' image.

```bash
sudo docker run hello-world
```

If everything is setup well, you should see a Hello, World! text on your terminal.



#### E. Deploy and run the Docker images

We are now set to deploy and run our Docker images.

First, we need to pull our ``docker-compose.prod.yml`` file that defines our service definitions.

```bash
curl https://raw.githubusercontent.com/<github-user-id>/django-dashboard-adminator/master/docker-compose.prod.yml -o docker-compose.prod.yml
```

Replace ``<github-user-id>`` with your user ID.

This will download the ``docker-compose.prod.yml`` file to our current directory.

Next, we need to add a ``.env`` file for our environment. Let's do that.

```bash
nano .env
```

Paste the following into it

```bash
DEBUG=False
SECRET_KEY=S3cr3t_K#Key
SERVER=<server_IP>
```

Instead of ``<server_IP>``, type the external IP address of the VM instance that can be found on the VM instances page of GCP.

Save the file.

Let's now bring up our containers:

```bash
sudo docker-compose -f docker-compose.prod.yml up -d
```

Navigate to the external IP address of the VM instance and you should be able to see the application up and running!





