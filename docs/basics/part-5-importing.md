So far we've seen how the FROM command in Earthly had the ability to referance another target's image as its base image.

```Dockerfile
VERSION 0.6
FROM golang:1.15-alpine3.13
WORKDIR /go-example
 
build:
    COPY main.go .
    RUN go build -o build/go-example main.go
    SAVE ARTIFACT build/go-example /go-example AS LOCAL build/go-example

test:
    FROM +build
    RUN go test
```
Here are `+test` target uses the image from the `+build` target.

But FROM also has the ability to import targets from Earthfiles in differnet directorys

```Dockerfile
test:
    FROM ./services/service-one+build
    RUN go test
```
This code tells FROM that there is another Earthfile in inside of the services/service-one directory and that that Earthfile contians a target called `+build`. In this case, if we were to run `+test` Earthly is smart enough to go into the subdirectoy, run the target in that Earthfile, and then use it as a the base image here.

We can also reference an Earthfile in another repo.

```Dockerfile
test:
    FROM github.com/example/project+remote-target
    RUN go test
```
