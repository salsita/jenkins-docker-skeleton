# docker-jenkins-skeleton

This repository contains a small tutorial on how to structure your app so that you can run and test it in Docker in the same way Salsita does.

Salsita uses Jenkins, so the tests are triggered by a push to GitHub, but this should not be really important. Having Docker installed along with a few custom scripts contained in this repository should suffice to run everything.

We often do Node.js + MongoDB, so this tutorial uses this combination as the example.

## The Basic Idea

* The project repository contains a Dockerfile, which defines the image that is supposed to be used for testing. If all the projects share the same base, it's good to create a base image and then just inherit it in the Dockerfile. It's not that important, though, because since the partial results during the image build are cached on the machine locally, then even if your Dockerfile incorporates many steps, they will be executed once per physical machine and then the cache will be used. It will just take a bit more disk space.
* The project repository contains a `prebuild`


