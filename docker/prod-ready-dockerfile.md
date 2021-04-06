# Production Grade Dockerfile Guide 🐳

🌏 Updated copy of this guide can be found [here](https://github.com/asjadjawed/devops/blob/main/docker/prod-ready-dockerfile.md).\
*(Pull Requests are welcome)*

## Aims 🎯

***(We will use Node.js environment as an example, for use with other environments modify as needed)***

This is a short guide on how to write production grade Dockerfile. The aim for this guide is to have Dockerfile that are:

- 🔐 *Secure*
- 🗜 *Lightweight*
- 🚀 *Fast*

## The File 💾

A sample dockerfile (multi-stage) we will be analyzing to achieve our aims:

```Dockerfile
# Build stage
FROM node:14.4.0 AS build
LABEL maintainer="mac.dev <mac.dev@company.com>"

USER node
WORKDIR /home/node/app

COPY --chown=node:node package.json yarn.lock ./
RUN yarn install --frozen-lockfile

COPY --chown=node:node src ./src

RUN yarn build

# Run-time stage
FROM node:14.4.0-alpine

USER node
EXPOSE 8080

WORKDIR /home/node/app

COPY --chown=node:node package.json yarn.lock ./
RUN yarn install --frozen-lockfile --production

COPY --chown=node:node --from=build /home/node/app/dist ./dist

CMD [ "node", "dist/app.js" ]
```

💡 We will now breakup this file line by line and understand how it works and avoid common pitfalls and bad practices.

This file is usually located at the root of the folder with the name 'Dockerfile'.

## Multi-Stage Builds 🪜

```Dockerfile
# Build stage
FROM node:14.4.0 AS build

# Run-time stage
FROM node:14.4.0-alpine
```

With multi-stage only necessary files end up in the final image. Dev dependencies aren't needed for running the application application.

Using multi-stage builds resources that are used to building, compiling, testing don't end up in the runtime image. This is the easiest way to cut down extra-weight.

**🚨 Build tools may contain vulnerabilities of their own. Use only what is needed.**

The image is tagged as `build`. This is later referenced when copying the files in the `COPY` command with the `--from` flag.

## Other Tips 🍪

```Dockerfile
LABEL maintainer="mac.dev <mac.dev@company.com>"
```

The `MAINTAINER` tag is deprecated use `LABEL` instead. Enter other info as key-value pairs as needed.