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
 
 - **named volumes** - by default the volumes are using SHAs as a name which is not very convinient 
 - **docker container run -d --name mysql -e MYSQL_ALLOW_EMPTY_PASSWORD=True -v mysql-db:/var/lib/mysql mysql** - will create new mysql container with a volume named mysql-db - a lot easier to use
 - in case we need to create a volume ahead of time use **docker volume create** - **docker volume create [OPTIONS]** **[VOLUME]** - used when we need to specify a driver or put a label - except these special cases, usually we are creating the volums with the VOLUME command in the Dockerfile
---
 - **bind mounting** - just a mapping of a host file or directory into a container file or directory
 - basically just 2 locations pointing to the same file/s
 - skips the UFS and host files overwrite any in container - when deleting the container the files on the host are not deleted and if the container use any files on the host and it has its own same files - the host wins and the container is using those files
 - **can't be use in Dockerfile**, must be at **container run** - the format is as follows:
   - ... run -v /Users/user/stuff:/path/inside/the/container (Mac/Linux) --> host:container
   - ... run -v //c/Users/user/stuff:/path/inside/the/container (Windows) --> host:container
 - **docker container run -d --name nginx -p 80:80 -v $(pwd):/usr/share/nginx/html nginx** - will create a nwe container listening on ports 80 on the host and the container, using the nginx, and mapping the current directory where the shell is ($(pwd)) to the /usr/share/nginx/html in the container
---
**Updating a DB example:** 
> TASK: we need to start with postgreSQL 9.6.1 and update to 9.6.2 - we can use named volumes

 - **docker container run -d --name psql -v psql:/var/lib/postgresql/data postgres:9.6.1** --> creates the container psql with the volume psql
 - **docker container logs -f psql** --> to check the logs in the container --> -f = follow --> to see when the container is ready to accept connections (done booting)
 - **docker container stop psql** --> to stop the running container
 - **docker container run -d --name psql2 -v psql:/var/lib/postgresql/data postgres:9.6.2** --> creates another container psql2 using the same volume and updates the container with version 9.6.2 of the image
 - **docker container ls -a** - at this point we have 2 containers 9.6.1 (stopped) and 9.6.2 (running) 
 - **docker volume ls** - the volume is only 1 - psql
 - **docker container logs -f psql2** --> to check the logs in the container and see when it is ready
 
> **NOTE:** this update works for minor versions - for the major version upgrade we need to use a different procedure, as most of the time the volume will not be compatible in format between the major versions
 ---

## Docker compose

 - **Why should we use docker compose?**
   - to configure relationships between containers
   - to save the docker container run settings in an easy-to-read file
   - to create one-liner development environment startups
   
 - Docker compose is comprised of 2 separate but realted parts:
   - 1. YAML - formatted file that describes our solution options for containers, networks and volumes
   - 2. A CLI tool docker-compose used for local dev/test automation with those YAML files
   
 - docker-compose.yml:
   - compose YAML format has its own versions: 1.x, 2.x, 3.x --> currently 3.7 (2019) - https://docs.docker.com/compose/compose-file/
   - the YAML file can be used with docker-compose command for local docker automation or with docker directly in production with Swarm (as of v1.13)

> Check the compose files in this repository.

> Check the documentation for the YAML file syntax: https://docs.docker.com/compose/compose-file/

## Swarm cluster

 - We need ot create several nodes - they can be VMs, BMs, created with docker machine, in docker labs, in the cloud, etc.
 - Setting up docker Swarm - https://docs.docker.com/engine/swarm/swarm-tutorial/ + https://docs.docker.com/engine/swarm/key-concepts/
 - **docker swarm init --advertise-addr \<MANAGER-IP>** - to initialize a dcoker swarm cluster manager node - see the documentation: https://docs.docker.com/engine/swarm/swarm-tutorial/create-swarm/ - will turn on the swarm mode and give back token with which the other nodes (managers/workers) can connect to the cluster - just copy the commands gave back - can be for other manager nodes or other workers
 - **docker swarm join-token worker** - if we want to get the command returned by swarm init - this will give back the token for worker nodes
 - **docker swarm join-token manager** - to get the token for a manager
 - **docker info** - to see the state of the docker swarm cluster
 - **docker node ls** - to see the nodes in the cluster - IMPORTANT: this information is visible only from the manager nodes
 - **docker swarm leave** - if we want a node to leave the cluster - for example if we want a node that was a worker to become a manager, we need to leave the cluster first and then add the node again as a manager
 - **docker service create --replicas 1 --name helloworld alpine ping docker.com** - to create a service helloworld wich is pinging docker.com and has 1 replica
 - **docker service scale helloworld=4** - to scale the application to run in 4 replicas - https://docs.docker.com/engine/swarm/swarm-tutorial/scale-service/
 - **docker service inspect --pretty helloworld** - to inspect the service and see the results in pretty format - **docker service inspect helloworld** - to see the JSON
 - **docker service ps \<SERVICE-ID>** - to see which nodes are running the service - which replica is started on which node - Swarm also shows you the DESIRED STATE and CURRENT STATE of the service task so you can see if tasks are running according to the service definition
 - **docker ps** - to see the running containers as usual
 - **docker service update** - to update a service (rolling update) - check https://docs.docker.com/engine/swarm/swarm-tutorial/rolling-update/


> EXERCISE: Creating a swarm with an overlay network to allow the containers to talk to each other - creating a drupal service on 2 nodes swarm cluster:

  - **docker swarm init --advertise-addr 192.168.0.18** - to initalize the swarm
  - **docker swarm join-token manager** - to joint the other node as a manager
  - **docker info + docker node ls + docker node ps** - to check the swarm cluster
  - **docker network create --driver overlay mydrupal** - to create the overlay network with name mydrupal + **docker network ls** (the ingress overlay network already exists - we are adding 1 more overlay network)
  - **docker service create --name psql --network mydrupal -e POSTGRES_PASSWORD=mypass postgres** - creates a postgreSQL - 1 replica by default + **docker service ls** to check if the servide is running + **docker service ps psql** - to see on which node (will be on node1)
  - **docker container logs psql.1.z3on94s9h56koc7g0cm0zpash** - to see the logs in the container - should say LOG:  database system is ready to accept connections if all is ok
  - **docker service create --name drupal --network mydrupal -p 80:80 drupal** - to create the drupal service and listen on 80 + **docker service ls** to check if the servide is running + **docker service ps psql** - to see on which node (will be on node2)
  - at this point we can see the **drupal install** - follow the process to install - use postgres for DB name, DB username + mypass for DB pass + Advanced options: host = psql (the name of the postgreSQL service) + default port = 5432 - then install in the normal way

## Routing mesh

  - check the documentation here: https://docs.docker.com/engine/swarm/ingress/

> Docker Engine swarm mode makes it easy to publish ports for services to make them **available to resources outside the swarm**. All nodes participate in an **ingress routing mesh. The routing mesh enables each node in the swarm to accept connections on published ports for any service running in the swarm, even if there’s no task running on the node.** The routing mesh **routes all incoming requests to published ports** on available nodes to an active container.

 - all the containers (tasks) using the docker overlay network driver can use the build-in load balancer to distribute the load in a round tobin fashion - if a task (container) for a service is running on only 1 node in the swarm cluster - it still can be reached via all the nodes in the cluster, because of the overlay connectivity
 
## Swarm Stacks and Production Grade Compose

 - **Stacks = Compose files for production** - stacks accept Compose files as their declarative definition for services, networks and volumes
 - **docker stack deploy** - using to create stacks rather than docker service create
 
 ![alt text](https://github.com/dppeykov/Docker/blob/master/Swarm_stack.png "Swarm Stack")
 

