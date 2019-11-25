# Docker Notes

> The containers are not mini VMs, they are just processes with limited resources

## Check the version of the docker CLI and engine

 - **docker version** - verifies that the cli can talk to the engine
 - **docker info** - most of the config values of the engine
 
## Docker command line structure 

 - **old** - docker <command> (options)
 - **new** - docker <command> <sub-command> (options)

## Basic commands

 - **docker container run --name webhost -p 8080:80 -d nginx** - creates a container with the name webhost, detached (running in the background), opening port 8080 on the host and listening on 80 in the container (<HOST>:<CONTAINER>) from the nginx image (if the image is not on the system it will be downloaded from docker hub - it will download nginx:latest as the release is not specified in this example)
 - **docker container top** - process list in one container
 - **docker container inspect** - details of one container config
 - **docker container stats** - performance stats for all containers
 - **docker container run -it** - start new container interactively
 - **docker container exec -it** - run additional command in existing container
 - **docker container port <container>** - quick port check
