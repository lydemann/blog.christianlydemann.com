---
title: "Setup MEAN stack Application with Docker"
date: 2018-01-28T18:55:41+01:00
draft: true
---

This post is about creating a MEAN stack app consisting of Angular 5, MongoDB, NodeJS and Express, running in Docker. It is based on a Github demo I created, which can be viewed [here](https://github.com/lydemann/docker-meanstack-demo).


### Docker overview

Docker is used for running applications in containers making them contain everything needed for running the application: runtimes, system tools, libraries, OS and everything you would otherwise need to install yourself to run the application.


#### Containers vs VMs

Running containers in Docker differ from VMs by being able to share the host system's ressources, like networking and kernel, whereas VMs are isolated (with HyperV), containing everything inside its own guest OS. Because containers can share the host systems ressources, containers can be much smaller than VMs and startup in a fraction of the time.

![VM vs Containers](/images/setup-meanstack-app-with-docker/vm-vs-container.jpg)


##### Docker concepts:

* Images: A read only specification of how a container should be created, that can be instantiated as a container.
* Containers: Container are instantiated images that contain all dependencies, runtime as well as libraries, for running the application.
* DOCKERFILE: A file used for building docker images, specifying dependencies and configuration for the image and how the image should be run. This file is read when running the "docker build" and "docker run" commands.
* Docker-engine: The engine providing the Docker containerization technology.
* Docker-compose: A file specifying how images should be build and run and runs on top of the “docker build” and “docker run” commands for setting up multiple containers on the same machine, eg. an image for the client application, one for server and one for database.

#### Benefits of Docker over traditional approaches:

With Docker it is easy to get an application running on different environments, because we can simply build our application as images and run these images as containers on machines with Docker installed. These eases the developer from a lot of the hassle with manually setting up environments and manage dependencies.

## Setting up the application parts

We will now, one by one, setup the different parts of the application and Dockerize them, by creating a Docker image for each part using DOCKERFILEs. This way we can later compose the mean stack app using the different images in a docker-compose.yml.

Before we start coding, we need to have some basic dependencies installed:

- NodeJS, which can be installed from [https://nodejs.org/](https://nodejs.org/) (Just go for stable version)
- Docker: [https://docs.docker.com/install/](https://docs.docker.com/install)
- Your favorite IDE (For this, mine is VS Code)

Let's get started!

### The Angular app


We will start by creating the Angular 5 application.
Luckily, Angular cli makes it easy to scaffold a new Angular app, so we will first make sure we have that installed:

``` npm i -g @angular/cli ```


You then generate a new Angular app with:
``` ng new client ```
This will create a new Angular app in a folder called client, containing a simple "hello world" Angular app. So long so good.


To make sure everything is working, open terminal inside the client directory and run:
``` npm start ```

You should now see this:

![Ng app start](/images/setup-meanstack-app-with-docker/ng-app-start.PNG)

Now we know it works natively. Let's start Dockerizing it.

#### Dockerizing the Angular application

Our goal now is to get the Angular application running in a Docker container. To make the Angular app accessible from outside the Docker container we need to tell it to run on 0.0.0.0 instead of localhost by going to "package.json" in the client folder and change the start script to:

``` "start": "ng serve -H 0.0.0.0",```

Before we start writing the Dockerfile, we create a .dockerignore file to ensure that we don't copy unwanted files to our image:

1. Create .docker inside the client folder
2. Add node_modules and npm-debug.log as this:

**client/.dockerignore**
```
node_modules
npm-debug.log
```

For setting this up we create a new file inside the *client* folder called *DOCKERFILE*. Below is shown a DOCKERFILE that is based on Node Carbon (Node version 8), that installs the npm modules and run using the npm start script:

**client/DOCKERFILE**
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

1. FROM node:carbon: Base image on Node carbon, giving access to npm and node.
2. WORKDIR: Setting the working directory inside the container, making it possible to reference this directory with "."
3. COPY package*.json .: Copying package.json and package-lock.json to the working directory inside the image
4. RUN npm i: Runs npm install at the working directory
5. Copy all from current directory (if not in *.dockerignore* file) to working directory
6. EXPOSE 4200: Showing that the app is to be exposed on port 4200 (but only actually exposed on port 4200 if we specify this in the “docker run” command using the -p parameter).
7. CMD: Specifying that the default command should be npm start. This can be overridden, if wanted, by specifying another command in “docker run”.

Now we can open a terminal inside the client folder and build the image with:

``` docker build -t meanstackapp:1.0 . ```

-t specifies a name and a tag for the build (meanstackapp is the name and 1.0 is the tag).
We specify "." because this will look for a file called *DOCKERFILE* at the current directory (client) in the terminal.

We hereafter run the image with:
``` docker run -p 4200:4200 meanstackapp:1.0 ```
and we should see our app running just like before.


### The Node application

The Node application acts as the api in this MEAN stack application.

We start by:

- Creating a new folder called server
- Add a package.json file using *npm init*
- Add a file called index.js that contains a basic express setup:

**server/index.js**
```
const express = require('express');
const config = require('./config');
const mongoose = require('mongoose');
const bodyParser = require('body-parser');
const app = express();

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

**server/routes.js**
```javascript
const express = require('express');

exports = module.exports = function(app) {

    app.get('/hello', (req, res) => res.send('Hello World!'))
}
 ```

We create a npm start script as:
``` "start": "node index.js",```

and run the application using *npm start*. We should now be able to request the hello endpoint at http://localhost:8080/hello.

#### Dockerizing the Node server

As in the Angular app, we open the terminal at the server directory and start by creating a .dockerignore file:

**server/.dockerignore**
```
node_modules
npm-debug.log
```

We then create DOCKERFILE in the server directory:

**server/DOCKERFILE**
```javascript
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

The above DOCKERIFLE is, in contrast to the Angular DOCKERFILE, a multi step build, making the image smaller by running the image with an Alpine OS.\\
In the first build step the *builder* copies the package.json and package-lock.json into the container and then installs npm dependencies. Step 2 is created from an carbon-alpine image, for smaller image size, and is copying the builder content to the final container. The image is exposed on 8080 and uses NPM start as default cmd.\\
Note: For production build this image can be smaller by using a javascript bundler that supports tree shaking, like Rollup and Webpack.

We can build the image with:\\
```docker build -t meanserver:1.0 .```\\
and run the image with:\\
```docker run -p 8080:8080 meanserver:1.0```

### MongoDB

We can run mongoDB from the official "mongo" image with:
```docker run -p 27017:27017 mongo```

With all the parts set up, let's start connecting them.

### Connecting the parts and running with docker-compose

Now you should have a project tree that looks like this:
```
├───client
│   │   .angular-cli.json
│   │   .dockerignore
│   │   .editorconfig
│   │   .gitignore
│   │   DOCKERFILE
│   │   karma.conf.js
│   │   package-lock.json
│   │   package.json
│   │   protractor.conf.js
│   │   README.md
│   │   tsconfig.json
│   │   tslint.json
│   │
│   ├───e2e
│   │       app.e2e-spec.ts
│   │       app.po.ts
│   │       tsconfig.e2e.json
│   │
│   └───src
│       │   favicon.ico
│       │   index.html
│       │   main.ts
│       │   polyfills.ts
│       │   styles.sass
│       │   test.ts
│       │   tsconfig.app.json
│       │   tsconfig.spec.json
│       │   typings.d.ts
│       │
│       ├───app
│       │       app.component.html
│       │       app.component.sass
│       │       app.component.spec.ts
│       │       app.component.ts
│       │       app.module.ts
│       │       app.service.spec.ts
│       │       app.service.ts
│       │
│       ├───assets
│       │       .gitkeep
│       │
│       └───environments
│               environment.prod.ts
│               environment.ts
│
└───server
        .dockerignore
        .gitignore
        config.js
        DOCKERFILE
        index.js
        package-lock.json
        package.json
        routes.js
```

We now need to connect the client with the server and connect the server to the MongoDB.

#### Connecting client and server

We connect the client and the server by creating a property called *apiUrl* inside the *environment.ts* file, like this:

**client/environment..ts**
```javascript
import { } from 'node';

export const environment = {
 production: false,
 apiUrl: process.env.API_URL || 'http://127.0.0.1:8080'
};
```


This is used as a base url for the http calls:
```javascript
public callHello() {
  return this.http.get(environment.apiUrl + 'hello')
    .map(resp => resp.json())
    .toPromise();
}
```


The ApiUrl can also be overridden using Node environment variables, because it will check if *API_URL* is set in Node environment variables (process.env) or fallback to localhost:8080.

#### Connecting server and database

An easy way for the Node server to interact with the Mongo database is using an ODM and here we use *Mongoose*.
Mongoose is installed by opening the command line at the Node server and run:
``` npm i —save mongoose ```\\
On the Node server in the index.js file we connect to Mongo:

**[server/index.js](https://github.com/lydemann/docker-meanstack-demo/blob/master/server/index.js)**
```javascript 
// Connect to MongoDB
console.log('Connection to mongoDb on uri: ' + config.mongo.uri);
mongoose.connect(config.mongo.uri, config.mongo.options);
mongoose.connection.on('error', function(err) {
 console.error('MongoDB connection error: ' + err);
});
```


#### Running the complete app with Docker Compose

Now that all the parts are setup we specify how the different images should be build and run using docker-compose. We create a docker-compose file containing a service for the angular app, the node server and the Mongo DB like this:

**docker-compose.yml**
```
version: '3' # specify docker-compose version

# Define the services/containers to be run
services:
 client:
   build: ./client
   ports:
     - "80:4200"
 server: # name of the first service
   build: ./server # specify the directory of the Dockerfile
   ports:
     - "8080:8080"
   environment:
     - MONGO_URL=mongodb://database/mean-app
   links:
     - database
   depends_on:
     - database
 database: # name of the third service
   image: mongo # specify image to build container from
   volumes:
     - "/data/db:/data/db"
   ports:
     - "27017:27017" # specify port forewarding
```

This docker-compose file is specifying a service for client, server and database:\\

- Client: We specify the DOCKERFILE for build as client/DOCKERFILE and map it's port from 4200 to 80 on the host.\\
- Server: We specify the DOCKERFILE for build as server/DOCKERFILE and map it's port from 8080 to 8080 on the host. Also links and depends_on is used for linking the database to the server.\\
- Database: The image mongo is used here and volumes for the database are mapped from the container to the host and exposing the container at port 27017 on the host.

Now we can build and run it by opening command line from the project root and type:\\
```docker-compose up````

We should now see our complete MEAN stack application running on localhost:80 (Angular app) and localhost:8080 (Node server).

### Conclusion

In this guide we saw how to setup a simple MEAN stack app and run it all by creating Docker images from DOCKERFILES and build and run them with docker-compose.\\
For easier development, you might also look into creating a dev docker-compose.yml, where you are mounting local files for enabling automatic reload of application on code change, without needing to to install local dependencies, like Node.
A complete demo can be found on my Github [here](https://github.com/lydemann/docker-meanstack-demo).
