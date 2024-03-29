
# Production Instance
[repository.tugraz.at](https://repository.tugraz.at/)

Production instance has **3** VMs:

1. Web Server VM (invenio01-prod)
2. Web Server VM (invenio02-prod)
3. Services VM (invenio03-prod)


These virtual machines are split into two categories, **Web Servers** and **Services**. In this guideline we will take a look on both:

## Web Server

Base image of the [repository](https://gitlab.tugraz.at/invenio/repository) instance.

* **[uWSGI](https://uwsgi-docs.readthedocs.io/en/latest/WSGIquickstart.html)**
is a software application that "aims at developing a full stack for building hosting services".
uwsgi (all lowercase) is the native binary protocol that uWSGI uses to communicate with other servers.

* **[Celery](https://docs.celeryproject.org/en/stable/userguide/application.html)**
is asynchronous task queue or job queue which is based on distributed message passing. While it supports scheduling, its focus is on operations in real time.

Alongside our base image, we are also pushing a [NGINX](https://gitlab.tugraz.at/invenio/nginx) container as a front-end proxy.

* **[Nginx](https://nginx.org/en/docs/)**
is a web server that can also be used as a reverse proxy, load balancer, mail proxy and HTTP cache. 

The web server VMs are configured to the F5 load balancer that is provided by [Tu Graz ZID](https://www.tugraz.at/tu-graz/organisationsstruktur/serviceeinrichtungen-und-stabsstellen/zentraler-informatikdienst/).

The F5 load balancer is forwarding the requests to one of the (Web Server VM), and then the requests are proxied by [NGINX](https://gitlab.tugraz.at/invenio/nginx) to [UWSGI web applications](https://uwsgi-docs.readthedocs.io/en/latest/WSGIquickstart.html). As shown below:

![](images/invenio-prod.png?raw=true)

The gitlab repository **[Repository](https://gitlab.tugraz.at/invenio/repository)** is the Web server, which holds all the files and data to build the application's base image and run the docker containers to our instances.

The **base image** is a Docker image that will include all the dependencies required to run the application web-server, and its build with the help of **Dockerfile** and the commands define in it.

From the base image then we are running three containers **web-ui**, **web-api**, and a **worker** with the help of the **docker-compose** file, as shown below.


![](images/baseimage.png?raw=true)


Alongside these three containers from the base image, the **docker-compose** is also responsible to push a **[NGINX](https://gitlab.tugraz.at/invenio/nginx)** container to our servers.
which then proxies the requests to the **web-ui**, and **web-api**, as shown below:

![](images/webserver.png?raw=true)


### docker-compose
docker-compose is a tool for defining and running multi-container Docker applications.

**docker-compose-prod.yml**

```yaml
# -*- coding: utf-8 -*-
#
# Copyright (C) 2021 Graz University of Technology
#
# Following services are included:
# - Frontend server: Nginx (exposed port: 8080)
# - UI application: UWSGI (not exposed)
# - API application: UWSGI (not exposed)
# - Worker: Celery (not exposed)

version: '2.2'
services:
  # Frontend
  frontend:
    image: registry.gitlab.tugraz.at/invenio/nginx:prod
    restart: "always"
    volumes:
      - static_data:/opt/invenio/var/instance/static
    links:
      - web-ui
      - web-api
    ports:
      - "8080:8080"

  # UI Application
  web-ui:
    command: ["uwsgi /opt/invenio/var/instance/uwsgi_ui.ini"]
    image: registry.gitlab.tugraz.at/invenio/repository:${TAG_PROD}
    env_file: ${ENV_FILE_PROD}
    ports:
      - "5000"
    volumes:
      - static_data:/opt/invenio/var/instance/static
      - uploaded_data:/opt/invenio/var/instance/data
      - archived_data:/opt/invenio/var/instance/archive

  # API Rest Application
  web-api:
    command: ["uwsgi /opt/invenio/var/instance/uwsgi_rest.ini"]
    image: registry.gitlab.tugraz.at/invenio/repository:${TAG_PROD}
    env_file: ${ENV_FILE_PROD}
    ports:
      - "5000"
    volumes:
      - uploaded_data:/opt/invenio/var/instance/data
      - archived_data:/opt/invenio/var/instance/archive

  # Worker
  worker:
    restart: "always"
    env_file: ${ENV_FILE_PROD}
    command: ["celery -A invenio_app.celery worker --loglevel=INFO"]
    image: registry.gitlab.tugraz.at/invenio/repository:${TAG_PROD}

volumes:
  static_data:
  uploaded_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /storage
  archived_data:

```

### Dockerfile
A Dockerfile is a text document that contains all the commands a user could call on the command line to assemble an image.

The Dockerfile for **[Repository](https://gitlab.tugraz.at/invenio/repository)**, as below.

```docker
# Dockerfile that builds a fully functional image of your app.
#
# This image installs all Python dependencies for your application. It's based
# on CentOS 8 with Python 3.8 (https://github.com/inveniosoftware/docker-invenio)
# and includes Pip, Pipenv, Node.js, NPM and some few standard libraries
# Repository usually needs.

# Pulling Centos8 with python3.8 image.
FROM inveniosoftware/centos8-python:3.8

# env arg
ARG ENVIRONMENT

# These packages are required for shibboleth authentication.
# Installing xmlsec packages
RUN yum localinstall -y http://mirror.centos.org/centos/8/PowerTools/x86_64/os/Packages/xmlsec1-devel-1.2.25-4.el8.x86_64.rpm \
                        http://mirror.centos.org/centos/8/PowerTools/x86_64/os/Packages/xmlsec1-openssl-devel-1.2.25-4.el8.x86_64.rpm

# Installing libtool package
RUN yum localinstall -y http://mirror.centos.org/centos/8/AppStream/x86_64/os/Packages/libtool-ltdl-devel-2.4.6-25.el8.x86_64.rpm

# Copy Pipfile and pipfile.lock to ./ directory
COPY Pipfile Pipfile.lock ./
# Install all the dependecies defined in the Pipfile.
RUN pipenv install --deploy --system --pre

# Copy other necessary files
COPY ./docker/uwsgi/ ${INVENIO_INSTANCE_PATH}
COPY ./prod/invenio.cfg ${INVENIO_INSTANCE_PATH}
COPY ./templates/ ${INVENIO_INSTANCE_PATH}/templates/
COPY ./app_data/ ${INVENIO_INSTANCE_PATH}/app_data/
COPY ./ .

# This will create, install and build all the statics & assets. 
RUN cp -r ./static/. ${INVENIO_INSTANCE_PATH}/static/ && \
    cp -r ./assets/. ${INVENIO_INSTANCE_PATH}/assets/ && \
    invenio collect --verbose  && \
    invenio webpack create && \
    invenio webpack install --unsafe && \
    invenio webpack build


# Instruction used to configure how the container will run.
ENTRYPOINT [ "bash", "-c"]
```

### Pipeline

![](images/pipeline_repo.png?raw=true)

### .gitlab-ci.yml
is a YAML file that you create on your project's root. This file automatically runs whenever you push code in the repository. Pipelines consist of one or more stages that run in order and can each contain one or more jobs that run in parallel. These jobs (or scripts) get executed by the [GitLab Runner](https://docs.gitlab.com/runner/) agent.

[Repository](https://gitlab.tugraz.at/invenio/repository) production pipeline consist of **three stages**:


#### build_prod
In this stage the pipeline is building base image for **Production instance**.

Login to docker, clean docker cache, for containers, volumes and images.
```yml
  before_script:
    - echo "login to docker..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
    - echo "Clean docker cache, for containers, volumes and images..."
    - docker system prune -f --volumes
    - docker system prune -a -f
    - docker image prune -f
```

Build and push base image to gitlab [Container Registry](https://gitlab.tugraz.at/invenio/repository/container_registry).
Using Tag name ```CI_COMMIT_TAG```, which is a reference to the latest repository tag/release.
```yml
  script:
    - echo "Build base image..." 
    - docker build --no-cache -t "$CI_REGISTRY_IMAGE":$CI_COMMIT_TAG --build-arg ENVIRONMENT=PROD . 
    - echo "Push image to the gitlab registry..." 
    - docker push "$CI_REGISTRY_IMAGE":$CI_COMMIT_TAG
```

Logout from docker
```yml
  after_script:
    - echo "Logout from docker..." 
    - "docker logout ${CI_REGISTRY}"
```

Only run for ```tags```.
```yml
  only:
    - tags
```

Execute jobs with Gitlab-runner tag name ```shell```:
```yml
  tags:
   - shell
```

Full build_prod stage:
```yml
build_prod:
  stage: build_prod
  before_script:
    - echo "login to docker..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
    - echo "Clean docker cache, for containers, volumes and images..."
    - docker system prune -f --volumes
    - docker system prune -a -f
    - docker image prune -f
  script:
    - echo "Build base image..." 
    - docker build --no-cache -t "$CI_REGISTRY_IMAGE":$CI_COMMIT_TAG --build-arg ENVIRONMENT=PROD . 
    - echo "Push image to the gitlab registry..." 
    - docker push "$CI_REGISTRY_IMAGE":$CI_COMMIT_TAG
  after_script:
    - echo "Logout from docker..." 
    - "docker logout ${CI_REGISTRY}"
  only:
    - tags
  tags:
   - shell
```


#### prod_one-deploy
In this stage pipeline is deploying the base image to the **Web Server(invenio01-prod)**.

Login to docker, clean docker cache, containers, volumes & images.
```yml
  before_script:
    - echo "login to docker..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
    - echo "Clean docker cache, for containers, volumes and images..."
    - docker-compose -f docker-compose.prod.yml down
    - docker system prune -f --volumes
    - docker system prune -a -f
    - docker image prune -f
```

Run docker-compose, pull images and run containers defined in ```docker-compose.prod.yml``` file.

```yml
  script:
    - echo "run docker-compose..."
    - docker-compose -f docker-compose.prod.yml up -d
```

Logout from docker
```yml
  after_script:
    - echo "Logout from docker..." 
    - "docker logout ${CI_REGISTRY}"
```

Only run for ```tags```.
```yml
  only:
    - tags
```

Execute jobs with Gitlab-runner tag name ```prod01```:
```yml
  tags:
   - prod01
```


Full prod_one-deploy stage:
```yml
prod_one-deploy:
  stage: prod_one
  before_script:
    - echo "login to docker..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
    - echo "Clean docker cache, for containers, volumes and images..."
    - docker-compose -f docker-compose.prod.yml down
    - docker system prune -f --volumes
    - docker system prune -a -f
    - docker image prune -f

  script:
    - echo "run docker-compose..."
    - docker-compose -f docker-compose.prod.yml up -d
  after_script:
    - echo "Logout from docker..." 
    - "docker logout ${CI_REGISTRY}"
  only:
    - tags
  tags:
   - prod01

```

#### prod_two-deploy
In this stage pipeline is deploying the base image to the **Web Server(invenio02-prod)**.

Login to docker, clean docker cache, containers, volumes & images.
```yml
  before_script:
    - echo "login to docker..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
    - echo "Clean docker cache, for containers, volumes and images..."
    - docker-compose -f docker-compose.prod.yml down
    - docker system prune -f --volumes
    - docker system prune -a -f
    - docker image prune -f
```

Run docker-compose, pull images and run containers defined in ```docker-compose.prod.yml``` file.

```yml
  script:
    - echo "run docker-compose..."
    - docker-compose -f docker-compose.prod.yml up -d
```
Logout from docker
```yml
  after_script:
    - echo "Logout from docker..." 
    - "docker logout ${CI_REGISTRY}"
```

Only run for ```tags```.
```yml
  only:
    - tags
```

Execute jobs with Gitlab-runner tag name ```prod02```:
```yml
  tags:
   - prod02
```

Full test_two stage:
```yml
prod_two-deploy:
  stage: prod_two
  before_script:
    - echo "login to docker..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
    - echo "Clean docker cache, for containers, volumes and images..."
    - docker-compose -f docker-compose.prod.yml down
    - docker system prune -f --volumes
    - docker system prune -a -f
    - docker image prune -f

  script:
    - echo "run docker-compose..."
    - docker-compose -f docker-compose.prod.yml up -d
  after_script:
    - echo "Logout from docker..." 
    - "docker logout ${CI_REGISTRY}"
  only:
    - tags
  tags:
   - prod02
```

#### Full .gitlab-ci.yml

```yml

# Pipline stages
stages:
  - build_prod
  - prod_one
  - prod_two

build_prod:
  stage: build_prod
  before_script:
    - echo "login to docker..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
    - echo "Clean docker cache, for containers, volumes and images..."
    - docker system prune -f --volumes
    - docker system prune -a -f
    - docker image prune -f
  script:
    - echo "Build base image..." 
    - docker build --no-cache -t "$CI_REGISTRY_IMAGE":$CI_COMMIT_TAG --build-arg ENVIRONMENT=PROD . 
    - echo "Push image to the gitlab registry..." 
    - docker push "$CI_REGISTRY_IMAGE":$CI_COMMIT_TAG
  after_script:
    - echo "Logout from docker..." 
    - "docker logout ${CI_REGISTRY}"
  only:
    - tags
  tags:
   - shell

# Deploy stage for web server 01
prod_one-deploy:
  stage: prod_one
  before_script:
    - echo "login to docker..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
    - echo "Clean docker cache, for containers, volumes and images..."
    - docker-compose -f docker-compose.prod.yml down
    - docker system prune -f --volumes
    - docker system prune -a -f
    - docker image prune -f

  script:
    - echo "run docker-compose..."
    - docker-compose -f docker-compose.prod.yml up -d
  after_script:
    - echo "Logout from docker..." 
    - "docker logout ${CI_REGISTRY}"
  only:
    - tags
  tags:
   - prod01

# Deploy stage for web server 02
prod_two-deploy:
  stage: prod_two
  before_script:
    - echo "login to docker..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
    - echo "Clean docker cache, for containers, volumes and images..."
    - docker-compose -f docker-compose.prod.yml down
    - docker system prune -f --volumes
    - docker system prune -a -f
    - docker image prune -f

  script:
    - echo "run docker-compose..."
    - docker-compose -f docker-compose.prod.yml up -d
  after_script:
    - echo "Logout from docker..." 
    - "docker logout ${CI_REGISTRY}"
  only:
    - tags
  tags:
   - prod02
```

## Services
TU Graz Repository consist of these services:

* **[Elasticsearch](https://gitlab.tugraz.at/invenio/elasticsearch)** is a search engine based on the Lucene library.
It provides a distributed, multitenant-capable full-text search engine with an HTTP web interface and schema-free JSON documents. Elasticsearch is developed in Java.

* **[PostgreSQL](https://www.postgresql.org/)** is a free and open-source relational database management system (RDBMS) emphasizing extensibility and SQL compliance.

* **[Redis](https://gitlab.tugraz.at/invenio/cache)** Remote Dictionary Server is an in-memory data structure project implementing a distributed, in-memory key–value database with optional durability. Redis supports different kinds of abstract data structures, such as strings, lists, maps, sets, sorted sets, HyperLogLogs, bitmaps, streams, and spatial indexes.

* **[EXIM4](https://gitlab.tugraz.at/invenio/exim4)** Exim4 is a Message Transfer Agent (MTA) developed at the University of Cambridge for use on Unix systems connected to the Internet. Exim can be installed in place of sendmail, although its configuration is quite different.

* **[RabbitMQ](https://gitlab.tugraz.at/invenio/rabbitmq)** RabbitMQ is an open-source message-broker software that originally implemented the Advanced Message Queuing Protocol and has since been extended with a plug-in architecture to support Streaming Text Oriented Messaging Protocol, MQ Telemetry Transport.


These services have a seperate repository in the Gitlab Group **[invenio](https://gitlab.tugraz.at/invenio)**.

Except [PostgreSQL](https://www.postgresql.org/) other Services deployment are the same.

As an Example we will have a look into our [Elasticsearch](https://gitlab.tugraz.at/invenio/exim4) deployment.

### Elasticsearch

#### Dockerfile
A Dockerfile is a text document that contains all the commands a user could call on the command line to assemble an image.

The Dockerfile for [Elasticsearch](https://gitlab.tugraz.at/invenio/elasticsearch), as below.
```docker
# Pull elasticsearch version 7
From docker.elastic.co/elasticsearch/elasticsearch-oss:7.9.3

# add single-node config
RUN echo "discovery.type: single-node" >> /usr/share/elasticsearch/config/elasticsearch.yml

# define docker User
USER elasticsearch
```

#### deploy-prod.sh
Is a helper file for deployment, and used by .gitlab-ci.yml. It contains instruction for docker commands.

```bash
CONTAINER_NAME=elasticsearch
IMAGE_NAME=registry.gitlab.tugraz.at/invenio/elasticsearch:latest

echo "################################################"
echo "Stopping and removing container with given name......$CONTAINER_NAME"
docker stop $CONTAINER_NAME || true && docker rm $CONTAINER_NAME || true

echo "################################################"
echo "Removing image if given name exists...$IMAGE_NAME"
if test ! -z "$(docker images -q $IMAGE_NAME)"; then
  echo "Image exist..."
  docker rmi -f $IMAGE_NAME
fi

echo "################################################"
echo "Running new container.............."
docker run --name="$CONTAINER_NAME" \
--mount type=bind,source=/storage,target=/usr/share/elasticsearch/data \
--memory 1g -p 9200:9200 -p 9300:9300 -d \
-e bootstrap.memory_lock=true \
-e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
-e discovery.type=single-node \
--health-cmd="curl --fail localhost:9200/_cluster/health?wait_for_status=green || exit 1" \
--health-interval=30s \
--health-retries=5 \
--health-timeout=30s \
--restart=always \
$IMAGE_NAME

echo "################################################"
echo "job ended..."
```

#### .gitlab-ci.yml
[Elasticsearch](https://gitlab.tugraz.at/invenio/elasticsearch) pipeline consist of **two stages**

##### build
Pipeline builds the docker image and pushes it to the Registry.

Login to docker:
```yml
  before_script:
    - echo "################################"
    - echo "login to docker..."
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
```
Build and push docker image:
```yml
  script:
    - echo "################################"
    - echo "Building the image..."
    - docker build -t "$CI_REGISTRY_IMAGE" .
    - echo "push image to registry..."
    - docker push "$CI_REGISTRY_IMAGE"
```
Only run for branch ```master```:
```yml
  only:
    - master
```
Execute jobs with Gitlab-runner tag name ```shell```:
```yml
  tags:
   - shell
```

##### test
In this stage the pipeline pulls the newly created image and creates a container to the VM.

```before_script``` run followings:

* Install ssh-agent if not already installed, it is required by Docker.
* Run ssh-agent (inside the build environment)
* Add the SSH key stored in PROD_03_PRIVATE_KEY variable to the agent store
* Create the SSH directory and give it the right permissions
* scan the keys of your private server from variable ```SSH_SERVER_HOSTKEYS```

```yml
  before_script:
    - echo "SSH-USER"
    - 'which ssh-agent || ( apt-get update -y && apt-get install openssh-client git -y )'
    - eval $(ssh-agent -s)
    - echo "$PROD_03_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
    - echo "$SSH_SERVER_HOSTKEYS" > ~/.ssh/known_hosts
    - chmod 644 ~/.ssh/known_hosts
```
```script```:  using ```ssh``` commands logs in to docker, and runs scripts in ```deploy-prod.sh``` to our server.
```yml
  script:
    - ssh $PROD_03_DOMAIN "echo $CI_REGISTRY_PASSWORD | docker login -u $CI_REGISTRY_USER $CI_REGISTRY --password-stdin"
    - ssh $PROD_03_DOMAIN "eval '$(cat ./deploy-test.sh)'"
```
Only run for branch ```master```:
```yml
  only:
    - master
```
Execute jobs with Gitlab-runner tag name ```shell```:
```yml
  tags:
   - shell
```

##### prod
In this stage the pipeline pulls the newly created image and creates a container to the Production instance.

```before_script``` run followings:

* Install ssh-agent if not already installed, it is required by Docker.
* Run ssh-agent (inside the build environment)
* Add the SSH key stored in PROD_03_PRIVATE_KEY variable to the agent store
* Create the SSH directory and give it the right permissions
* scan the keys of your private server from variable ```SSH_SERVER_HOSTKEYS```

```yml
  before_script:
    - echo "SSH-USER"
    - 'which ssh-agent || ( apt-get update -y && apt-get install openssh-client git -y )'
    - eval $(ssh-agent -s)
    - echo "$PROD_03_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
    - echo "$SSH_SERVER_HOSTKEYS" > ~/.ssh/known_hosts
    - chmod 644 ~/.ssh/known_hosts
```
```script```:  using ```ssh``` commands logs in to docker, and runs scripts in ```deploy-prod.sh``` to our server.
```yml
  script:
    - ssh $PROD_03_DOMAIN  "echo $CI_REGISTRY_PASSWORD | docker login -u $CI_REGISTRY_USER $CI_REGISTRY --password-stdin"
    - ssh $PROD_03_DOMAIN  "eval '$(cat ./deploy-prod.sh)'"
```
Only run for branch ```master```:
```yml
  only:
    - master
```
Execute jobs with Gitlab-runner tag name ```shell```:
```yml
  tags:
   - shell
```

#### Full .gitlab-ci.yml
```yml
stages:
  - build
  - prod

build:
  stage: build
  before_script:
    - echo "################################"
    - echo "login to docker..."
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
  script:
    - echo "################################"
    - echo "Building the image..."
    - docker build -t "$CI_REGISTRY_IMAGE" .
    - echo "push image to registry..."
    - docker push "$CI_REGISTRY_IMAGE"
  only:
    - master
  tags:
   - shell

prod:
  stage: prod
  before_script:
    - echo "################################"
    - echo "SSH-USER"
  ##
  ## Install ssh-agent if not already installed, it is required by Docker.
  ## (change apt-get to yum if you use an RPM-based image)
  ##
    - 'which ssh-agent || ( apt-get update -y && apt-get install openssh-client git -y )'
  ##
  ## Run ssh-agent (inside the build environment)
  ##
    - eval $(ssh-agent -s)
  ##
  ## Add the SSH key stored in SSH_PRIVATE_KEY variable to the agent store
  ## We're using tr to fix line endings which makes ed25519 keys work
  ## without extra base64 encoding.
  ## https://gitlab.com/gitlab-examples/ssh-private-key/issues/1#note_48526556
  ##
    - echo "$PROD_03_PRIVATE_KEY" | tr -d '\r' | ssh-add -
  ##
  ## Create the SSH directory and give it the right permissions
  ##
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
  ##
  ## Use ssh-keyscan to scan the keys of your private server. Replace gitlab.com
  ## with your own domain name. You can copy and repeat that command if you have
  ## more than one server to connect to.
  ##
    #- ssh-keyscan invenio03-test.tugraz.at >> ~/.ssh/known_hosts
    #- chmod 644 ~/.ssh/known_hosts
  ##
  ## Alternatively, assuming you created the SSH_SERVER_HOSTKEYS variable
  ## previously, uncomment the following two lines instead.
  ##
    - echo "$SSH_SERVER_HOSTKEYS" > ~/.ssh/known_hosts
    - chmod 644 ~/.ssh/known_hosts

  script:
    - echo "################################"
    - ssh $PROD_03_DOMAIN "echo $CI_REGISTRY_PASSWORD | docker login -u $CI_REGISTRY_USER $CI_REGISTRY --password-stdin"
    - ssh $PROD_03_DOMAIN "eval '$(cat ./deploy-prod.sh)'"
  only:
    - master
  tags:
   - shell
```
