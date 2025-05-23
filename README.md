# TP DevOps Correction Docker

Docker TP Report

1-1 For which reason is it better to run the container with a flag -e to give the environment variables rather than
put them directly in the Dockerfile?

Because it helps with security (sets an environment) and makes it flexible for usage (no need to rebuild).

1-2 Why do we need a volume to be attached to our postgres container?

So the data can be kept intact if the container is modified in any way, since the data is stored in the host system

1-3 Document your database container essentials: commands and Dockerfile.

Dockerfile:

FROM postgres:17.2-alpine

ENV POSTGRES_DB=db \
    POSTGRES_USER=usr \
    POSTGRES_PASSWORD=pwd

COPY 01-CreateScheme.sql /docker-entrypoint-initdb.d/
COPY 02-InsertData.sql /docker-entrypoint-initdb.d/

Build command:

docker build -t my-postgres-initdb .

Run command (with volume):

docker run -d \
  --name my-db-container \
  --network app-network \
  -v postgres-data:/var/lib/postgresql/data \
  my-postgres-initdb

1-4 Why do we need a multistage build? And explain each step of this dockerfile.

It helps keep the image a small size, separates build dependencies from runtime, helps with security and cleanliness.

It has a build and a runtime stage, which helps build the workinf directory and then runds the application.

1-5 Why do we need a reverse proxy?

As it acts as an intermediary between clients and backend servers, it helps with security, caching, load balancing and simplifies the access of clients.

1-6 Why is docker-compose so important?

Because it permits the user to define and manage multi-container applications with a single YAML file. Instead of running each container manually, you can start, stop, and configure entire environments (like backend + database + reverse proxy) with one command. It simplifies orchestration, development, and deployment.

1-7 Document docker-compose most important commands.

docker-compose up
Starts all services defined in the docker-compose.yml.

docker-compose up --build
Builds images and starts services.

docker-compose down
Stops and removes containers, networks, and volumes created by up.

docker-compose ps
Lists all running services.

docker-compose logs
Shows logs from all containers.

1-8 Document your docker-compose file.

version: '3.8'

services:
  simple-api-backend:
    build: ./backend-api
    ports:
      - "8081:8080"
    networks:
      - app-network

  reverse-proxy:
    image: httpd:2.4
    ports:
      - "8080:8080"
    volumes:
      - ./simple-http-server/config/my-proxy.conf:/usr/local/apache2/conf/extra/httpd-my-proxy.conf
    command: >
      sh -c "echo 'Include conf/extra/httpd-my-proxy.conf' >> /usr/local/apache2/conf/httpd.conf && httpd-foreground"
    depends_on:
      - simple-api-backend
    networks:
      - app-network

networks:
  app-network:

1-9 Document your publication commands and published images in dockerhub.

# Step 1: Login to DockerHub
docker login

# Step 2: Tag the local image
docker tag session1-docker-simple-api-backend:latest jochystudent/simple-api-backend:1.0

# Step 3: Push the image to DockerHub
docker push jochystudent/simple-api-backend:1.0

1-10 Why do we put our images into an online repo?

So we can:

Share them with teammates or deploy to other machines easily

Integrate them into CI/CD pipelines

Avoid rebuilding the image every time

Track versions and updates

Make the image accessible from anywhere (cloud, servers, dev machines)

GitHub TP Report

2-1 What are test containers?

It is a Java library that is used to run real(not mocked) Docker containers during automated tests.

2-2 For what purpose do we need to use secured variables?

In order to protect the credentials and sensitive data while allowing automated tasks to work securely.

2-3 Why did we put needs: build-and-test-backend on this job? Maybe try without this and you will see!

In order to create a dependency between jobs in the GitHub actions workflow. If we don't use this, the workflow could try to build and publish a Docker image even if the code doesn't compile or tests are failing — which is dangerous in a CI/CD pipeline.

2-4 For what purpose do we need to push docker images?

To make the application portable, reproducible, and ready for automated deployment.

Ansible TP Report

3-1 Document your inventory and base commands

Inventory:
```yaml
all:
  vars:
    ansible_user: admin
    ansible_ssh_private_key_file: "/Users/Jochy/Documents/Educación/EPF Engineering School/DevOps/Session 3 - Ansible/id_rsa"
  children:
    prod:
      hosts:
        joseantonio.silvestremejia.takima.cloud:

Base Commands:
    ansible all -i ansible/inventories/setup.yml -m ping
    ansible all -i ansible/inventories/setup.yml -m setup -a "filter=ansible_distribution*"
    ansible all -i ansible/inventories/setup.yml -m apt -a "name=apache2 state=absent" --become

3-2 Document your playbook

docker.yml:

- hosts: all
  gather_facts: true
  become: true

  roles:
    - docker

roles/docker/tasks/main.yml

- name: Install required packages
  apt:
    name:
      - apt-transport-https
      - ca-certificates
      - curl
      - gnupg
      - lsb-release
      - python3-venv
    state: latest
    update_cache: yes

- name: Add Docker GPG key
  apt_key:
    url: https://download.docker.com/linux/debian/gpg
    state: present

- name: Add Docker APT repository
  apt_repository:
    repo: "deb [arch=amd64] https://download.docker.com/linux/debian {{ ansible_facts['distribution_release'] }} stable"
    state: present
    update_cache: yes

- name: Install Docker
  apt:
    name:
      - docker-ce
      - docker-ce-cli
    state: present

- name: Install Python3 and pip3
  apt:
    name:
      - python3
      - python3-pip
    state: present

- name: Create a virtual environment for Docker SDK
  command: python3 -m venv /opt/docker_venv
  args:
    creates: /opt/docker_venv

- name: Install Docker SDK for Python

This playbook helps with Docker's secure installation and configuration, its SDK for Python is ready in a virtualvenv, it is running well, and is tasks are modularized in a role.

3-3 Document your docker_container tasks configuration.

Role: database
Set Python interpreter for Docker modules
Sets a custom Python interpreter path used by Ansible to interact with Docker SDK.

Run PostgreSQL container
Launches a PostgreSQL container named db using the official postgres:13 image.
The container is started with environment variables for user credentials and database name, connected to the app_net Docker network, with port 5432 published.

Role: app
Set Python interpreter for Docker modules
Sets the Python interpreter for Docker SDK in the virtual environment.

Run app container
Runs the backend application container named my-app from the image jose65/simple-api-backend:1.0.
Container is connected to app_net network, publishes port 3000, restarts automatically, and receives database connection environment variables pointing to the database container.

Role: proxy
Copy nginx config file
Copies the custom default.conf file from the role's files directory to /tmp on the host, and triggers a handler to restart the proxy if changed.

Run Nginx container as reverse proxy
Runs an Nginx container named reverse-proxy that uses the official nginx image.
It maps port 80 and mounts the config file from /tmp/default.conf to the container’s config directory.
The container is connected to the app_net Docker network to proxy requests to the app container.

Handler for proxy role
Restart nginx container
Restarts the reverse-proxy container to apply changes when the Nginx config file is updated.

