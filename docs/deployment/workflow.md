# Deployment Workflow

To understand TU Graz repository deployment, we will devide the process into two sections.

1. Web Server deployment
2. Services deployment

## Web Server deployment

The **[Repository](https://gitlab.tugraz.at/invenio/repository)** is the Web server, which holds all the files and data to build the application's base image and run the docker containers to our instances.

The **base image** is a Docker image that will include all the dependencies required to run the application web-server, and its build with the help of **Dockerfile** and the commands define in it.

From the base image then we are running three containers **web-ui**, **web-api**, and a **worker** with the help of the **docker-compose** file, as shown below.


![](images/baseimage.png?raw=true)


Alongside these three containers from the base image, the **docker-compose** is also responsible to push **[NGINX](https://gitlab.tugraz.at/invenio/nginx)** container to our servers.
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
