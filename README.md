# Docker: Building Node Apps Without Installing Node

Docker allows us to be able to experiment with different technology without the overhead of polluting our systems with one-off installations. If you decide to continue working with the tech, Docker offers us a sandboxed, fast, and predictable development environment.

## Requirements

* Have Docker installed in your system.
* Basic knowledge of Docker.

## Getting Node

The first question is: how to scaffold a Node project without installing Node? I want my project to run the current LTS version of Node which is `10.15.0`. How do I get it? I'd pull the image of that version of Node from Docker:

```bash
docker pull node:10.15.0
```

## Setting a Node App with Docker

Once that image is pulled, I'll use it to initiation my Node project using `npm init -y`:

```bash
docker run --rm -v "${PWD}:/src" -w /src  node:10.15.0 npm init -y
```

I'll need to define some dependencies for this project as follows:

* `express` to build an API.
* `nodemon` to restart the Node app whenever there are changes on the source code. 

I'll run a similar command as before with `npm install` this time around:

```
docker run --rm -it -v "${PWD}:/src" -w /src  node:10.15.0 npm i express && npm i -D nodemon
```

I also added the `-it` flag to be able to see the NPM installer in action and track the installation progress.

As I am going to start the app using `nodemon`, I'll create a custom NPM script for it within `package.json`:

```
{
  "name": "src",
  "version": "1.0.0",
  "description": "Creating a basic Node Dev Env with Docker",
  "main": "index.js",
  "scripts": {
    "start": "nodemon index.js"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "express": "^4.16.4"
  },
  "devDependencies": {
    "nodemon": "^1.18.9"
  }
}
```

Next, I need to create the entry point of my Node app, `index.js`:

```bash
touch index.js
```

```javascript
// index.js
'use strict';

const express = require('express');

// Constants
const PORT = process.env.PORT || 3000;
const HOST = '0.0.0.0';

// App
const app = express();
app.get('/', (req, res) => {
  res.send('Hello world\n');
});

app.listen(PORT, HOST);
console.log(`Running on http://${HOST}:${PORT}`);
```

I got a `package.json` and an Node app ready within `index.js`. What I need now is a way to _run_ this. Enter Docker Compose.

## Running a Node App with Docker

I'll create a Docker Compose file to take care of this:

```bash
touch docker-compose.yml
```

```yml
version: "3"
services:
  web:
    image: node:10.15.0
    command: sh -c "npm install && npm start"
    working_dir: /app
    volumes:
      - ./:/app
    ports:
      - 8080:8080
    environment:
      - PORT=8080
```

Now I just need to run Docker Compose as follows:

```bash
docker-compose up
```

Docker Compose will build the containers needed from the services specified in the `docker-compose.yml`. When it's done, I'll see some output in the shell about `nodemon` running and the message `Running on http://0.0.0.0:8080`.

I'll open a new tab and call the endpoint like this:

```bash
curl 0.0.0.0:8080
```

Response:

```bash
Hello world
```

I want to change the response to be `Hola gente!`. I open `index.js` and update it, save my code, and `nodemon` takes care of updating running server. If I curl again, I get the updated response.

That's it. I now having a running Node Development Environment without the need to install Node. I can take commit this code to GitHub, clone it, and running in any platform where Docker and Docker Compose are present. Not only the environment is portable but also predictable.

## What's Next?

I am sure I'll be extending this application. Wat if I need other Node packages like `body-parser`. For that, while my app is running through containers in Docker, I can go into the running Node environment and run any commands that I need:

I need to get the name of the container that's running my application:

```bash
docker-compose ps
```

Output:

```bash
      Name                    Command               State           Ports         
----------------------------------------------------------------------------------
node-basic_web_1   sh -c npm install && npm start   Up      0.0.0.0:8080->8080/tcp
```

Next, I need to go into the bash environment of that container:

```
docker exec -it  node-basic_web_1 /bin/bash
```

I'll get a shell as output:

```bash
root@:/app# 
```

I can `ls` to see the source of my app. I can also install NPM packages like I would if the app was running locally:

```
npm i body-parser
```

If I open my local `package.json`, I will see that the `dependencies` have been updated. This is because I have mounted my local volume into the Docker container.

I can keep this bash shell open to do all sort of cool things like installing Redis! But if I need Redis, I have to install it somewhere. Yes, I can install it used Docker Compose and a Redis image.

## Installing Redis

In the bash shell I opened I can install `redis` as follows:

```npm
npm i redis
```

I am also going to create a promisified client for Redis:

```bash
touch redis-client.js
```

```javascript
// redis-client.js

const redis = require('redis');
const {promisify} = require('util');
const client = redis.createClient(process.env.REDIS_URL);

module.exports = {
  ...client,
  getAsync: promisify(client.get).bind(client),
  setAsync: promisify(client.set).bind(client),
  keysAsync: promisify(client.keys).bind(client)
};
```

Before I used this Redis client in my app, I need to link my Docker Compose `web` service with a Redis container:

```yml
version: "3"
services:
  web:
    image: node:10.15.0
    command: sh -c "npm install && npm start"
    working_dir: /app
    volumes:
      - ./:/app
    ports:
      - 8080:8080
    environment:
      - PORT=8080
      - REDIS_URL=redis://cache
    links:
      - data
  data:
    image: redis
    container_name: cache
    expose: 
      - 6379
```

Since I am changign the way my containers are build. I need to stop the running Docker Compose environment and start it again. I few `CTRL + C`'s will do and then run it again:

```bash
docker-compose up
```

Since I had not gotten the `redis` image before, Docker Compose will spend some time donwloading it. Once that's done, I'll see some Redis output in the console, along with the `nodemon` one.

I can how start using my Redis client in `index.js` without errors:

```javascript
// index.js
'use strict';

const express = require('express');
const redisClient = require('./redis-client');

// Constants
const PORT = process.env.PORT || 3000;
const HOST = '0.0.0.0';

// App
const app = express();
app.get('/', (req, res) => {
  res.send('Hola  world\n');
});

app.post('/store/:key', async (req, res) => {
  const { key } = req.params;
  const value = req.query;
  await redisClient.setAsync(key, JSON.stringify(value));
  return res.send('Success');
});
app.get('/:key', async (req, res) => {
  const { key } = req.params;
  const rawData = await redisClient.getAsync(key);
  return res.json(JSON.parse(rawData));
});

app.listen(PORT, HOST);
console.log(`Running on http://${HOST}:${PORT}`);
```

I can add some data using `curl` in another shell tab:

```bash
curl -X POST 0.0.0.0:8080/store/tomato?color=red
```

I can retrive that data:

```bash
curl 0.0.0.0:8080/tomato
```

## Conclusion

Cool. Just like that I have a Node API connected to a Redis store to handle some data. All this was done just by installing and using Docker. Nothing else. If you clone this repo and want to run the app, all you have to do after cloning it is run:

```
docker-compose up
```

