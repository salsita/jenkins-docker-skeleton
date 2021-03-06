# jenkins-docker-skeleton

This repository contains a small tutorial on how to structure your app so that you can run and test it in Docker in the same way Salsita is probably going to do one day.

Salsita uses Jenkins, so the tests are triggered by a push to GitHub, but this should not be really important. Having Docker installed along with a few custom scripts contained in this repository should suffice to run everything.

We often do Node.js + MongoDB, so this tutorial uses this combination as the example.

## The Basic Idea

These are the basic ideas that are behind the whole thing. 

* The project repository contains a Dockerfile, which defines the image that is supposed to be used for testing and potentially running the project. If all the projects share the same base, it's good to create a base image and then just inherit it in the Dockerfile. It's not that important, though, because since the partial results during the image build are cached on the machine locally, then even if your Dockerfile incorporates many steps, they will be executed once per physical machine and then the cache will be used. It will just take a bit more disk space.
* The project repository may contain `setup` script that is run once before the application is started. So if there is anything that cannot be done in the Dockerfile, this is the place where to finish the installation. This step should take care of any initialization that requires the actual project sources, because those are not available when the image is being built. Well, they could be available, if you give us an example where it is necessary :-) So, to put it simply, setup is there instead of a Chef cookbook or so. We don't need any complex setup, so this is enough.
* The project uses [Supervisord](http://supervisord.org/) to manage all the processes that are necessary. This is required by Docker. There must be a single process that can be started and monitored by Docker. And even if the app is not being stared by Docker, it is handy to have a common interface to be able to start it.
* The project repository contains `test` script that runs all the unit tests. There might be multiple scripts to execute different kinds of tests, but for now let's just count with a single script.

Now those are just the core ideas, we are still yet to put them into practice, but it is pretty clear what can be achieved by using this framework - an environment that is quick to boot up and destroy so that it can all happen from within a Jenkins job, from the beginning until the end.

## Example

All the necessary (exemplar) files are or will be in this repository. Everything is instrumented by the Jenkins job mentioned below.

### Jenkins Job

1. Set up custom workspace to be `/tmp/jenkins-buildenv/${JOB_NAME}/${BUILD_NUMBER}/src`.
2. The Shell build step follows.

```bash
# Get around current version permissions issues. Use TCP instead of a UNIX socket.
DOCKER="docker -H tcp://127.0.0.1"

### INIT - Build the directory to be mounted into Docker.
# Create log and db to hold output and db files from the build.
# This is not that much of use for builds, but will be useful for production
# where those files are supposed to be persistent.
MNT="$WORKSPACE/.."
mkdir "$MNT/log"
mkdir "$MNT/db"

### BUILD - Build the Docker image to use for the job.
# A good idea might be to dispose of those cached images regularly, like every day.
# This stuff can be written better, but then it does not work in Jenkins because of how
# streams are being handled.
$DOCKER build . > "$MNT/docker_build.log"
cat "$MNT/docker_build.log"
IMAGE=$(cat "$MNT/docker_build.log" | tail -1 | awk '{ print $NF }')

### RUN
### Execute the build inside Docker.

# Run in the background so that we know the container id.
# Use the image we've just built.
CONTAINER=$($DOCKER run -d -e PROJECT_HOME="/mnt/project" -v "$MNT:/mnt/project" $IMAGE /bin/bash -c 'ls -la "$PROJECT_HOME" && ls -la "$PROJECT_HOME/src"')

# Attach to the container's streams so that we can see the output.
$DOCKER attach $CONTAINER

# As soon as the process exits, get its return value.
RC=$($DOCKER wait $CONTAINER)

# Delete the container we've just used to free unused disk space.
# as well as the temporary mount directory.
$DOCKER rm $CONTAINER
rm -Rf "$WORKSPACE/../../$BUILD_NUMBER"

# Exit with the same value that the process exited with.
exit $RC
```
