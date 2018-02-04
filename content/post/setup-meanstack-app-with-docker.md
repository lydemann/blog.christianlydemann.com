---
title: "Setup Meanstack App With Docker"
date: 2018-01-28T18:55:41+01:00
draft: true
---

This post is about creating a MEAN stack app consisting of Angular 5, MongoDB, NodeJS and Express, running in Docker. It is based on a Github demo I created, which can be viewed [here](https://github.com/lydemann/docker-meanstack-demo)


### Docker overview

Docker is, very briefly, used for running applications in containers making them contain everything needed for running the application: runtimes, system tools, libraries, OS and everything you would otherwise need to install yourself to run the application.


##### Docker overall deal with these overall elements:

* Containers: Containers that contain all dependencies, runtime as well as libraries, for running the application.
* Images: A read only specification of how a container should be created, that can be instantiated as a container.
* DOCKERFILE: A file used for building docker images, specifying dependencies and configuration for the image and how the image should be run.
* Docker-compose: A file specifying coordination between multiple images used for spin up a whole application setup contain eg. an image for app and one for database.

With Docker it is much easier to get an application running, when we simply need to run one command to get our local setup running with same dependencies on the server.

#### Benefits of Docker over traditional approaches:



#### Containers vs VMs

Running containers in Docker differ from VMs being able to share the host system's ressources, like networking and kernel, whereas VMs are isolated, containering everything. Because containers share alot of the host systems ressources, containers can be much smaller than VMs and startup in a fraction of the time.

![VM vs Containers](http://images\setup-meanstack-app-with-docker.jpg)


## Setting up the application parts

We will now, one by one, setup the different parts of the application and Dockerize them, by creating an executable image for each, with a DOCKERFILE. This way we can later just compose the mean stack app using the different images in the docker-composer.yml.

Before we start coding we need to have some basic stuff installed:

- Node, which can be installed from [https://nodejs.org/](https://nodejs.org/) (Just go for stable version)
- Docker: [https://docs.docker.com/install/](https://docs.docker.com/install)
- Your favorite IDE (For this, mine is VS Code


Let's get started!


### The Angular app


We will start by creating the Angular 5 application.
Luckily, Angular cli makes it easy to scaffold a new Angular app, so we will first make sure we have that installed:

``` npm i -g @angular/cli ```


You then generate a new Angular app with:
``` ng new client ```
This will create a new folder called client, containing a simple "hello world" Angular app. So long so good.


To make sure everything is working, open terminal inside the client directory and run:
``` npm start ```

You should now see this:

![Ng app start](http://images\ng-app-start.PNG)

Now we know it works natively. Let's start Dockerizing it.

#### Dockerizing the Angular app

Our goal now is to get the Angular app running in a Docker container. To make the Angular app accessible from outside the Docker container we need to tell it to run on 0.0.0.0 instead of localhost by going to "package.json" in the client folder and change the start script to:

``` "start": "ng serve -H 0.0.0.0",```

Before we start writing the Dockerfile we create a .dockerignore file to ensure that we don't copy unwanted files to our image:

1. Create .docker inside the client folder
2. Add node_modules and npm-debug.log as this:

```
node_modules
npm-debug.log
```

For setting this up we create a new file inside the *client* folder called *DOCKERFILE*.Below is shown a DOCKERFILE that is based on Node Carbon (Node version 8) that installs the npm modules and run using the npm start script:

1. FROM node:carbon: Base image on Node carbon, giving access to npm and node.
2. WORKDIR: Setting the working directory inside the container, making it possible to copy files to the working dir just using "."
3. COPY package*.json .: Copying package.json and package-lock.json to the working directory inside the image
4. RUN npm i: Runs npm install at the working directory
5. Copy all from current directory (if not in *.dockerignore* file) to working directory
6. EXPOSE 4200: Showing that the app is to be exposed on port 4200 (but only actually exposed on port 4200 if specify this to the running container using the -p parameter.
7. CMD: Specifying that the default command should be npm start, if other cmd is not specified when running the image as a container. 


```




FROM node:carbon

WORKDIR /usr/src/app

# Install app dependencies
# A wildcard is used to ensure both package.json AND package-lock.json are copied
# where available (npm@5+)
COPY package*.json .

# Install any needed packages
RUN npm i

# Bundle app source
COPY . .

EXPOSE 4200

CMD [ "npm", "start" ]



```

Now we can build the image with:
``` docker build -t meanstackapp:1.0 . ```
Where -t specifies a tag for the build (meanstackapp is the image name and 1.0 is the tag), so the container can be better identified.
We specify "." because this will look for a file called *DOCKERFILE* at the current directory in the terminal.

After that we can run the image with:
``` docker run -p 4200:4200 meanstackapp:1.0 ``` and we should see our app running just like before.


### The Node app

The Node app acts as the api in this setup.

We start by:

- Creating a new folder called server
- Add a package.json file using *npm init*
- Add a file called index.js that contains a basic express setup:

``` const express = require('express');
const config = require('./config');
const mongoose = require('mongoose');
const bodyParser = require('body-parser');
const app = express();

// Connect to MongoDB
console.log('Connection to mongoDb on uri: ' + config.mongo.uri);
mongoose.connect(config.mongo.uri, config.mongo.options);
mongoose.connection.on('error', function(err) {
  console.error('MongoDB connection error: ' + err);
});

// Parsers for POST data
app.use(bodyParser.json());
app.use(bodyParser.urlencoded({ extended: false }));

// Cross Origin middleware
app.use(function(req, res, next) {
  res.header("Access-Control-Allow-Origin", "*")
  res.header("Access-Control-Allow-Headers", "Origin, X-Requested-With, Content-Type, Accept")
  next()
})

require('./routes')(app);

app.listen(config.port, () => console.log(`Example app listening on ${config.port}!`))


```

We also create a routes.js file for api endpoints containing a simple "hello world" endpoint:
```
const path = require('path');
const express = require('express');

exports = module.exports = function(app) {

    const staticFilesPath = path.resolve('./static');
  
    app.get('/hello', (req, res) => res.send('Hello World!'))

    app.use(express.static(staticFilesPath))

}
  
```

We create an npm start script as:
``` "start": "node index.js",```

And run the application using npm start. We should now be able to request the hello enpoint at [http:localhost8080/hello](http:localhost8080/hello.

#### Dockerizing the Node server

As in the Angular app we open terminal at the server directory and start by creating a .dockerignore file containing:
```
node_modules
npm-debug.log
```

We then create DOCKERFILE in the server directory and type:
```
FROM node:carbon as builder

# Create app directory
WORKDIR /usr/src/app

# Install app dependencies
# A wildcard is used to ensure both package.json AND package-lock.json are copied
# where available (npm@5+)
COPY package*.json ./

# Install any needed packages
RUN npm install

# Bundle app source
COPY . .

# Stage 2 build for creating smaller image
FROM node:carbon-alpine
WORKDIR /usr/src/app

COPY --from=builder /usr/src/app .

EXPOSE 8080

CMD [ "npm", "start" ]

```

The above Docker file, is in contrast to the Angular DOCKERFILE, a multi step build, making the image smaller by running the image with an Alpine OS.
In the first build step the *builder* copies the package.json and package-lock.json into the container and then installs npm dependencies. Step 2 is created from an carbon-alpine image, for smaller image size, and is copying the builder content to the final container. The image is exposed on 8080 and uses NPM start as default cmd.


### MongoDB


### Connecting the parts and running with docker-compose

#### Connecting client and server
#### Connecting server and database

Now you should have folders that looks like this:
![MEAN stack folders](http://images\mean-stack-folders.PNG)


### Conclusion


