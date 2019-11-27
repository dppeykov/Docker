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
 - **docker container exec -it proxy ls** - will just execute the command and exit the container (the container needs to be up and after the command execution it stays up) - exec actually runs an additional process in the container and does not affect the already running process - exec runs additional commands in an existing container


## Docker networks

 - **docker container port webhost** - will check what port is open on the container with the name webhost - REMEMBER: the -p option is used to open ports when creating a container (-p <HOST>:<CONTAINER>)

> When starting a container, we are connecting in the background to a network:
> - By default this is the bridge network (to see the networks use **docker network ls** -> will give **bridge, host and null**). 
> - Each virtual network routes through NAT firewall on the host IP. 
> - All containers on a virtual network can talk to each other without -p option
> - Best practice is to create new virtual networks for each app
> - In docker "the batteries are included, but removable" = the defaults work just fine, but they can be easily modified

 - **docker container inspect --format '{{ .NetworkSettings.IPAddress }}' webhost** - to get the IP of the container network (this will give an IP address in the docker0/bridge network, which is not the host network)
 
> So if we have a container created with -p -> this will attach the container to the bridge/docker0 network and also open a port in the host network, so the traffic can go back and forth from and to the container from the the outside network (the host). This is done with NAT between the bridge and the host networks. 

>**IMPORTANT**: if we just create a conteiner/s without the -p option -> those containers will be able to talk to each other by default as all of them will be attached to the bridge/docker0 network, but they won't be able to talk to other (outside) networks 

> Also we can create more than 1 virtual networks (bridge) - the purpose is to separate the apps we are creating

>**IMPORTANT**: we can't have more than 1 container listening on the same port on a host level. Example: if we have 1 container listening on port 80 on the host, we can't create another container with the same (because the port is already open and in use). 

 - **docker network ls** - bridge (docker0) - the default that bridges through the host NAT firewall (NATed behind the host IP), the host - can attach the container directly to the host network (security concerns + usually imporves the performance), none - like an interface that is not attached to anything - just spawns new virtual network where we can attach containers
 - **docker network inspect bridge** - can show all the containers that are attached to the bridge network
 - **docker network create name_of_the_net** - will create a network with a driver of bridge (use **docker network create --help** for more options)
 
**ROUND-ROBIN EXERCISE:**

 - **docker network create dude** - creates a network with the name dude
 - **docker container run -d --net dude --net-alias search elasticsearch:2** - will create a detached container with random name and will attach it to the dude network + will create an alias search (with the DNS alias we can use the same name and load balance between several containers with the same alias)
 - **docker container run --rm --net dude alpine nslookup search** - to check what containers will resolve the alias name and respond to a request
 - **docker container run --rm --net dude centos curl -s search:9200** - testing with a centos image - the result is that one time the DNS is sending the request to the first and then to the second container (if we have 2) - IMPORTANT: it is not perfect load balancing (e.g. it can send more requests to either of the servers)

## Docker Images

 - **Image** - app binaries and dependencies + metadata about the image data and how to run the image - it's not a complete OS (no kernel, kernel modules, etc.) - can be really small (a single file) or big as a whole OS (multiple GB)
 - **Official definition** - an image is an ordered collection of root filesystem changes and the corresponding execution parameters for use within a container runtime

![alt text](https://docs.docker.com/v17.09/engine/userguide/storagedriver/images/sharing-layers.jpg "Images")

 - **docker history nginx** - old format of the command - will show all the layers of the nginx image - all changes made in an image
 - Every image starts with a blank layer (aka **sctratch**), then every set of changes that happens after that is know as a layer and gets its own unique SHA value (to be able to identify the separate layers)
 - Example Ubuntu image: (Ubuntu -> Apt -> ENV) image
 - Example custom image: (Custom scratch layer -> Existing layer/s -> Our changes)
 - The layers are using SHAs (able to recognize the unique layers), because this allows docker to store a layer only once in the system no matter in how many images we are using it - reduces the disk space
 - **Container** - when we run a new container (e.g. base read only image), all docker does is to **run a new read/write image layer on top** of the existing image
 - **Copy on write (cow)** - if we are changing something in the base image (a file for example), it will simply copy the change (file) into the write the write top layer in the container and then save all the differences (the changed file) in the same layer (the container)

 - **DockerHub** - https://hub.docker.com/ - docker registry with a lot of images - typically we alwas start with the official image (no xxx/ in front of the name -> example nginx VS nginx/unit) - the official images usually have better documentation - we have versioning (stable + edge) organized with tags
 - **docker image ls** - to see what is downloaded on the host 
 - **docker pull nginx** - will download the nginx image - good practice in a production is to use the same images, same versions 
 - **docker image inspect nginx** - showing all of the details about the image (aka metadata) - basic infor like the image ID, tags, etc. + details how the image expects to be ran
 
 - **Image tags** - just labels that are pointing to the same image ID - tags are used for the versions - like :latest, :1.11.0, :mainline etc. - the :latest tag is the default tag
 
 - **docker image tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]** - to create our own tags:
   - Example: **docker image tag nginx dpp/nginx** - will create a new image from nginx with REPOSITORY dpp/nginx and use the latest tag by default (as we didn't specify a tag). If we want to push the image to DockerHub: 
   - **docker image push dpp/nginx** - important is that the dpp repository (account) must exist before we are able to push to dockerhub & we have to login first (docker login/logout commands) - we can check the config.json file for the auth - cat .docker/config.json - with this command we will upload (push) the image repo dpp/nginx with tag :latest

> **IMPORTANT**: if you're using an untrusted machine ALWAYS logout! 

### Creating Docker Images
  
#### Dockerfile Basics

 - The Dockerfile is executed from top to the bottom, so the order in the statements inside matters
 - **FROM** - minimal distribution - like alpine - all images must have FROM
 - **ENV** - optional env vars - a lot of the times used for keys
 - **RUN** - run commands - update, installing software, logs etc. - && chains the commands
 - **EXPOSE** - exposes ports from the container on the docker virtual network
 - **CMD** - required - the command that runs when container is launched
 - **WORKDIR** - changes the working directory = running cd /some/path
 - **COPY** - copy local files into the container

**EXAMPLE FILE**: https://github.com/dppeykov/Docker/blob/master/Dockerfile

**EXAMPLE FILE**: https://github.com/dppeykov/Docker/blob/master/Dockerfile-sample2

- **docker image build -t customnginx .** - will use the Dockerfile in the same directory to build an image with repository customnginx and tag latest (default) - all images mentioned in the Dockerfile will be downloaded and used in the build - on every change in the Dockerfile, we can build the image again and it will use the already cached layers to build it faster

 - **docker image prune** - to clean up just "dangling" images 
 - **docker system prune** - will clean up everything
 - **docker image prune -a** - will remove all images you're not using
 - **docker system df** - to see space usage

Video on prune and df: https://www.youtube.com/watch?v=_4QzP7uwtvI&feature=youtu.be

## Container lifetime and persistent data

 - containers are usually **immutable and ephemeral** (exist only briefly - very short time)
 - **immutable infrastructure** - only **re-deploy containers, never change** - if we need an upgrade or some change, we're re-deploying a whole new container - gives huge benefits in reliability, consistency and making changes reproducable
 - ideally a container **shouldn't contain any unique data (persistent data)** = ensures the **"separation of concerns"** - the unique data is preserved while the container can be recycled 
 - docker has 2 solutions for the persistent data problem - **volumes & bind mounts**
 - **volumes** - make special location outside of the container UFS (Union File System) to store unique data - this allows to attach the data needed to the container/s we need and the container sees it as a local file path 
 
> **IMPORTANT:** volumes need manual deletion, they are not removed with the container

 - **bind mounts** - link container path to host path - sharing (mounting) a host directory or file into a container - again it looks like a local file/directory path from within the container and doesn't look like it is comming from the host
 
### Data volumes

 - **VOLUME** command in Dockerfile - maps the volume on the host with a directory in the container
 - **docker volume ls** - to see what volumes are created - will show the SHAs as volume names
 - **docker volume inspect 47e48** - will show metadata for the volume starting with 47e48 - when it is created, mount point, name, etc.

> **IMPORTANT**: if we delete the containers, the volumes will stay - they will outlive the containers, because those are forms of a persistence storage - even using docker system prune will not delete the volumes that are in use - we need to stop the containers that are using the volumes first and then they can be removed
 



