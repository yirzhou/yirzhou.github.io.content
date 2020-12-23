---
title: "Perfect Sandbox Environment for C++ Development"
date: 2020-12-15
draft: false
toc: false
tags:
  - cpp
  - cmake
  - container
  - vscode
  - technical
---

> This is one of the most valuable skills that I learned in my current internship. We all love VSCode but this just blew me away.

## 1. Motivation

As a software engineer working with various languages and their ecosystems, it's easy to mess up your development machine when installing dependencies directly on it. Even when we just want to try out a new language, we still need to have that language runtime installed somewhere on our system.

Ideally, we want to keep it separate such that it will leave the rest of our system intact.

## 2. My story

My current work is entirely C++-based: I write code in C++ and I build my project using [CMake](https://www.cmake.org) and [Ninja](https://ninja-build.org/). I develop on a MacBook Pro but my project targets Linux. C++ on MacOS is different from Linux in terms of their compilers and standard libraries: Apple develops their own toolchains using LLVM and uses Clang as its compiler, while most Linux distributions use GCC. There are subtle differences between their standard libraries as well in terms of supporting the newer C++ standards.

Most people should never worry about those nuances, but they happened to cause an obstacle for me: my project could only build on Linux because some C++17 features were missing in the Clang standard library (`libstdc++`).

For a while, I was working on a remote development machine using SFTP and the command line. It was fairly unproductive: SFTP heavily depends on how fast my network is, and building and tracing bugs down in the command line require very sharp eyes which I don't have :sweat_smile:. Also, I don't have all the features in VSCode properly set up when developing remotely. In short, it was a real struggle for me.

## 3. How about containerization?

You might have been laughing at my naïveté for a while now: why not working in a [Docker container](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/container-docker-introduction/)? We can run any operating system and install any dependencies in an isolated environment.

In fact, I did try developing inside a container locally, which got rid of SFTP entirely. However, I was still using the clunky CLI and missing the "Intellisense" and all the other goodies of [VSCode](https://code.visualstudio.com/).

It all changed when I was introduced to [Remote-Containers](https://code.visualstudio.com/docs/remote/containers).

Remote-Containers is a [VSCode extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) that lets me develop inside a Docker container, using VSCode, as a full-featured development environment. That means that I can enjoy all the features that I need while developing inside a container:

- Full Intellisense and auto-completion with C++ STL
- CMake extensions
- Clang-format for code formatting on save

You could [read more about it](https://code.visualstudio.com/docs/remote/containers) if you are interested, and I highly encourage you to get on board. It essentially allows us to open a container in VSCode as a regular file system, and install our favourite extensions and configure settings in a isolated environment.

## 4. Setup

In the following sections, I will walk through my routine setup for C++ development using VSCode Remote-Containers; however, this skill can be transferred to any other languages or technology stack. I assume that the reader is familiar with container

### 4.1 Pre-requisites

- Docker Engine
- Visual Studio Code

### 4.2 `.devcontainer.json`

First, we need to define how VSCode can build and open up a container in a special file: `.devcontainer.json`. This file can be standalone in your root project directory, or in a separate `.devcontainer` directory.

We can set values of plenty of [properties](https://code.visualstudio.com/docs/remote/devcontainerjson-reference) that define the locations of `docker-compose.yml`, `Dockerfile`, on-init and post-init commands, extensions we wish to install, and many more. For me, I have the following setup:

```json
{
  "dockerComposeFile": ["../docker-compose.yml", "docker-compose.override.yml"],
  "initializeCommand": "mkdir -p debian && cat */debian/control > debian/control",
  "service": "dev-env",
  "workspaceFolder": "/workspace",
  "extensions": [
    "ms-vscode.cpptools",
    "ms-vscode.cmake-tools",
    "twxs.cmake",
    "ryanluker.vscode-coverage-gutters",
    "pucelle.run-on-save",
    "xaver.clang-format"
  ],
  "settings": {
    "http.proxyStrictSSL": false,
    "C_Cpp.default.includePath": ["/usr/include", "/workspace/**"],
    "C_Cpp.default.cStandard": "c11",
    "C_Cpp.default.cppStandard": "c++17",
    "C_Cpp.default.intelliSenseMode": "gcc-x64",
    "C_Cpp.updateChannel": "Default",
    "clang-format.style": "Google",
    "clang-format.fallbackStyle": "LLVM"
  }
}
```

I put my `.devcontainer.json` in the `.devcontainer` directory, and I have two `docker-compose` files. Some notable areas are:

- `"workspaceFolder": "/workspace"` specifies the workspace directory inside the container. I will be running Ubuntu, and it will locate me in `/workspace`. I will also mount my project to this directory later.
- `extensions` specifies a list of VSCode extensions (C++-specific) to install. Note that those extensions will be separate from our editor extensions on our machine and are exclusive to this container.
- `settings` specifies container-specific VSCode settings.

### 4.3 `docker-compose.yml`

[Docker Compose](https://docs.docker.com/compose/) enables defining and running **multi-container** Docker applications. I personally think that its main feature is to spin up containers in an order if there are dependencies between them. For the purpose of this setup, I will only spin up one container.

```yaml
version: "3.7"
services:
  dev-env:
    build:
      context: .
      dockerfile: dev.Dockerfile
    volumes:
      - .:/workspace:z
    working_dir: /workspace
    environment:
      - CTEST_OUTPUT_ON_FAILURE=1
      - GTEST_COLOR=1
      - CMAKE_GENERATOR=Ninja
```

This file has minimal configurations: the most important part is `volumes` that mounts my project directory to `/workspace` in the container. I also set some C++-specific environment variables for CMake.

### 4.4 `Dockerfile`

The last missing piece is the `Dockerfile` (in my case, `dev.Dockerfile`) which defines the build steps of my container:

```Dockerfile
FROM ubuntu:latest

LABEL description="Development environment workspace"

ENV TZ=Etc
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

RUN echo "root:docker" | chpasswd

RUN apt-get update && \
    apt-get install -y --force-yes \
    build-essential \
    clang-format \
    devscripts \
    equivs \
    g++ \
    gdb \
    libssl-dev \
    ninja-build \
    openssh-server \
    rsync && \
    apt-get clean

# Copy just the control file.
COPY ./debian/control /tmp/debian/control

# Install project build dependencies
RUN mk-build-deps -i -t "apt-get -o Debug::pkgProblemResolver=yes --no-install-recommends -y" /tmp/debian/control && \
    apt-get clean

# install quantum
WORKDIR /opt/
RUN wget https://github.com/bloomberg/quantum/archive/v2.1.tar.gz && \
    tar -zxvf v2.1.tar.gz && \
    cd quantum-2.1 && \
    cmake -Bbuild DQUANTUM_ENABLE_TESTS=ON . && \
    cd build && \
    make install


EXPOSE 22

CMD ["/usr/sbin/sshd", "-D"]

```

There's not much to say about this file - you can run different runtimes or operating systems of your choice.

## 5. How it feels like

In short, it's **awesome**.

I can open my project in a container:

![Open project in container](/gif/open_in_container.gif)

My C++ extensions are working as expected - I get Intellisense, CMake, and Clang-format:

![C++ Intellisense in container](/gif/cpp_intellisense.gif)

I can build using CMake (non CLI):

![CMake extension in container](/gif/cmake.gif)

What helps me the most, is navigating through errors - I never need to go into the CLI any more:

![Navigate code base in VSCode](/gif/error_navigation.gif)

I don't use a lot of tools for my C++ development setup, but I have already realized how amazing it is for my productivity. If your tech stack requires more tools (I'm aware that Java and JavaScript have much richer ecosystems), you are going to appreciate it more.

## 6. Final thoughts

When I started my internship, I failed to realize its significance in my productivity until much later. This is such an elegant approach to solve dependency issues while retaining all the goodies of VSCode, and it demonstrates perfectly the principle of _using the right tools to solve the right problems_.

There are some imperfections about this setup, but most of them are not related to VSCode but to containerization:

1. If you are using a corporate VPN at work, you should consult with your IT department to circumvent it with proxy configurations in order to download VSCode server and extensions. It's not an issue, but definitely something to keep in mind.
2. If the project is huge, you may need to allocate more memory to the container. In my experience, GCC failed during compilation because of the insufficient RAM.

I wish I would know it earlier, but I'm still glad that I do now!
