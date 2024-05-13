---
layout: post
title:  "Why you might want to avoid building C++ applications w/ `vcpkg`"
date:   2024-05-13 13:25:00 +0200
excerpt_separator: <!--more-->
---

Dockerizing 🐳 a C++ application can be much more difficult than dockerizing a Node.js or Python application. And it is not the execution of the binary that is difficult (that is easy). It's rather that compiling the source code requires a further stage in the Dockerfile, a so-called *build stage*, in which all build tools (`gcc`, `cmake`, etc.) and all dependencies (`boost`, `zlib`, etc.) must be installed and available.

To define these dependencies, one might be tempted to use `vcpkg`, a "C++ Library Manager for Windows, Linux, and MacOS" ([https://github.com/microsoft/vcpkg](https://github.com/microsoft/vcpkg)), which is basically the equivalent of using `npm`+`package.json` in Node.js or `pip`+`Pipfile` in Python, right?

Unfortunately, it is not. This approach leads to several problems.
<!--more-->

1. `vcpkg` feels much clumsier than a `Pipfile`, for example.
2. `vcpkg` proves to be less robust than other package managers.
3. The build step with `vcpkg` is much, much more time-consuming.

Where points 1 and 2 are also somewhat subjective, point 3 can be illustrated relatively easy.

For comparison:

```sh
git clone https://github.com/madietlx/dockerfiles.git
```

```sh
make -C dockerfiles/ubuntu-20-04-compile-cxx-with-vcpkg build
```

downloads a simple example in C++, including a `Makefile` and a `Dockerfile` (build and runtime environment in one), and compiles it. You will get more or less the following output:

```sh
docker build -t built-with-vcpkg:latest .
[+] Building 202.6s (16/16) FINISHED                                                               docker:desktop-linux
 => [internal] load build definition from Dockerfile                                                               0.0s
 => => transferring dockerfile: 1.53kB                                                                             0.0s
 => [internal] load metadata for docker.io/library/ubuntu:20.04                                                    1.6s
 => [internal] load .dockerignore                                                                                  0.1s
 => => transferring context: 2B                                                                                    0.0s
 => [internal] load build context                                                                                  0.0s
 => => transferring context: 1.31kB                                                                                0.0s
 => [stage-1 1/4] FROM docker.io/library/ubuntu:20.04@sha256:80ef4a44043dec4490506e6cc4289eeda2d106a70148b74b5ae9  0.0s
 => => resolve docker.io/library/ubuntu:20.04@sha256:80ef4a44043dec4490506e6cc4289eeda2d106a70148b74b5ae91ee670e9  0.0s
 => => sha256:80ef4a44043dec4490506e6cc4289eeda2d106a70148b74b5ae91ee670e9c35d 1.13kB / 1.13kB                     0.0s
 => => sha256:4aa61d4985265be6d872cc214016f2f91a77b1c925dab5ce502db2edc4a7e5af 424B / 424B                         0.0s
 => => sha256:3048ba0785953b689215053519eb1c34853e2e3af512eed001be59fec1f32e42 2.31kB / 2.31kB                     0.0s
 => [build 2/8] RUN apt-get update   && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommend  16.2s
 => [stage-1 2/4] RUN groupadd -r otto && useradd -r -g otto otto                                                  0.3s
 => [stage-1 3/4] WORKDIR /home/otto                                                                               0.0s
 => [build 3/8] WORKDIR /root                                                                                      0.0s
 => [build 4/8] RUN   wget --progress=dot:giga -O - https://apt.kitware.com/keys/kitware-archive-latest.asc 2>/d  26.4s
 => [build 5/8] RUN   git clone https://github.com/microsoft/vcpkg   && ./vcpkg/bootstrap-vcpkg.sh                65.2s
 => [build 6/8] RUN   vcpkg/vcpkg install     boost-iostreams     boost-program-options                           91.5s
 => [build 7/8] COPY src/ ./src                                                                                    0.0s
 => [build 8/8] RUN cmake -DCMAKE_BUILD_TYPE=Release -S src -B src/build   -DCMAKE_TOOLCHAIN_FILE=/root/vcpkg/scr  1.4s
 => [stage-1 4/4] COPY --from=build /root/src/build/ ./build                                                       0.0s
 => exporting to image                                                                                             0.0s
 => => exporting layers                                                                                            0.0s
 => => writing image sha256:950072b6eb7802f8f30d3fdf09897376b3d6001d972a2471c935cd21ca56eef3                       0.0s
 => => naming to docker.io/library/built-with-vcpkg:latest                                                         0.0s
```

➡️ 202.6 seconds 🐌

For comparison (without `vcpkg`):

```sh
make -C dockerfiles/ubuntu-20-04-compile-cxx-without-vcpkg build
```

```sh
docker build -t built-without-vcpkg:latest .
[+] Building 21.7s (13/13) FINISHED                                                                docker:desktop-linux
 => [internal] load build definition from Dockerfile                                                               0.0s
 => => transferring dockerfile: 546B                                                                               0.0s
 => [internal] load metadata for docker.io/library/ubuntu:20.04                                                    1.3s
 => [internal] load .dockerignore                                                                                  0.0s
 => => transferring context: 2B                                                                                    0.0s
 => [build 1/5] FROM docker.io/library/ubuntu:20.04@sha256:80ef4a44043dec4490506e6cc4289eeda2d106a70148b74b5ae91e  0.0s
 => => resolve docker.io/library/ubuntu:20.04@sha256:80ef4a44043dec4490506e6cc4289eeda2d106a70148b74b5ae91ee670e9  0.0s
 => => sha256:80ef4a44043dec4490506e6cc4289eeda2d106a70148b74b5ae91ee670e9c35d 1.13kB / 1.13kB                     0.0s
 => => sha256:4aa61d4985265be6d872cc214016f2f91a77b1c925dab5ce502db2edc4a7e5af 424B / 424B                         0.0s
 => => sha256:3048ba0785953b689215053519eb1c34853e2e3af512eed001be59fec1f32e42 2.31kB / 2.31kB                     0.0s
 => [internal] load build context                                                                                  0.0s
 => => transferring context: 1.26kB                                                                                0.0s
 => [build 2/5] RUN apt-get update   && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommend  18.7s
 => [stage-1 2/4] RUN groupadd -r otto && useradd -r -g otto otto                                                  0.3s
 => [stage-1 3/4] WORKDIR /home/otto                                                                               0.0s
 => [build 3/5] WORKDIR /root                                                                                      0.0s
 => [build 4/5] COPY src/ ./src                                                                                    0.0s
 => [build 5/5] RUN cmake -DCMAKE_BUILD_TYPE=Release -S src -B src/build   && cmake --build src/build              1.5s
 => [stage-1 4/4] COPY --from=build /root/src/build/ ./build                                                       0.1s
 => exporting to image                                                                                             0.0s
 => => exporting layers                                                                                            0.0s
 => => writing image sha256:2f0a4d8c966239cc521ff559847c9db3c98769f39e9dbe02aeeb6d9098a01118                       0.0s
 => => naming to docker.io/library/built-without-vcpkg:latest                                                      0.0s
```

➡️ 21.7 seconds 🚀

The massive difference is almost entirely due to the time it takes to bootstrap `vcpkg` and build the dependencies. The actual build step with `cmake` is then practically the same in both cases.

The bottom line is that not using `vcpkg` leads to faster builds as well as fewer (third-party) dependencies and is accomplished solely with the package management of the operating system (here: Ubuntu 20.04 (LTS) 🐧), which will be much more robust and less error-prone over the years.

P.S. This article was written in collaboration with a colleague of mine ([@favph](https://github.com/favph) on GitHub), who was the first to point out the disadvantages of `vcpkg` to me that then led to this article. 🙏
