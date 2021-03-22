# Deployment Workflow

The deployment workflow for [Test Instance](https://invenio-test.tugraz.at) & [Production Instance](https://repository.tugraz.at) are the same.

To understand how the deployment works. We will divide the process into two sections.


1. Web Server deployment
2. Services deployment

## Web Server deployment

The **[Repository](https://gitlab.tugraz.at/invenio/repository)** is the Web server, which holds all the files and data to build the application's base image and run the docker containers to our instances.

The **base image** is a Docker image that will include all the dependencies required to run the application web-server, and its build with the help of **Dockerfile** and the commands define in it.

From the base image then we are running three containers **web-ui**, **web-api**, and a **worker** with the help of the **docker-compose** file, as shown below.


![](images/baseimage.png?raw=true)


Alongside running our three containers from the base image, the **docker-compose** is also responsible to push **[NGINX](https://gitlab.tugraz.at/invenio/nginx)** container to our servers.
which then proxies the requests to the **web-ui**, and **web-api**, as shown below:

![](images/webserver.png?raw=true)


### docker-compose
docker-compose is a tool for defining and running multi-container Docker applications

In **[Repository](https://gitlab.tugraz.at/invenio/repository)** there are two **docker-compose** files. 

  1. **doker-compose-test.yml**

    As the name suggests this docker-compose file is used for [Test Instance](https://invenio-test.tugraz.at).

  2. **docker-compose-prod.yml**

    As the name suggests this docker-compose file is used for [Production Instance](https://repository.tugraz.at).

```yaml
# -*- coding: utf-8 -*-
#
# Copyright (C) 2021 Graz University of Technology
# Maintainer Mojib Wali
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
Both docker-compose files have the same commands, but diffrent conatiner names and environment variables.

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

[Repository](https://gitlab.tugraz.at/invenio/repository) pipeline consist of **seven stages**:

#### dev
In this stage the pipeline is building base image for **Dev instance**, and deploying it to the development server [https://invenio-dev01.tugraz.at](https://invenio-dev01.tugraz.at).

Login to docker, clean docker cache, for containers, volumes and images.
```yml
  before_script:
    - echo "login to docker..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
    - echo "Clean docker cache, for containers, volumes and images..."
    - docker-compose -f docker-compose.dev.yml down
    - docker system prune -f --volumes
    - docker system prune -a -f
    - docker image prune -f
```

* Build and push base image to gitlab [Container Registry](https://gitlab.tugraz.at/invenio/repository/container_registry).Using Tag name ```DEV```.
* Run docker-compose for scripts in ```docker-compose.dev.yml``` file, and deploy to the development instance.
* Run ```wipe_recreate.sh``` scripts for clean Db, Es, Cache and demo rdm records.
```yml
  script:
    - echo "build the image..."
    - docker build --no-cache -t "$CI_REGISTRY_IMAGE":$DEV --build-arg ENVIRONMENT=DEV .
    - echo "Push image to the gitlab registry..."
    - docker push "$CI_REGISTRY_IMAGE":$DEV
    - echo "run docker-compose..."
    - docker-compose -f docker-compose.dev.yml up -d
    - echo "create demo records.."
    - docker exec repository_web-api_1 chmod +x ./wipe_recreate.sh
    - docker exec repository_web-api_1 ./wipe_recreate.sh
```

Only run for branch ```dev``` & ```merge_requests``` to the master branch.
```yml
  only:
    - dev
    - merge_requests
```

Execute jobs with Gitlab-runner tag name ```dev```:
```yml
  tags:
   - dev
```

Full dev stage:

```yml
dev-deploy:
  stage: dev
  before_script:
    - echo "login to docker..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
    - echo "Clean docker cache, for containers, volumes and images..."
    - docker-compose -f docker-compose.dev.yml down
    - docker system prune -f --volumes
    - docker system prune -a -f
    - docker image prune -f
  script:
    - echo "build the image..."
    - docker build --no-cache -t "$CI_REGISTRY_IMAGE":$DEV --build-arg ENVIRONMENT=DEV .
    - echo "Push image to the gitlab registry..."
    - docker push "$CI_REGISTRY_IMAGE":$DEV
    - echo "run docker-compose..."
    - docker-compose -f docker-compose.dev.yml up -d
    - echo "create demo records.."
    - docker exec repository_web-api_1 chmod +x ./wipe_recreate.sh
    - docker exec repository_web-api_1 ./wipe_recreate.sh
  only:
    - dev
    - merge_requests
  tags:
   - dev
```

#### build_test
In this stage the pipeline is building base image for **Test instance**.

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
Using Tag name ```TEST```.
```yml
  script:
    - echo "Build base image..." 
    - docker build --no-cache -t "$CI_REGISTRY_IMAGE":$TEST --build-arg ENVIRONMENT=PROD .
    - echo "Push image to the gitlab registry..." 
    - docker push "$CI_REGISTRY_IMAGE":$TEST
```

Only run for branch ```master``` & ```test```.
```yml
  only:
    - master
```

Execute jobs with Gitlab-runner tag name ```shell```:
```yml
  tags:
   - shell
```

Full build_test stage:
```yml
build-test:
  stage: build_test
  before_script:
    - echo "login to docker..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
    - echo "Clean docker cache, for containers, volumes and images..."
    - docker system prune -f --volumes
    - docker system prune -a -f
    - docker image prune -f
  script:
    - echo "Build base image..." 
    - docker build --no-cache -t "$CI_REGISTRY_IMAGE":$TEST --build-arg ENVIRONMENT=PROD .
    - echo "Push image to the gitlab registry..." 
    - docker push "$CI_REGISTRY_IMAGE":$TEST
  after_script:
    - echo "Logout from docker..." 
    - "docker logout ${CI_REGISTRY}"
  only:
    - master
  tags:
   - shell
```


#### test_one
In this stage pipeline is deploying the base image to the **Web Server(invenio01-test)**.

Login to docker, clean docker cache, for containers, volumes and images.
```yml
  before_script:
    - echo "login to docker..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
    - echo "Clean docker cache, for containers, volumes and images..."
    - docker-compose -f docker-compose.test.yml down
    - docker system prune -f --volumes
    - docker system prune -a -f
    - docker image prune -f
```

Run docker-compose, pull images and run containers defined in ```docker-compose.test.yml``` file.

```yml
  script:
    - echo "run docker-compose..."
    - docker-compose -f docker-compose.test.yml up -d
```

Only run for branch ```master``` & ```test```.
```yml
  only:
    - master
```

Execute jobs with Gitlab-runner tag name ```test01```:
```yml
  tags:
   - test01
```

Logout from docker
```yml
  after_script:
    - echo "Logout from docker..." 
    - "docker logout ${CI_REGISTRY}"
```

Full test_one stage:
```yml
test_one-deploy:
  stage: test_one
  before_script:
    - echo "login to docker..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
    - echo "Clean docker cache, for containers, volumes and images..."
    - docker-compose -f docker-compose.test.yml down
    - docker system prune -f --volumes
    - docker system prune -a -f
    - docker image prune -f

  script:
    - echo "run docker-compose..."
    - docker-compose -f docker-compose.test.yml up -d
  after_script:
    - echo "Logout from docker..." 
    - "docker logout ${CI_REGISTRY}"
  only:
    - master
  tags:
   - test01
```

#### test_two
In this stage pipeline is deploying the base image to the **Web Server(invenio02-test)**.

Login to docker, clean docker cache, for containers, volumes and images.
```yml
  before_script:
    - echo "login to docker..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
    - echo "Clean docker cache, for containers, volumes and images..."
    - docker-compose -f docker-compose.test.yml down
    - docker system prune -f --volumes
    - docker system prune -a -f
    - docker image prune -f
```

Run docker-compose, pull images and run containers defined in ```docker-compose.test.yml``` file.

```yml
  script:
    - echo "run docker-compose..."
    - docker-compose -f docker-compose.test.yml up -d
```

Only run for branch ```master``` & ```test```.
```yml
  only:
    - master
```

Execute jobs with Gitlab-runner tag name ```test02```:
```yml
  tags:
   - test02
```

Logout from docker
```yml
  after_script:
    - echo "Logout from docker..." 
    - "docker logout ${CI_REGISTRY}"
```

Full test_two stage:
```yml
test_two-deploy:
  stage: test_two
  before_script:
    - echo "login to docker..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
    - echo "Clean docker cache, for containers, volumes and images..."
    - docker-compose -f docker-compose.test.yml down
    - docker system prune -f --volumes
    - docker system prune -a -f
    - docker image prune -f

  script:
    - echo "run docker-compose..."
    - docker-compose -f docker-compose.test.yml up -d
  after_script:
    - echo "Logout from docker..." 
    - "docker logout ${CI_REGISTRY}"
  only:
    - master
  tags:
   - test02
```

#### Full .gitlab-ci.yml

```yml
# environment variables
# environment variables
variables:
  DEV: dev
  TEST: test

# Pipline stages
stages:
  - dev
  - build_test
  - test_one
  - test_two
  - build_prod
  - prod_one
  - prod_two

# Build & deploy stage for dev instance
dev-deploy:
  stage: dev
  before_script:
    - echo "login to docker..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
    - echo "Clean docker cache, for containers, volumes and images..."
    - docker-compose -f docker-compose.dev.yml down
    - docker system prune -f --volumes
    - docker system prune -a -f
    - docker image prune -f
  script:
    - echo "build the image..."
    - docker build --no-cache -t "$CI_REGISTRY_IMAGE":$DEV --build-arg ENVIRONMENT=DEV .
    - echo "Push image to the gitlab registry..."
    - docker push "$CI_REGISTRY_IMAGE":$DEV
    - echo "run docker-compose..."
    - docker-compose -f docker-compose.dev.yml up -d
    - echo "create demo records.."
    - docker exec repository_web-api_1 chmod +x ./wipe_recreate.sh
    - docker exec repository_web-api_1 ./wipe_recreate.sh
  only:
    - dev
    - merge_requests
  tags:
   - dev

# Build stage for test instance
build-test:
  stage: build_test
  before_script:
    - echo "login to docker..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
    - echo "Clean docker cache, for containers, volumes and images..."
    - docker system prune -f --volumes
    - docker system prune -a -f
    - docker image prune -f
  script:
    - echo "Build base image..." 
    - docker build --no-cache -t "$CI_REGISTRY_IMAGE":$TEST --build-arg ENVIRONMENT=PROD .
    - echo "Push image to the gitlab registry..." 
    - docker push "$CI_REGISTRY_IMAGE":$TEST
  after_script:
    - echo "Logout from docker..." 
    - "docker logout ${CI_REGISTRY}"
  only:
    - master
  tags:
   - shell

# Deploy stage for web server 01
test_one-deploy:
  stage: test_one
  before_script:
    - echo "login to docker..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
    - echo "Clean docker cache, for containers, volumes and images..."
    - docker-compose -f docker-compose.test.yml down
    - docker system prune -f --volumes
    - docker system prune -a -f
    - docker image prune -f

  script:
    - echo "run docker-compose..."
    - docker-compose -f docker-compose.test.yml up -d
  after_script:
    - echo "Logout from docker..." 
    - "docker logout ${CI_REGISTRY}"
  only:
    - master
  tags:
   - test01

# Deploy stage for web server 02
test_two-deploy:
  stage: test_two
  before_script:
    - echo "login to docker..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
    - echo "Clean docker cache, for containers, volumes and images..."
    - docker-compose -f docker-compose.test.yml down
    - docker system prune -f --volumes
    - docker system prune -a -f
    - docker image prune -f

  script:
    - echo "run docker-compose..."
    - docker-compose -f docker-compose.test.yml up -d
  after_script:
    - echo "Logout from docker..." 
    - "docker logout ${CI_REGISTRY}"
  only:
    - master
  tags:
   - test02

#################################################
# Build stage for prod
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

## Services deployment
Services such as 
[Elasticsearch](https://gitlab.tugraz.at/invenio/elasticsearch),
[PostgreSQL](https://www.postgresql.org/),
[Redis](https://gitlab.tugraz.at/invenio/cache),
[EXIM4](https://gitlab.tugraz.at/invenio/exim4), 
and 
[RabbitMQ](https://gitlab.tugraz.at/invenio/rabbitmq)
have a seperate repository in the Gitlab Group **[invenio](https://gitlab.tugraz.at/invenio)**.

Except [PostgreSQL](https://www.postgresql.org/) other Services deployment are the same.

### EXIM4

#### Dockerfile
A Dockerfile is a text document that contains all the commands a user could call on the command line to assemble an image.

The Dockerfile for [EXIM4](https://gitlab.tugraz.at/invenio/exim4), as below.
```docker
FROM debian:jessie

# This dockerfile was inpired by greinacker/exim4

# install packages: exim4, mailutils
RUN apt-get update \
 && apt-get install --no-install-recommends -y \
  exim4-daemon-light mailutils xtail vim \
 # Slim down image
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* /usr/share/man/?? /usr/share/man/??_*

# add the exim4 start script
ADD start.sh /exim_start
RUN chmod +x /exim_start
ENV EXIM_LOCALINTERFACE=0.0.0.0
ENTRYPOINT ["/exim_start"]
```

#### deploy-prod.sh
Is a helper file for deployment, and used by ```.gitlab-ci.yml```. It contains instruction for docker commands.

```bash
CONTAINER_NAME=smarthost
IMAGE_NAME=registry.gitlab.tugraz.at/invenio/exim4:latest

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
docker run --name="$CONTAINER_NAME" -p 25:25 -d \
-e HOSTNAME="repository.tugraz.at" \
-e EXIM_SMARTHOST="**********" \
-e EXIM_PASSWORD="**********" \
-e EXIM_LOCALINTERFACE="0.0.0.0" \
-e EXIM_DOMAIN="repository.tugraz.at" \
-e EXIM_ALLOWED_SENDERS="**********/28, **********/28" \
--restart=always \
$IMAGE_NAME

echo "################################################"
echo "job ended..."
```

#### .gitlab-ci.yml
[EXIM4](https://gitlab.tugraz.at/invenio/exim4) pipeline consist of **three stages**

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
In this stage the pipeline pulls the newly created image and creates a container to the Test instance.

```before_script``` run followings:

* Install ssh-agent if not already installed, it is required by Docker.
* Run ssh-agent (inside the build environment)
* Add the SSH key stored in SERVER_3_PRIVATE_KEY variable to the agent store
* Create the SSH directory and give it the right permissions
* scan the keys of your private server from variable ```SSH_SERVER_HOSTKEYS```

```yml
  before_script:
    - echo "SSH-USER"
    - 'which ssh-agent || ( apt-get update -y && apt-get install openssh-client git -y )'
    - eval $(ssh-agent -s)
    - echo "$SERVER_3_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
    - echo "$SSH_SERVER_HOSTKEYS" > ~/.ssh/known_hosts
    - chmod 644 ~/.ssh/known_hosts
```
```script```:  using ```ssh``` commands logs in to docker, and runs scripts in ```deploy-test.sh``` to our server.
```yml
  script:
    - ssh $SERVER_3_DOMAIN "echo $CI_REGISTRY_PASSWORD | docker login -u $CI_REGISTRY_USER $CI_REGISTRY --password-stdin"
    - ssh $SERVER_3_DOMAIN "eval '$(cat ./deploy-test.sh)'"
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
  - test
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


test:
  stage: test
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
    - echo "$SERVER_3_PRIVATE_KEY" | tr -d '\r' | ssh-add -
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
    - ssh $SERVER_3_DOMAIN "echo $CI_REGISTRY_PASSWORD | docker login -u $CI_REGISTRY_USER $CI_REGISTRY --password-stdin"
    - ssh $SERVER_3_DOMAIN "eval '$(cat ./deploy-test.sh)'"
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
