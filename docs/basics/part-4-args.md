ARGS in Earthly work similar to ARGS in Dockerfiles, however there are few differences when it comes to scope. Also, Earthly has a number ob [built in ARGS]() that are available to use.

Let's say we wanted to have the option to pass in a tag for our Docker image.

```Dockerfile
docker:
    ARG tag='latest'
    COPY +build/go-example .
    ENTRYPOINT ["/go-example/go-example"]
    SAVE IMAGE go-example:$tag
```
In our docker target we can create an ARG called tag. In this case, we give it a default value of `latest`. If we do not provide a defalault value the defalut will be an empty string.

Then, down in our `SAVE IMAGE` we are able to reference the ARG with `$` plus the ARG name.

Now we can take advantage of this when we run Earthly.

```bash
earthly +docker --tag='my-new-image-tag'
```
In this case `my-new-image-tag` will override the default value and become the new tag for our docker image. If we hadn't passed in a value for tag, then the default `latest` would have been used. 

```bash
earthly +docker
# tag for image will be 'latests'
```
## Built in ARGS
TODO

## More Examples
TODO