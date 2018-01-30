---
title: "Setup Meanstack App With Docker"
date: 2018-01-28T18:55:41+01:00
draft: true
---

This post is about creating a MEAN stack app consisting of Angular 5, MongoDB, NodeJS and Express, running in Docker.

### Docker overview

Docker is, very briefly, used for running applications in containers making them contain everything needed for running the application: runtimes, system tools, libraries, OS and everything you would otherwise need to install youself to run the application.


##### Docker overall deal with these overall elements:

* Containers: Containers that contain all dependencies, runtime as well as libraries, for running the application.
* Images: A read only specification of how a container should be created, that can be instantiated as a container.
* DOCKERFILE: A file used for building docker images, specifying dependencies and configuration for the image and how the image should be run.
* Docker-compose: A file specifying coordination between multiple images used for spin up a whole application setup contain eg. an image for app and one for database.

With Docker it is much easier to get an application running, when we simply need to run one command to get our local setup running with same dependencies on the server.

#### Benefits of Docker over traditional approaches:



#### Containers vs VMs

Running containers in Docker differ from VMs being able to share a lot of the host systems ressources, like networking and kernel, wheras VMs are more isolated in this manner, containering verything within the VM. Because containers share alot of the host systems ressources, containers can be much smaller than VMs and startup in a fraction of the time.


### The Angular app

### The Node app

### MongoDB

### Running the application with Docker-compose

### Conclusion

