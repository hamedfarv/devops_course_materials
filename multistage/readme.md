Docker Multi-Stage Build Demonstration

This repository contains example Dockerfiles to demonstrate the benefits of multi-stage builds in creating smaller, more efficient Docker images. We will compare a standard build process with a multi-stage approach, focusing on the final image size.

docker build -t test-no-stage-alpine .

docker build -t test-no-stage .

docker build -t test-with-stage-alpine -f Dockerfile-withstaging .

#Comparing Image Sizes

docker images | grep 'test-'
