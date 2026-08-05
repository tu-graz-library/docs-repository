# Docker image for TU Graz Repository

## Repository
The gitlab repository **[Repository](https://gitlab.tugraz.at/invenio/repository)** is the Web server, which holds all the files and data to build the application's base image and run the docker containers to our instances.

The **base image** is a Docker image that will include all the dependencies required to run the application web-server, and its build with the help of **Dockerfile** and the commands define in it.

### Dockerfile
A Dockerfile is a text document that contains all the commands a user could call on the command line to assemble an image.

The [current Dockerfile](https://gitlab.tugraz.at/invenio/repository/-/blob/master/Dockerfile?ref_type=heads) used to build the Repository Image.


## ArgoCD Deployments
TODO

## Pipeline
TODO
