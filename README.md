## Rust Auth Microservice

The auth service will have three primary features:
1. Sign in
2. Sign up
3. Sign out

## Objective
In this project, we aim to learn and practice the following:
* Designing, building, and deploying microservices
* Using gRPC to communicate between microservices
* Monitoring the health of microservices
* Setting up continuous integration & continuous deployment
* Using session based authentication
* Writing testable code
* Organizing code using modules
* Navigating and contributing to an existing code base

## Terminologies

__Session based authentication__

[Session based auth](https://www.geeksforgeeks.org/session-vs-token-based-authentication/) works by giving the client a session token which can be used in subsequent requests to authenticate the user.

__Microservices__

[Microservices](https://microservices.io/) is an architectural style that structures an application as a collection of services that are independently deployable, loosely coupled, organized around business capabilities, and owned by a small team.

__CI/CD__

[CI/CD](https://www.redhat.com/en/topics/devops/what-is-ci-cd) (Continuous Integration/Continuous Delivery or Continuous Deployment) is a set of practices and techniques that help software development teams deliver high-quality software faster and more reliably. Continuous Integration refers to the process of frequently merging code changes from multiple developers into a central repository and running automated tests to detect any integration issues early on. Continuous Delivery/Deployment takes this a step further, automating the entire software release process, from building and testing to deploying the application to production. These practices help teams deliver software more frequently and with higher quality, reducing time-to-market and increasing customer satisfaction.

## CI/CD Tools

There are the tools we'll be using to setup CI/CD.

### Github Actions

[GitHib Actions](https://docs.github.com/en/actions) allow you to execute arbitrary workflows by simply adding a `YAML` file to your repository.

Here is a great overview video: https://youtu.be/eB0nUzAI7M8

### Docker

[Docker](https://www.docker.com/) is a platform for building, running, and shipping applications in containers. 

Containerization is a technology that allows developers to package an application with all of its dependencies into a standardized unit, called a container, which can be easily deployed across different environments, including local machines, data centers, and cloud providers. Containers are lightweight, portable, and secure, enabling teams to build and deploy applications faster and more reliably.

Docker images are the blueprints or templates used to create Docker containers. An image contains all the necessary files, libraries, and dependencies required to run an application. A container, on the other hand, is a running instance of an image. It's a lightweight, isolated environment that runs the application and its dependencies. Multiple containers can be created from the same image, each with its own unique state and running independently.

Here is a great overview video: https://youtu.be/Gjnup-PuquQ

### DigitalOcean

[DigitalOcean](https://www.digitalocean.com/) is a cloud computing platform that provides virtual servers (called "Droplets") and other infrastructure services to developers and businesses. It offers a simple and intuitive user interface, along with flexible pricing plans that allow users to pay only for what they use. DigitalOcean supports a wide range of operating systems and application stacks, making it a popular choice for hosting web applications, databases, and other workloads.

Here are 2 great overview videos: https://youtu.be/goiq9PZLlEU & https://youtu.be/HODYl1KffDE

__Configuring Docker__

__File overview__

`.dockerignore`

Similar to `.gitignore` this tells Docker which files/directories it should ignore.

---

`Dockerfile-auth` & `Dockerfile-health`

These are the Docker files for our two services. 

A Dockerfile is a script that contains instructions to build a Docker image. It specifies the base image to use, adds application code, and sets up configuration options. By running a Dockerfile, developers can automate the creation of Docker images, making it easier to deploy and scale applications.

Let's take a look at the contents of `Dockerfile-auth`:

```docker
FROM rust:1.68.2-alpine3.17 AS chef
```

We start with the official Rust image which has all the dependencies we need to build a Rust project.

```docker
RUN apk add --no-cache musl-dev & cargo install cargo-chef
```

Then we install `musl-dev` and [cargo-chef](https://crates.io/crates/cargo-chef). [musl](https://musl.libc.org/) is an implementation of the C standard library built on top of the Linux system call API. [We need it for `cargo-chef` to work](https://github.com/LukeMathWalker/cargo-chef/blob/5f791e86e87db5bf3add5be5a91e0d06b03c42b4/docker/Dockerfile#L6). `cargo-chef` allows us to cache dependencies in our Rust project and speed up your Docker builds.

```docker
FROM chef AS planner
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

FROM chef AS builder
COPY --from=planner /microservice-project/recipe.json recipe.json
# Build dependencies - this is the caching Docker layer!
RUN cargo chef cook --release --recipe-path recipe.json
# Build application
RUN apk add --no-cache protoc
COPY . .
RUN cargo build --release --bin auth
```

Then we run `cargo-chef`, install `protoc`, and build the service  in release mode.

```docker
FROM debian:buster-slim AS runtime
WORKDIR /microservice-project
COPY --from=builder /microservice-project/target/release/auth /usr/local/bin
ENTRYPOINT ["/usr/local/bin/auth"]
```

Finally we create a new bare-bones image, copy over the binary we created in the previous step, and execute it! One of the advantages of Rust is that our apps can be compiled down to a single binary.

`docker-compose.yaml`

[Docker Compose](https://docs.docker.com/compose/) is a tool for defining and running multi-container Docker applications. With Compose, you use a YAML file to configure your application’s services. A service is a high-level concept that refers to a set of containers running the same image. Then, with a single command, you create and start all the services from your configuration.

In our case we need to define an auth service and a health check service:

```yaml
version: "3.9"
services:
  health-check:
    image: letsgetrusty/health-check # specify name of image on Docker Hub
    build: # specify which Docker file to use
      context: .
      dockerfile: Dockerfile-health
    restart: "always" # automatically restart container when server crashes
    depends_on: # ensure that `health-check` starts after `auth` is running
      auth:
        condition: service_started
  auth:
    image: letsgetrusty/auth # specify name of image on Docker Hub
    build: # specify which Docker file to use
      context: .
      dockerfile: Dockerfile-auth
    restart: "always" # automatically restart container when server crashes
    ports:
      - "50051:50051" # expose port 50051 so that applications outside the container can connect to it 
```

---

__Setting up continuous integration__

We will add a GitHub Actions workflow that builds and tests your code before it is merged into master.

Add the `.github/workflow/prod.yml` file to your repository. 

This file defines the workflow. Let's go through it:

```yaml
on:
  push:
    branches:
      - master
   pull_request:
    branches:
      - master
```

First we define when we want our workflow to run. In this case we want to run the workflow whenever code gets pushed to master or when a pull request targeting the master branch is created.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Install protoc
      uses: arduino/setup-protoc@v1

    - name: Checkout code
      uses: actions/checkout@v2
```

Then we define each step in our `build` job. First we install protoc and checkout the source code.

```yaml
    - name: Cache dependencies
      uses: actions/cache@v3
      with:
        path: |
          ~/.cargo
          target/
        key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}
        restore-keys: ${{ runner.os }}-cargo-
```

Then we use the cache action to cache the `/.cargo` and `target/` directories. This will decrease compile times between runs.

```yaml
    - name: Install Rust
      uses: actions-rs/toolchain@v1
      with:
        profile: minimal
        toolchain: stable

    - name: Build and test code
      run: |
        cargo build --verbose
        cargo test --verbose
```

Finally we build and test the app.

After adding `.github/workflow/prod.yml` to your repository and pushing the changes up to master, you should see your workflow execute.