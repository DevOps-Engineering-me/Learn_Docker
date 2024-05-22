# NGINX / Node.js / Redis - Sample Node.js application with Nginx proxy and a Redis database.

WE USE 
[awesome-compose](https://github.com/docker/awesome-compose/blob/master/nginx-nodejs-redis/compose.yaml). for practice the docker compose 


## this is an simple docker compose practice.

We use:
- Redis
- NGINX
- Node.js

Work Flow will be like:

![NRN](https://github.com/MdShafiqulSaymon/Portfolio/assets/68004638/a128bb95-e348-4211-93e7-347f41c72af2)

Project structure:
```
.
├── README.md
├── compose.yaml
├── nginx
│   ├── Dockerfile
│   └── nginx.conf
└── web
    ├── Dockerfile
    ├── package.json
    └── server.js
```
## let's Understant in deeply

**1) First We use Dockerfile to build the images.**

**NOTE:Please Follow The Upper Working Directory**

**Create Web Image First**
- initialize the node project using
  ```
  npm init
  ```
- paste in package.json
- ```
  {
  "name": "web",
  "version": "1.0.0",
  "description": "Running Node.js and Express.js on Docker",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.17.2",
    "redis": "3.1.2"
  },
  "author": "",
  "license": "MIT"
  }
  ```
- create server.js
- paste in server.js
- ```
  const os = require('os');
  const express = require('express');
  const app = express();
  const redis = require('redis');
  const redisClient = redis.createClient({
    host: 'redis',
    port: 6379
  });

  app.get('/', function(req, res) {
      redisClient.get('numVisits', function(err, numVisits) {
          numVisitsToDisplay = parseInt(numVisits) + 1;
          if (isNaN(numVisitsToDisplay)) {
              numVisitsToDisplay = 1;
          }
         res.send(os.hostname() +': Number of visits is: ' + numVisitsToDisplay);
          numVisits++;
          redisClient.set('numVisits', numVisits);
      });
  });

  app.listen(5000, function() {
      console.log('Web application is listening on port 5000');
  });
  ```
  - Create Dokcerfile for creating Web Image:
  ```
  FROM node:14.17.3-alpine3.14

  WORKDIR /usr/src/app

  COPY package.json package-lock.json ./
  RUN npm ci
  COPY ./server.js ./

  CMD ["npm","start"]
  ```

Your Web Image is Ready .

**Now Create Nginx Image**
- Create nignx.conf
- ```
    upstream loadbalancer {
    server web1:5000;
    server web2:5000;
  }

  server {
    listen 80;
    server_name localhost;
    location / {
      proxy_pass http://loadbalancer;
    }
  }
  ```
- Create Dockerfile for NGINX image Build
- ```
  FROM nginx:1.21.6
  RUN rm /etc/nginx/conf.d/default.conf
  COPY nginx.conf /etc/nginx/conf.d/default.conf
  ```


**Our Both Image are Ready but Not Build YET**
  
  
- Create Build All Image and Containarize it
- Create compose.yaml


```
  
services:
  redis:
    image: 'redislabs/redismod'
    ports:
      - '6379:6379'
  web1:
    restart: on-failure
    build: ./web
    hostname: web1
    ports:
      - '81:5000'
  web2:
    restart: on-failure
    build: ./web
    hostname: web2
    ports:
      - '82:5000'
  nginx:
    build: ./nginx
    ports:
    - '80:80'
    depends_on:
    - web1
    - web2
```

![compose](https://github.com/MdShafiqulSaymon/Portfolio/assets/68004638/dfcc6071-50d8-44ec-a22b-4246506575bb)


**Let's visualize the port forwarding or mapping**

![final](https://github.com/MdShafiqulSaymon/Portfolio/assets/68004638/71e155b4-2036-46fb-8507-d88730bb68e9)


  


