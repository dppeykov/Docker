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
