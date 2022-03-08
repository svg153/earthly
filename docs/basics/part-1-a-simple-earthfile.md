This tutorial focuses on useing Earthly with a Go project, but you can find examples of in [Python](#examples-in-other-languages), [Javascript](#examples-in-other-languages) and [Java](#examples-in-other-languages) at the bottom of each page. 

Below you'll find a simple example of an earthfile. All the magic of Earthly happens in the Earthfile, which you may notice is very similar to a Dockerfile. This is an intentional design decision. Existing Dockerfiles can easily be ported to Earthly by copying them to an Earthfile and tweaking them slightly. Let's take a closer look at the Earthfile. 

```Dockerfile
VERSION 0.6
FROM golang:1.15-alpine3.13
WORKDIR /go-example
 
build:
    COPY main.go .
    RUN go build -o build/go-example main.go
    SAVE ARTIFACT build/go-example /go-example AS LOCAL build/go-example

docker:
    COPY +build/go-example .
    ENTRYPOINT ["/go-example/go-example"]
    SAVE IMAGE go-example:latest
```

Throughtout this tutorial, we'll build up this example earthfile from scratch, and then add even more to it so that by the end you'll hopefully have a better grasp of how Earthly works and the power and repeatability it can bring to your build process.

## Creating an Earthfile

To start we have two files, a `main.go` file:
```go
package main

import "fmt"

func main() {
	fmt.Println("hello world")
}
```
And an Earthfile, which we will build up step by step.

Earthfiles are always named Earthfile, regardless of their location in the codebase. Let's take a look at the first few lines of the Earthfile.
The Earthfile starts off with a version defintion. This will tell Earthly which features to enable and which ones not to, so that the build script maintains compatibility over time, even if Earthly itself is updated.

```Dockerfile
VERSION 0.6
FROM golang:1.15-alpine3.13
WORKDIR /go-example
```

The first commands in the file are part of the `base` target and are implicitly inherited by all other targets. We'll talk more about targets in a bit, but for now, targets are just sets of instructions we can call on from within the Earthfile, or when we run earthly at the command line. Targets need an enviorment to run in. In this case we are saying that all the instructions in our eathfile will run in our `golang:1.15-alpine3.13` 

Lastly we change our working directory to `/go-example`. So far very similar to a docker file.

## Creating Our First Target
Earthly aims to replace Dockerfile, makefile, bash scripts and more. We can take all the setup, configuration and build steps we'd normally define in those files and put them in our Earthfile in the form of `targets`.

Let's start by defining a target to build our simple Go app. (This target may be invoked from the command line as `earthly +build`.)

```Dockerfile
build:
    COPY main.go .
    RUN go build -o build/go-example main.go
    SAVE ARTIFACT build/go-example /go-example AS LOCAL build/go-example
```
The first thing we do is copy our `main.go` from the build context (the directory where the Earthfile resides) to the build environment (the containerized environment where Earthly commands are run).
Next we run a go build command against the previously copied `main.go` file.

Lastly, we save the output of the build command as an artifact. Call this artifact `/go-example` (it can be later referenced as `+build/go-example`). In addition, store the artifact as a local file (on the host) named `build/go-example`. This local file is only written if the entire build succeeds.

Declare a target, `docker`.

```Dockerfile
docker:
    COPY +build/go-example .
    ENTRYPOINT ["/go-example/go-example"]
    SAVE IMAGE go-example:latest
```

You may notice the command `COPY +build/... ...`, which has an unfamiliar form. This is a special type of `COPY`, which can be used to pass artifacts from one target to another. In this case, the target `build` (referenced as `+build`) produces an artifact, which has been declared with `SAVE ARTIFACT`, and the target `docker` copies that artifact in its build environment.

With Earthly you have the ability to pass such artifacts or images between targets within the same Earthfile, but also across different Earthfiles across directories or even across repositories. To read more about this, see the [target, artifact and image referencing guide](../guides/target-ref.md).

Copy the artifact `/go-example` produced by another target, `+build`, to the current directory within the build environment.
Set the entrypoint for the resulting docker image.
Save the current state as a docker image, which will have the docker tag `go-example:latest`. This image is only made available to the host's docker if the entire build succeeds.

## Target Enviorments

Notice how we already had Go installed for both our `+build` and `+docker` targets. This is because  targets inherit from the base target which for us was the `FROM golang:1.15-alpine3.13` that we set up at the top of the file, but it's worth nothing that targets can define their own docker enviorments. For example:

```Dockerfile
VERSION 0.6
FROM golang:1.15-alpine3.13
WORKDIR /go-example

build:
    COPY main.go .
    RUN go build -o build/go-example main.go
    SAVE ARTIFACT build/go-example /go-example AS LOCAL build/go-example

npm:
    FROM node:12-alpine3.12
    WORKDIR /src
    RUN npm install
    COPY assets/ .
    RUN npm test
```

## Running the build

In the example `Earthfile` we have defined two explicit targets: `+build` and `+docker`. When we run Earthly, we can tell it to execute either target by passing the target name. In this case our docker target calls on our build target, so we can run both with:

```bash
earthly +docker
```

The output might look like this:

![Earthly build output](../guides/img/go-example.png)

Notice how to the left of `|`, within the output, we can see some targets like `+base`, `+build` and `+docker` . Notice how the output is interleaved between `+docker` and `+build`. This is because the system executes independent build steps in parallel. The reason this is possible effortlessly is because only very few things are shared between the builds of the recipes and those things are declared and obvious. The rest is completely isolated.

In addition, notice how even though the base is used as part of both `build` and `docker`, it is only executed once. This is because the system deduplicates execution, where possible.

Furthermore, the fact that the `docker` target depends on the `build` target is visible within the command `COPY +build/...`. Through this command, the system knows that it also needs to build the target `+build`, in order to satisfy the dependency on the artifact.

Finally, notice how the output of the build (the docker image and the files) are only written after the build is declared a success. This is due to another isolation principle of Earthly: a build either succeeds completely or it fails altogether.

Once the build has executed, we can run the resulting docker image to try it out:

```
docker run --rm go-example:latest
```

### Examples in Other Languages
<details open>
<summary>JavaScript</summary>

To copy the files for [this example ( Part 1 )](https://github.com/earthly/earthly/tree/main/examples/tutorial/js/part1) run

```bash
mkdir tutorial
cd tutorial
earthly --artifact github.com/earthly/earthly/examples/tutorial/js:main+part1/part1 ./part1
```

`./Earthfile`

```Dockerfile
VERSION 0.6
FROM node:13.10.1-alpine3.11
WORKDIR /js-example

build:
    # In JS, there's nothing to build in this simple form.
    # The source is also the artifact used in production.
    COPY src/index.js .
    SAVE ARTIFACT index.js /dist/index.js AS LOCAL ./dist/index.js

docker:
    COPY +build/dist dist
    ENTRYPOINT ["node", "./dist/index.js"]
    SAVE IMAGE js-example:latest
```

The code of the app might look like this

`./src/index.js`

```js
console.log("hello world");
```

</details>


<details open>
<summary>Java</summary>

`./Earthfile`

```Dockerfile
VERSION 0.6
FROM openjdk:8-jdk-alpine
RUN apk add --update --no-cache gradle
WORKDIR /java-example

build:
    COPY build.gradle ./
    COPY src src
    RUN gradle build
    RUN gradle install
    SAVE ARTIFACT build/install/java-example/bin /bin AS LOCAL build/bin
    SAVE ARTIFACT build/install/java-example/lib /lib AS LOCAL build/lib

docker:
    COPY +build/bin bin
    COPY +build/lib lib
    ENTRYPOINT ["/java-example/bin/java-example"]
    SAVE IMAGE java-example:latest
```

The code of the app might look like this

`./src/main/java/hello/HelloWorld.java`

```java
package hello;

public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("hello world");
    }
}
```

`./build.gradle`

```groovy
apply plugin: 'java'
apply plugin: 'application'

mainClassName = 'hello.HelloWorld'

jar {
    baseName = 'hello-world'
    version = '0.0.1'
}

sourceCompatibility = 1.8
targetCompatibility = 1.8
```

##### Note

To copy the files for [this example ( Part 1 )](https://github.com/earthly/earthly/tree/main/examples/tutorial/java/part1) run

```bash
mkdir tutorial
cd tutorial
earthly --artifact github.com/earthly/earthly/examples/tutorial/java:main+part1/part1 ./part1
```
</details>


<details open>
<summary>Python</summary>
`./Earthfile`

```Dockerfile
VERSION 0.6
FROM python:3
WORKDIR /code

build:
     # In Python, there's nothing to build.
    COPY src src
    SAVE ARTIFACT src /src

docker:
    COPY +build/src src
    ENTRYPOINT ["python3", "./src/hello.py"]
    SAVE IMAGE python-example:latest
```

The code of the app might look like this

`./src/hello.py`

```python
print("hello world")
```

##### Note

To copy the files for [this example ( Part 1 )](https://github.com/earthly/earthly/tree/main/examples/tutorial/python/part1) run

```bash
mkdir tutorial
cd tutorial
earthly --artifact github.com/earthly/earthly/examples/tutorial/python:main+part1/part1 ./part1
```

</details>


