# docker-builds
Docker files for containerized applications

All Dockerfiles for lab pipelines for applications that do not already exist are maintained here. The build and test of applications should be confirmed individually, but all Dockerfiles hosted here automatically go through a build and test check prior to pushing onto Docker Hub. Only the main branch is published to Docker Hub and the main branch can only be changed through a merged pull request passing CI checks.

The repo for images is currently a [personal repo](https://hub.docker.com/repositories/gptiley), but this might move to an organization if the need grows.
