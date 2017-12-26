# PoC For Docker-Swarm In Digital Ocean

To provision Docker-Machine into Digital Ocean you can 

```
docker-machine create --driver digitalocean --digitalocean-access-token=<access_token> cgondm
``` 

Then you can check the docker-machines with
```
docker-machine ls
``` 

You can check the provisioned docker machine environment with the following command.

```
docker-machine env <docker_machine_name>
PS C:\Users\user\Desktop> docker-machine env cgondm
$Env:DOCKER_TLS_VERIFY = "1"
$Env:DOCKER_HOST = "tcp://104.131.100.107:2376"
$Env:DOCKER_CERT_PATH = "C:\Users\user\.docker\machine\machines\cgondm"
$Env:DOCKER_MACHINE_NAME = "cgondm"
$Env:COMPOSE_CONVERT_WINDOWS_PATHS = "true"
# Run this command to configure your shell:
# & "C:\Program Files\Docker\Docker\Resources\bin\docker-machine.exe" env cgondm | Invoke-Expression
``` 

Run the suggested command to configure the client which will be connecting to the remote docker machine.
```
& "C:\Program Files\Docker\Docker\Resources\bin\docker-machine.exe" env cgondm | Invoke-Expression
```

With executing this command in your PowerShell on windows you are actually connecting to the remote provisioned docker machine like magic.

After connecting to your remote docker machine you can compose up your yml file
```
docker-compose -f prod.yml up -d
```

With this command you compose up the following configuration on the remote cloud infrastructure.
```
version: "3.0"
services:
  dockerapp:
    image: netasankara/poc-docker-ci-py:2cbe0e984298287a46eb0e821bbe83a0c5159c16
    ports:
      - "5000:5000"
    depends_on:
      - redis
  redis:
    image: redis:3.2.0
```

Docker swarm is a tooll that clusters many docker engines and schedules containers.
Docker Swarm decides which host to run the container based on your scheduling methods.

In a docker swarm there are this key elements :

```
Docker Client ---> Docker Swarm (Docker Swarm Manager) -----> (Docker Daemon 1, Docker Daemon 2, Docker Daemon 3, Docker Daemon 4 ...)
```

There are three important concepts in a Docker Swarm :
 
1. **Node :** A node is an instance of docker engine participating in the swarm.
2. **Manager Node :**
3. **Worker Node :**

To deploy your application to a swarm, you submit your service to a manager node.
The manger node dispatches units of work called tasks to worker nodes.
Manager nodes also perform the orchestration and cluster management functions required to maintain the desired state of the swarm.
Worker nodes receive and execute tasks dispatched from manager nodes.
An agent runs on each worker node and reports on the tasks assigned to it. The worker node notifies the manager node of the current state of its assigned tasks so that the manager can maintain the desired state of each worker.

In this read me world we will set up the following :
	
1. Deploy two VM's one will be used for the Swarm manager node, and the other will be used as a worker node. 
2. Appoint the first VM as Swarm manager node and initialize a Swarm cluster
  * ```docker swarm init```
3. Let the second VM join the swarm cluster as a worker node.
  * ```docker swarm join```

First create swarm manager on the Digitail Ocean platform 
```
docker-machine create --driver digitalocean --digitalocean-access-token=<access_token> cgon-swarm-manager
```

This will be provision a new docker machine on the digital ocean

In the next step we will create a new vm for the worker node.
```
docker-machine create --driver digitalocean --digitalocean-access-token=<access_token> cgon-swarm-node
```

After creating two VMs one for swarm manager and one for the worker node we will initialize a swarm cluster.
Before doing it you need to be active in **cgon-swarm-manager** docker machine :

```
PS C:\Users\user\Desktop\poc-docker-swarm\001_Dokcer_Swarm_Digitail_Ocean> docker-machine ls
NAME                 ACTIVE   DRIVER         STATE     URL                          SWARM   DOCKER        ERRORS
cgon-swarm-manager   *        digitalocean   Running   tcp://***.***.***.***:*              v17.11.0-ce
cgon-swarm-node      -        digitalocean   Running   tcp://***.***.***.***:*              v17.11.0-ce
cgondm               -        digitalocean   Running   tcp://***.***.***.***:*              v17.11.0-ce
```
Then issue the following command 

```
docker swarm init --advertise-addr <public_ip>
Swarm initialized: current node (8nakp2p1qzpxlyqi27t1nohnz) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token <token> <manager_ip:port>

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.

PS C:\Users\user\Desktop\poc-docker-swarm\001_Dokcer_Swarm_Digitail_Ocean>
```

In the next step we will add the worker node to the swarm cluster. Before doing it we can actively switch to the cgon-swarm-node or we can ssh into the remote docker-machine.

```
docker-machine ssh cgon-swarm-node
```

And issue the joining command.

```
docker swarm join --token <token> <manager_ip:port>
This node joined a swarm as a worker.
```

RECAP
```
docker swarm init
docker swarm join
docker swarm leave
```

You can scale your applicaiton with the compose file  :
```
version: "3.0"
services:
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '0.1'
          memory: 50M
      restart_policy:
        condition: on-failure
```

With this configuration there will be three nginx instances in the docker swarm.

A docker stack is a group of interrelated serverices that share dependencies, and can be orchestrated and scaled together.
You can image that a stack is a live collection of all the services defined in your docker compose file.

Create a stack from your docker compose file :
```
docker stack deploy
```

**In the Swarm Mode**
  * Docker compose files can be used for service definitions.
  * Docker compose commands cant' be reused. Docker compose commands can only schedule the containers to a single node.
  * We have to use ```docker stack``` command. You can think of ```docker stack``` as the docker compose in the swarm mode.

Let's deploy the following configuration to our swarm :
```
version: "3.0"
services:
  dockerapp:
    image: netasankara/poc-docker-ci-py:2cbe0e984298287a46eb0e821bbe83a0c5159c16
    ports:
      - "5000:5000"
    depends_on:
      - redis
    deploy:
      replicas : 2
  redis:
    image: redis:3.2.0
```

Before deploying make sure your cgon-swarm-manager docker machine is the active docker machine and then issue the following command.
```
docker stack deploy --compose-file prod.yml dockerapp_stack

Creating network dockerapp_stack_default
Creating service dockerapp_stack_dockerapp
Creating service dockerapp_stack_redis

```

We can list all the stacks on swarm cluster bu running the following command.
```
docker stack ls

NAME                SERVICES
dockerapp_stack     2

```

To list the details of the stack you can :
```
docker stack services dockerapp_stack

ID                  NAME                        MODE                REPLICAS            IMAGE                               PORTS
4bzvvmivbplx        dockerapp_stack_dockerapp   replicated          2/2                 netasankara/poc-docker-ci-py:2c**   *:5000->5000/tcp
zarv0dcv98nr        dockerapp_stack_redis       replicated          1/1                 redis:3.2.0
```

You can change the compose file and issue the following command to update the service state on the remote machine.
```
version: "3.0"
services:
  dockerapp:
    image: netasankara/poc-docker-ci-py:2cbe0e984298287a46eb0e821bbe83a0c5159c16
    ports:
      - "5000:5000"
    depends_on:
      - redis
    deploy:
      replicas : 10
  dockerapp_failover:
    image: netasankara/poc-docker-ci-py:2cbe0e984298287a46eb0e821bbe83a0c5159c16
    ports:
      - "4000:5000"
    depends_on:
      - redis
    deploy:
      replicas : 10
  redis:
    image: redis:3.2.0
```

```
docker stack deploy --compose-file prodv2.yml dockerapp_stack

user@cgon-hp MINGW64 ~/Desktop/poc-docker-swarm/001_Dokcer_Swarm_Digitail_Ocean (master)
$ docker stack deploy --compose-file prodv2.yml dockerapp_stack
Updating service dockerapp_stack_dockerapp (id: 4bzvvmivbplxrrp5756iis8tp)
Creating service dockerapp_stack_dockerapp_failover
Updating service dockerapp_stack_redis (id: zarv0dcv98nrnes8oo4kharyc)

user@cgon-hp MINGW64 ~/Desktop/poc-docker-swarm/001_Dokcer_Swarm_Digitail_Ocean (master)
$ docker stack services dockerapp_stack

ID                  NAME                                 MODE                REPLICAS            IMAGE                                                                   PORTS
4bzvvmivbplx        dockerapp_stack_dockerapp            replicated          10/10               netasankara/poc-docker-ci-py:2cbe0e984298287a46eb0e821bbe83a0c5159c16   *:5000->5000/tcp
qbb38v5naj51        dockerapp_stack_dockerapp_failover   replicated          10/10               netasankara/poc-docker-ci-py:2cbe0e984298287a46eb0e821bbe83a0c5159c16   *:4000->5000/tcp
zarv0dcv98nr        dockerapp_stack_redis                replicated          1/1                 redis:3.2.0

```

You can destroy the VMs on Digital Ocean with the following command :
```
docker stack rm dockerapp_stack

docker-machine rm <docker machine name>
```

Make sure you destroy the droplets.

