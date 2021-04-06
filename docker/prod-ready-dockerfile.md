# Production Grade Dockerfile Guide 🐳

🌏 Up to date copy of this guide can be found [here](https://github.com/asjadjawed/devops/blob/main/docker/prod-ready-dockerfile.md).\
*(Pull Requests are welcome)*

## Aims 🎯

***(We will use Node.js environment as an example, for use with other environments modify as needed)***

This is a short guide on how to write production grade Dockerfile. The aim for this guide is to have Dockerfile that are:

- 🔐 *Secure*
- 🗜 *Lightweight*
- 🚀 *Fast*

## The File 💾

A sample dockerfile (multi-stage) we will be using to understand how to achieve our aims:

```Dockerfile
# Build stage
FROM node:14.16.0 AS build
LABEL maintainer="mac.dev <mac.dev@company.com>"

USER node
WORKDIR /home/node/app

COPY --chown=node:node package.json yarn.lock ./
RUN yarn install --frozen-lockfile

COPY --chown=node:node src ./src

RUN yarn build

# Run-time stage
FROM node:14.16.0-alpine

USER node
EXPOSE 8080

WORKDIR /home/node/app

COPY --chown=node:node package.json yarn.lock ./
RUN yarn install --frozen-lockfile --production && yarn cache clean
#  Alternate command for npm (uses package-lock.json instead of yarn.lock)
# RUN npm ci --production && npm clean cache --force

COPY --chown=node:node --from=build /home/node/app/dist ./dist

CMD [ "node", "dist/app.js" ]
```

💡 We will now breakup this file line by line and understand how it works and avoid common pitfalls and bad practices.

This file is usually located at the root of the folder with the name `Dockerfile`.

## Multi-Stage Builds 🪜

```Dockerfile
# Build stage
FROM node:14.16.0 AS build

# Run-time stage
FROM node:14.16.0-alpine
```

With multi-stage only necessary files end up in the final image. Dev dependencies aren't needed for running the application.

Using multi-stage builds dependencies that are used for building, compiling, testing don't end up in the runtime image. This is the easiest way to cut down extra-weight.

**🚨 Build tools may contain vulnerabilities of their own. Only what is "needed" should be in Prod image.**

The image is tagged as `build` in `FROM` instruction. This is later referenced when copying the files in the `COPY` command with the `--from` flag.

## Use .dockerignore file 👎

A sample .dockerignore file must be located at project root.

```.dockerignore
node_modules
dist
.git
.vscode
coverage
.editorconfig
**/npm-debug.log
README.md
LICENSE
.env
.aws
```

💡 This is critical to avoid copying secrets and reduce build size.

.dockerignore will significantly cut-down the build time. Make sure to exclude unnecessary files & secrets. Some examples given above, customize as needed.

[How to write a .dockerignore file](https://stackoverflow.com/questions/25490911/how-do-you-add-items-to-dockerignore)

## Structuring your project 🌳

With Typescript Node.js project have a transpiling process. For this the project should be stored in such a way that the `build / dist`, `coverage`, `secrets`, `source / src` are arranged in such a way that they can be targeted effectively by `Dockerfile` & `.dockerignore`.

## Correct Images 🏞

```Dockerfile
FROM node:14.16.0 AS build
FROM node:14.16.0-alpine
```

🚨 Use an explicit image version / variant / digest, never refer `latest` or ignore tagging, which defaults to `latest` .

Using the `latest` tag doesn't guarantee the latest version is used (docker caches images), only using an explicit version guarantees that everyone is using the same (latest) version & have consistent builds.

🚨 Large images lead have higher exposure to vulnerabilities (due to extra software installed) and increased space and network bandwidth consumption. Use lean Docker images, such as `Slim` or `Alpine` Linux variants.

💡 For reference selecting the full version may lead to sizes in excess of `1GB` while optimized images can be as small as `100MB - 200MB`. Consider the difference in storing, transfer, loading of these images. The Node.js v14.4.0 Docker image is `~345MB` in size versus `~39MB` for the Alpine version, which is almost `10x` smaller.

## Privilege Escalation 🔐

```Dockerfile
USER node
```

🚨 By default container have the same permissions the `root`.

This is rarely needed if at all. We should use the `node` user within official Node images.

```Dockerfile
WORKDIR /home/node/app
COPY --chown=node:node package.json yarn.lock ./
```

🏠 Subsequent copy and other commands should be used accordingly. The files should be `chown` for user `node` and `home` directory used to store the app. Since `node` can only write to home and can't change the underlying OS.

For other languages check documentation or `/etc/passwd` in the container OS for non-privileged users.

## Use `node` command, avoid `npm start` 🚦

```Dockerfile
CMD [ "node", "dist/app.js" ]
```

Use CMD `['node','server.js']` to start the app, avoid using npm scripts which hinder the passing of OS signals. This prevents problems with child-process, signal handling, graceful shutdown.

🚨 If OS signals aren't passed, it will not shutdown, instead it will be force to quit and will loose current requests and data and may leave open connections to the DB / Caches.

When using Container orchestrators (e.g., Kubernetes ☸), invoke the Node.js process directly don't use custom replication (like PM2, Cluster module). Let K8s handle how many processes are needed, how to spread them and what to do in case of crashes.

🚨 Container that crash will get restarted and K8s won't be able to optimally handle these containers.

[Learn more about process signals inside containers](https://maximorlov.com/process-signals-inside-docker-containers/)

## Efficient caching 💽

Rebuilding a whole docker image from cache can be nearly instantaneous if done correctly.

Less updated instructions should be at start of the Dockerfile and constantly changing (like the source code) should be at the end.

🚨 Wrong ordering will result in very long build time and consume a lot of resources, even if changes are trivial.

## Manage & Cleanup Dependencies before Prod 🧹

```Dockerfile
COPY --chown=node:node package.json yarn.lock ./
RUN yarn install --frozen-lockfile --production && yarn cache clean
```

```Dockerfile
#  Alternate command for npm (uses package-lock.json instead of yarn.lock)
RUN npm ci --production && npm clean cache --force
```

[Why npm ci should be used instead of npm install](https://docs.npmjs.com/cli/v7/commands/npm-ci)

DevDependencies are needed during building and testing. The image that is shipped to production should be minimal and have no development dependencies.

🚨 Only necessary code should be shipped, DevDependencies may have vulnerabilities of their own.

[Example of an attack through es-lint](https://eslint.org/blog/2018/07/postmortem-for-malicious-package-publishes)

Please also ensure that the caches of package managers are also cleaned, these are required for quick re-installation, this isn't required in the case of containers, the caches have an extra copy of dependencies.

[Yarn vs NPM Speed Comparisons](https://shift.infinite.red/yarn-1-vs-yarn-2-vs-npm-a69ccf0229cd)

## Handling Secrets

There are 2 methods for handling secrets in Docker:

1. The best method is using Docker --secret feature (experimental as of July 2020) which allows mounting a file during build time only. Example usage:

    ```Dockerfile
    # syntax = docker/dockerfile:1.0-experimental
    FROM node:14-slim
    WORKDIR /usr/src/app
    COPY package.json package-lock.json ./
    RUN --mount=type=secret,id=npm,target=/root/.npmrc npm ci
    # The rest of the docker file.
    ```

    [YouTube video explaining this feature](https://youtu.be/noHHEzqP6XA)

2. The second uses multi-stage build with args. This still leaves secrets in Docker history.

    ```Dockerfile
    FROM node:14-slim AS build
    ARG NPM_TOKEN
    WORKDIR /usr/src/app
    COPY . /dist
    RUN echo "//registry.npmjs.org/:\_authToken=\$NPM_TOKEN" > .npmrc && \
    npm ci --production && \
    rm -f .npmrc

    FROM build as prod
    COPY --from=build /dist /dist
    CMD ["node","index.js"]

    # The ARG and .npmrc will not appear in the final image 
    # But these can still be found in the Docker daemon un-tagged images list
    ```

## Graceful Shutdown 🔌

🚨 Ensure that the app handles `SIGTERM` event and closes all existing connection and resources.

Shutting down containers is not a rare event, due to the nature of container orchestration this can be a frequent event.

🚨 If not handled properly will result in users disconnecting or data corruption or unneeded connections to the Database.

[Understanding Shutdown of Node Server](https://blog.risingstack.com/graceful-shutdown-node-js-kubernetes/)

## Other Tips 🍪

### Scan Container images

Before using any image ideally scan them for known vulnerabilities. Use tools like [Trivy](https://github.com/aquasecurity/trivy), or [Snyk](https://support.snyk.io/hc/en-us/articles/360003946897-Container-security-overview).

### Use Labels

```Dockerfile
LABEL maintainer="mac.dev <mac.dev@company.com>"
```

`MAINTAINER` is deprecated use `LABEL` instead. Enter other info as key-value pairs as needed (build date, project, etc).

### Prefer `COPY` instead of `ADD` instruction

`COPY` is simpler and only allows copying local files. `ADD` also allows downloading files from URLs. Use the appropriate instruction.

### Avoid base image updates

Updating the container os (e.g. `apt update`) creates inconsistent images (different updates may be downloaded) and requires elevated privileges. Use updated images instead.

### Limit Memory

Always configure a memory limit using both Docker and the Orchestration tool such as K8s or ECS.

🚨 Noisy and misbehaving containers will eat-up resources and starve the system of its resources.

```bash
$ docker run --memory 512m my-node-app
>> Loading program (with memory capped)
```

### General Advice from Docker

[General Advice for Writing Better Dockerfile(s)](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#leverage-build-cache)

Thank you for reading 👍
