You may find that you need to run Docker commands inside of a target. For those cases Earthly offers `WITH DOCKER`. `WITH DOCKER` will initialize a Docker daemon that can be used in the context of a `RUN` command. Let's take a look at a few examples. 

Whenever you need to use `WITH DOCKER` we recommend (though it is not required) that you user Earthly's own Docker in Docker (dind) image `earthly/dind:alpine`.

Notice `WITH DOCKER` creates a block of code that has an END keyword. Everything that happens within this block is going to take place within our `earthly/dind:alpine` container.

```Dockerfile
hello:
    FROM earthly/dind:alpine
    WITH DOCKER --pull hello-world
        RUN docker run hello-world
    END

```
You can see in the command above that we can pass a flag to `WITH DOCKER` telling it to pull an image from Docker Hub. We can pass other flags to load in artifacts build by other targets `--load` or even images defined by docker-compose `--compose`. These images will be available within the context of `WITH DOCKER`'s docker daemon.

## A Real World Example

One common use case for `WITH DOCKER` is running integration tests. In this case we might need to set up a database in addition to our application.

TODO: 
1. Learn how to write go tests and get this example to work
2. Write a more detailed breakdown of these files.
3. Create examples for Java, javascript and Python (steal this one from Circle Ci article)

```yml
version: "3.9"
   
services:
  db:
    image: postgres
    container_name: db
    hostname: postgres
    environment:
      - POSTGRES_NAME=postgres
      - POSTGRES_USER=postgres
    ports:
      - 5432:5432
```

```Dockerfile
VERSION 0.6
FROM earthly/dind:alpine
WORKDIR /go-example
RUN wget postgresql-client

build:
    FROM golang:1.15-alpine3.13
    COPY main.go .
    RUN go build -o build/go-example main.go
    SAVE ARTIFACT build/go-example /go-example AS LOCAL build/go-example

docker:
    COPY +build/go-example .
    ENTRYPOINT ["/go-example/go-example"]
    SAVE IMAGE go-example:latest

run-tests:
    COPY ./docker-compose.yml .
    WITH DOCKER --compose docker-compose.yml
        RUN while ! pg_isready --host=localhost --port=5432 dia --username=example; do sleep 1; done ;\
            go test
    END
```
