# Docker Notes

> Docker playground: https://labs.play-with-docker.com/

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

## Moving in and out of containers

 - **docker container run -it --name proxy nginx bash** - will create and run a new container with the nginx image and will enter the bash shell (-it = interactive + tty (ssh like)) - to exit Ctrl+C (stops the container)
 - **docker container start -ai proxy** - will connect to the container (-ai = attach + interactive) - we need start as once we are out of the container it will be stopped (if using Ctrl+C to exit)
 - **docker container exec -it proxy ls** - will just execute the command and exit the container (the container needs to be up and after the command execution it stays up) - exec actually runs an additional process in the container and does not affect the already running process
