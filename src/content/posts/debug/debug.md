---
title: My setup for debugging in white-box testing
published: 2025-08-26
description: 'How to debug code flow in python, nodejs, php application'
image: './debugThumb.png'
tags: [debug, setup, vscode]
category: 'Setup'
draft: false 
lang: ''
---

:::warning
This setup have only been tested in Macos, in other operating system I don't know. Also this is just note that I take in Notion, so it's kinda messy
:::


# Nodejs

## Debug Docker Nodejs app in VsCode

`docker-compose.yaml`
```yaml
version: "3.9"
services:
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./deploy/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - app
  app:
    build:
      context: ./deploy/app
    ports:
      - "3000"
      - "9229:9229"  # Debug port
    volumes:
      - ./deploy/app:/usr/src/app
      - /usr/src/app/node_modules
```

`Dockerfile`
```dockerfile
FROM node:17

ENV USER bob
ENV PORT 3000

RUN adduser --disabled-password $USER

RUN apt-get update -y && apt-get install -y curl gcc

WORKDIR /usr/src/app

COPY package*.json ./

RUN npm install

COPY . .
RUN mv my-* ./node_modules

COPY ./flag.c /flag.c
RUN gcc /flag.c -o /flag && \
    chmod 111 /flag && rm /flag.c ./flag.c && \
    chown $USER:$USER /flag

USER $USER
EXPOSE $PORT

//ENV NODE_OPTIONS=--inspect=0.0.0.0:9229
//EXPOSE 9229
CMD [ "node", "--inspect-brk=0.0.0.0:9229", "app.js" ]
// CMD ["nodemon", "app.js"]
//CMD ["node", "app.js"]
```

`.vscode/launch.json`
```json
{
  "version": "0.2.0",
  "configurations": [
      {
          "type": "node",
          "request": "attach",
          "name": "Attach to Docker Container",
          "address": "localhost",
          "port": 9229,
          "localRoot": "${workspaceFolder}/deploy/src",
          "remoteRoot": "/usr/src/app",
          "skipFiles": ["<node_internals>/**"],
          "restart": true
      }
  ]
}
```

`Run`
```bash
docker build . -t image
docker run --rm -p 3000:3000 -p 9229:9229 -it image
  
// Cmd + Shift + D -> Docker: Attach to Node
```

## Debug Nodejs app without Docker in VsCode:

- Create Javascript debug Terminal
- Run node index.js

## Typescript

`Dockerfile`
```dockerfile
npm install -g typescript
npm run build
EXPOSE 3000 9229
CMD ["npm", "start"]
```

`package.json`
```json
"scripts": {
    "start": "nodemon --no-warnings --inspect=0.0.0.0:9229 --loader ./esm-loader.js dist/index.js",
    "build": "tsc",
    "dev": "pnpm build && pnpm start",
    "lint": "eslint ./src --fix"
},
```

`tsconfig.json`
```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "node",
    "strict": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "outDir": "dist",
    "rootDir": "./src",
    "sourceMap": true, 
  },
  "include": ["src/**/*", "@types/**/*"],
  "exclude": ["node_modules", "**/*.spec.ts"]
}
```

`docker-compose.yaml`
```yaml
services:
  db:
    image: mariadb:latest
    environment:
      MYSQL_DATABASE: test
      MYSQL_ROOT_PASSWORD: password
  chall:
    build: .
    ports:
      - "3000:3000"
      - "9229:9229"  # Debug port
    volumes:
      - ./deploy/src:/app/src
      - ./deploy/dist:/app/dist
    depends_on:
      - db
```     

`.vscode/launch.json`
```json
{
    "version": "0.2.0",
    "configurations": [
        {
        "type": "node",
        "request": "attach",
        "name": "Docker: Attach to Node",
        "remoteRoot": "/app",
        "localRoot": "${workspaceFolder}/deploy",
        "port": 9229,
        "restart": true,
        "address": "localhost",
        "outFiles": ["${workspaceFolder}/deploy/dist/**/*.js"],
        "sourceMaps": true,
        "skipFiles": ["<node_internals>/**"],
        }
    ]
}      
```

`Run`
```bash
tsc test.ts | node test.js
// tsc ra folder dist roi moi build docker
```

## Switch multiple nodejs versions: nvm
```bash
nvm ls

nvm install <version>
nvm uninstall <version>

nvm use <version>

nvm exec <version> <command>
nvm run <version> <script.js>
```

# Python
## Debug python without Docker

`.vscode/launch.json`
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Python: Flask",
            "type": "python",
            "request": "launch",
            "module": "flask",
            "env": {
                "FLASK_APP": "./src/app.py",
                "FLASK_DEBUG": "1"
            },
            "args": [
                "run",
                "--no-debugger",
                "--no-reload"
            ],
            "jinja": true,
            "justMyCode": true
        }
    ]
}

```

## Debug Docker flask app

```dockerfile
# Dockerfile
FROM python:3.11-alpine

ENV USER chall
ENV PORT 1116

# Add user
RUN adduser -D -g "" $USER

# Add files
COPY --chown=root:root app /app
COPY --chown=root:$USER flag /flag

WORKDIR /app
RUN chmod 705 run.sh
RUN pip --trusted-host pypi.org --trusted-host files.pythonhosted.org install -r requirements.txt
RUN pip install debugpy

USER $USER
EXPOSE $PORT

# ENTRYPOINT ["/bin/ash"]
# CMD ["./run.sh"]
CMD ["python3", "-m", "debugpy", "--listen", "0.0.0.0:5678", "--wait-for-client", "-m", "flask", "run", "--host=0.0.0.0", "--port=1116"]
```

`docker-compose.yaml`
```yaml
version: "3.9"

services:
  app:
    build:
      context: ./deploy
    networks:
      - internal
    ports:
      - "1116:1116"
      - "5678:5678"  

networks:
  internal:
```
`.vscode/launch.json`
```json
{
    "version": "0.2.0",
    "configurations": [
      {
        "name": "Docker: Python Remote Attach",
        "type": "python",
        "request": "attach",
        "connect": {
          "host": "localhost",
          "port": 5678
        },
        "pathMappings": [
          {
            "localRoot": "${workspaceFolder}/deploy/app",
            "remoteRoot": "/app"  
          }
        ]
      }
    ]
  }
```

## Switch multiple python versions: pyenv
```bash
pyenv versions

pyenv install <version>
pyenv uninstall <version>

pyenv global <version>

pyenv local <version>
```

# PHP
## Debug php app without Docker:

`/opt/homebrew/etc/php/8.4/conf.d/99-xdebug.ini`
```php
zend_extension = xdebug
xdebug.mode=debug
xdebug.start_with_request=yes
xdebug.client_port=9003
xdebug.client_host = localhost
```

`launch.json`
```json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Attach to Process",
            "type": "go",
            "request": "attach",
            "mode": "local",
            "processId": 0
        },
        {
            "name": "Listen for Xdebug",
            "type": "php",
            "request": "launch",
            "port": 9003
        },
        {
            "name": "Launch currently open script",
            "type": "php",
            "request": "launch",
            "program": "${file}",
            "cwd": "${fileDirname}",
            "port": 0,
            "runtimeArgs": [
                "-dxdebug.start_with_request=yes"
            ],
            "env": {
                "XDEBUG_MODE": "debug,develop",
                "XDEBUG_CONFIG": "client_port=${port}"
            }
        },
        {
            "name": "Launch Built-in web server",
            "type": "php",
            "request": "launch",
            "runtimeArgs": [
                "-dxdebug.mode=debug",
                "-dxdebug.start_with_request=yes",
                "-S",
                "localhost:0"
            ],
            "program": "",
            "cwd": "${workspaceRoot}",
            "port": 9003,
            "serverReadyAction": {
                "pattern": "Development Server \\(http://localhost:([0-9]+)\\) started",
                "uriFormat": "http://localhost:%s",
                "action": "openExternally"
            }
        },
        {
            "name": "Remote Xdebug",
            "type": "php",
            "request": "launch",
            "port": 9003,
            "pathMappings": {
              "/www": "${workspaceFolder}/src"
            }
        }
    ]
}
```

## Switch between multiple php version

```bash
brew install php@7.4

echo 'export PATH="/opt/homebrew/opt/php@7.4/bin:$PATH"' >> ~/.zshrc
echo 'export PATH="/opt/homebrew/opt/php@7.4/sbin:$PATH"' >> ~/.zshrc

Change to php 8.3:
Go to ~/.zshrc -> comment export PATH line of the old version
source ~/.zshrc
```

## Debug php app in Docker:
Do the same in this link: https://github.com/dhakalananda/wp-xdebug-docker, then use Dev container extension

## Wordpress

https://patchstack.com/academy/wordpress/getting-started/wordpress-setup/

https://github.com/dhakalananda/wp-xdebug-docker


```jsx

// Create launch.json in the container, use Dev Container extension to set breakpoint
// inside code in the container
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Listen for Xdebug",
      "type": "php",
      "request": "launch",
      "port": 9003,

      // Path mapping is CRITICAL. Map the path PHP executes to the path VS Code sees.
      // Example if Apache serves /var/www/html but your Dev Container workspace is /workspaces/wp:
      "pathMappings": {
        "/var/www/html": "${workspaceFolder}"
      }
    }
  ]
}

```

# Java
I debug in Intellij IDEA because look like everyone is using it(and most tutorial too), also in this [article](https://book.jorianwoltjer.com/languages/java#debugging), the author said Intellij IDEA will give the best experience.

Here's how: Click **Run**(in the top tab of Intellij) -> **Edit Configurations** -> **+** -> **Remote JVM Debug** -> **Copy the command line arguments** then click **OK** to exit

Paste the command line argument to Dockerfile, for ex:
```dockerfile
EXPOSE $PORT 5005
CMD ["java","-Dserver.port=${PORT}","-jar", "-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005" ,"./app.jar"]
```

Then:
- Start the container.
- In IntelliJ, set breakpoints, then Debug the remote config.

Note: Use the same sources locally that produced the JAR so IntelliJ can map line numbers.

Also, if we get the error: "Source code doesn't match bytecode", do this:
- Inside the container: java -version, for example it show Temurin 17
- In IntelliJ Project SDK, pick a Temurin 17 install (same major + ideally same build)

# Docker cheatsheet
## Common commands
```bash
docker build -t imageName .
docker compose up -d 
docker compose up -d --build
docker compose down

docker run --rm -p 5000:5000 -it tag-name
docker run --rm -p 5000:5000 --privileged -it tag-name

docker ps
docker stop <the-container-id>
docker pull
docker image ls

# Bind directory and also run(read an write perm)
docker run -v ~/home/deploy:/app -p 5000:5000 blind
docker run -v ~/home/deploy:/app:rw -p 5000:5000 blind
docker run -v `pwd`:/app -p 8000:8000 bypassif
	# More control
docker run --mount type=bind,source=/HOST/PATH,target=/CONTAINER/PATH,readonly nginx

# Hash mismatch problem - MacOS
RUN echo "Acquire::http::Pipeline-Depth 0;" > /etc/apt/apt.conf.d/99custom && \
    echo "Acquire::http::No-Cache true;" >> /etc/apt/apt.conf.d/99custom && \
    echo "Acquire::BrokenProxy    true;" >> /etc/apt/apt.conf.d/99custom
    
RUN apt-get update -y

# Get file from Docker to host
docker cp "container-id":"path-to-file-in-docker" "path-to-file-in-host"

#Logging in the terminal
docker run -d -v ~/deploy:/app -p 5000:5000 blind
import logging

logging.basicConfig(level=logging.DEBUG)
logging.debug(f"Received request with UID: {uid}")  # Logs to console

# Named volume(don't disappear after the container is stop and deleted)
docker volume create log-data
docker run -d -p 80:80 -v log-data:/logs docker/welcome-to-docker
docker volume ls # List all volumes

```

## Write Dockerfile
```dockerfile
# CMD executed when the container starts and sets the default command to run
# providing default arguments that can be overridden at runtime 
CMD ["python", "app.py"]

docker run <image> bash # Run bash instead of python app.py

# ENTRYPOINT 
# Defining fixed command that should always be executed when the container starts
# Cannot be overridden, but can append additional arguments

ENTRYPOINT ["python", "app.py"]
docker run <image_name> --help # Run python app.py --help
```

## MongoDB
```bash
docker run -d --name mongodb -p 27017:27017 mongo

docker run -d --name mongodb-latest -p 27017:27017 mongo:latest

docker run -d --name mongodb-3.6 -p 27017:27017 mongo:3.6


mongosh
show dbs
use mydb

db.mycollection.insertOne({name: "test"})
```

## MySQL

```bash
docker run --name my-mysql \
    -e MYSQL_ROOT_PASSWORD=secret-pw \
    -e MYSQL_DATABASE=db \
    -e MYSQL_USER=user \
    -e MYSQL_PASSWORD=pass \
    -p 3306:3306 \
    -d mysql:latest

app.config['MYSQL_HOST'] = os.environ.get('MYSQL_HOST', '127.0.0.1')
app.config['MYSQL_USER'] = os.environ.get('MYSQL_USER', 'user')
app.config['MYSQL_PASSWORD'] = os.environ.get('MYSQL_PASSWORD', 'pass')
app.config['MYSQL_DB'] = os.environ.get('MYSQL_DB', 'db')

CREATE DATABASE IF NOT EXISTS db;
CREATE USER IF NOT EXISTS 'user'@'%' IDENTIFIED BY 'pass';
GRANT ALL PRIVILEGES ON db.* TO 'user'@'%';    

docker cp init.sql my-mysql:/init.sql
mysql -u root -psecret-pw < /init.sql

mysql -u root -p
mysql -u root -p db
mysql -u root -psecret-pw db
SHOW TABLES;

```

## Nginx
```bash
docker pull nginx:latest

docker run --name my-nginx -p 80:80 -d nginx

```

# Ref

- https://code.visualstudio.com/docs/containers/debug-common
- https://endy21.hashnode.dev/setup-debug-nodejs-python-java-php
- https://getgrav.org/blog/macos-sequoia-apache-multiple-php-versions
- https://xdebug.org/wizard