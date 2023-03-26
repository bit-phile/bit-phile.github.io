---
title: "Docker Compose Example"
date: 2023-03-26T13:25:14+05:30
draft: true
tags: []
author: 'Nitin'
comments: false
---


## Setting up an application

Before moving ahead, let's fulfil some perquisites. Please clone this [project](https://github.com/nitinsharmacs/bitphile-blogs-examples) as we will be understanding docker compose through this example.

After clone, move to `docker/docker-compose` directory.

```shell
⟩ tree                                                                    (base)
.
├── Dockerfile
├── README.md
├── index.js
├── package-lock.json
└── package.json

0 directories, 5 files
```

Cool. Let's start defining a minimal `docker-compose.yml` for our small project.

```yml
name: todo-backend

services:
	app:
		build:
			dockerfile: Dockerfile
			context: .	
```

Save it in `docker-compose.yml` and run the following command.

```shell
⟩ docker compose up                                                       (base)
[+] Building 0.1s (10/10) FINISHED                                                                                          
 => [internal] load build definition from Dockerfile                                                                   0.0s
 => => transferring dockerfile: 200B                                                                                   0.0s
 => [internal] load .dockerignore                                                                                      0.0s
 => => transferring context: 60B                                                                                       0.0s
 => [internal] load metadata for docker.io/library/node:12-alpine                                                      0.0s
 => [1/5] FROM docker.io/library/node:12-alpine                                                                        0.0s
 => [internal] load build context                                                                                      0.0s
 => => transferring context: 1.75kB                                                                                    0.0s
 => CACHED [2/5] WORKDIR /app                                                                                          0.0s
 => CACHED [3/5] COPY package.json package.json                                                                        0.0s
 => CACHED [4/5] RUN npm install                                                                                       0.0s
 => CACHED [5/5] COPY index.js index.js                                                                                0.0s
 => exporting to image                                                                                                 0.0s
 => => exporting layers                                                                                                0.0s
 => => writing image sha256:c41c8b0bf0350e4d8f7b6d0151f5232badf5404febc644e2e7fc8389ffebeb28                           0.0s
 => => naming to docker.io/library/todo-backend_app                                                                    0.0s
[+] Running 2/2
 ⠿ Network todo-backend_default  Created                                                                               0.1s
 ⠿ Container todo-backend-app-1  Created                                                                               0.1s
Attaching to todo-backend-app-1
todo-backend-app-1  | 
todo-backend-app-1  | > backend@1.0.0 start /app
todo-backend-app-1  | > node .
todo-backend-app-1  | 
todo-backend-app-1  | (node:19) UnhandledPromiseRejectionWarning: Error: getaddrinfo ENOTFOUND mysql-server
todo-backend-app-1  |     at Object.createConnection (/app/node_modules/mysql2/promise.js:242:31)
todo-backend-app-1  |     at Object.<anonymous> (/app/index.js:43:4)
todo-backend-app-1  |     at Module._compile (internal/modules/cjs/loader.js:999:30)
todo-backend-app-1  |     at Object.Module._extensions..js (internal/modules/cjs/loader.js:1027:10)
todo-backend-app-1  |     at Module.load (internal/modules/cjs/loader.js:863:32)
todo-backend-app-1  |     at Function.Module._load (internal/modules/cjs/loader.js:708:14)
todo-backend-app-1  |     at Function.executeUserEntryPoint [as runMain] (internal/modules/run_main.js:60:12)
todo-backend-app-1  |     at internal/main/run_main_module.js:17:47
todo-backend-app-1  | (node:19) UnhandledPromiseRejectionWarning: Unhandled promise rejection. This error originated either by throwing inside of an async function without a catch block, or by rejecting a promise which was not handled with .catch(). To terminate the node process on unhandled promise rejection, use the CLI flag `--unhandled-rejections=strict` (see https://nodejs.org/api/cli.html#cli_unhandled_rejections_mode). (rejection id: 1)
todo-backend-app-1  | (node:19) [DEP0018] DeprecationWarning: Unhandled promise rejections are deprecated. In the future, promise rejections that are not handled will terminate the Node.js process with a non-zero exit code.
todo-backend-app-1 exited with code 0
```

🙄, it bombarded us with error. Let's see what this error is. If we see, the app started to run but it has a dependency of `mysql` service which is not there. 
