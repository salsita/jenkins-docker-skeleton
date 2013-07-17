# docker-jenkins-skeleton

This repository contains a small tutorial on how to structure your app so that you can run and test it in Docker in the same way Salsita does.

Salsita uses Jenkins, so the tests are triggered by a push to GitHub, but this should not be really important. Having Docker installed along with a few custom scripts contained in this repository should suffice to run everything.

We often do Node.js + MongoDB, so this tutorial uses this combination as the example.

## The Basic Idea

These are the basic ideas that are behind the whole thing. 

* The project repository contains a Dockerfile, which defines the image that is supposed to be used for testing and potentially running the project. If all the projects share the same base, it's good to create a base image and then just inherit it in the Dockerfile. It's not that important, though, because since the partial results during the image build are cached on the machine locally, then even if your Dockerfile incorporates many steps, they will be executed once per physical machine and then the cache will be used. It will just take a bit more disk space.
* The project repository may contain `install` script that is run once before the application is started. So if there is anything that cannot be done in the Dockerfile, this is the place where to finish the installation. This step should take care of any initialization that requires the actual project sources, because those are not available when the image is being built. Well, they could be available, if you give us an example where it is necessary :-)
* The project uses [Supervisord](http://supervisord.org/) to manage all the processes that are necessary. This is necessary because we need a single process that can be started by Docker. And even if the app is not being stared by Docker, we need a common interface to be able to start it.
* The project repository contains `test` script that runs all the unit tests. There might be multiple scripts to execute different kinds of tests, but for now let's just count with a single script.

## Example

### Dockerfile

```
FROM tchap/centos-epel
 
MAINTAINER Ondrej Kupka "ondrejk@salsitasoft.com"
 
# Update packages
RUN yum update -y --exclude=upstart
 
# Install Node.js
RUN yum install -y nodejs npm
 
# Install MongoDB
RUN yum install -y mongodb-server
 
# Install Supervisord
RUN yum install -y python-setuptools
RUN easy_install supervisor
 
# Copy Supervisord files into the image
ADD supervisord.conf /etc/supervisord.conf
ADD supervisord.d    /etc/supervisord.d
 
# Expose 3000
EXPOSE 3000
 
# Run Supervisord on image start
# It could be just "supervisord", but let's get rid of a few WARNing messages.
CMD ["supervisord", "-c", "/etc/supervisord.conf"]
```

You can find a demo `supervisord.conf` and `supervisord.d` in this repository.

### `install` Script

This script will be run inside of the container in the directory containing the project sources. It should contain things like `npm install` to get your Node.js packages and stuff like that.

### `test` Script

TBD

### Jenkins Job

1. Set up custom workspace to be `/tmp/jenkins-buildenv/${JOB_NAME}/workspace` and check `Delete workspace before build starts`.
2. The Shell build step follows.

```
### INIT
### Build the directory to be mounted into Docker.

MNT="$WORKSPACE/.."

# Create log and db to hold output and db files from the build.
# This is not that much of use for builds, but will be useful for production
# where those files are supposed to be persistent.
mkdir "$MNT/log"
mkdir "$MNT/db"

### RUN
### Execute the build inside Docker.

# Have to find out how to name that image ID better...
# In any case, it's CentOS with Node.js and MongoDB...

# Run in the background so that we know the container id.
CONTAINER=$(docker run -d -b "$MNT:/mnt/project" 4dd5fb10358d /bin/bash -c 'ls -la /mnt/project/workspace')

# Attach to the container's streams so that we can see the output.
docker attach $CONTAINER

# As soon as the process exits, get its return value.
RC=$(docker wait $CONTAINER)

# Delete the container we've just used to free unused disk space.
# as well as the temporary mount directory.
docker rm $CONTAINER
rm -R "$MNT/log"
rm -R "$MNT/db"

# Exit with the same value that the process exited with.
exit $RC
```
