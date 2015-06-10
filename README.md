ecs-docker
======================
ecs-docker is a tutorial on using EMC's ECS as a multi-node Docker container setup using Docker Swarm, Docker Machine and Docker Compose.

## Description
[EMC Elastic Cloud Storage (ECS)](https://www.emc.com/storage/ecs-appliance/index.htm?forumID) is a software-defined cloud storage platform that exposes S3, Swift, and Atmos endpoints. This walk-through will demonstate the setup of a 3 Node ECS Cluster using Docker containers and Docker tools.

## Requirements
* Each ECS node requires 16GB of RAM and an attached volume with a minimum of 512GB. The attached volume stores persistent data that is replicated between  three nodes.
* [Docker Machine](https://docs.docker.com/machine/) installed on your local laptop
* [Docker Compose](https://docs.docker.com/compose/) installed on your local laptop
* [Docker Client](https://docs.docker.com/machine//) installed on your local laptop. (follow Docker Client installation directions)

This is for test and development purposes. Not for production use. Please reach out to your local EMC SE for information on production usage. EMC ECS ships as a commodity storage appliance for production use.

## Lets Get Going

#### Setup 3 Ubuntu Hosts with Docker Swarm using Docker Machine
The following examples use Ubuntu. This is a requirement for using the setup scripts.

[Docker Machine](https://docs.docker.com/machine/) examples are shown with the AWS driver with the standard Ubuntu AMI. However, any cloud or compatible infrastructure with Docker Machine can be used.

1. Create a [Docker Swarm](https://docs.docker.com/swarm/) ID. 
    1. This can be done by using `docker run swarm create` on any host that can run Docker. If no host is available, use Docker Machine to create the host: `docker-machine -D create --driver amazonec2 --amazonec2-access-key MYKEY --amazonec2-secret-key MYSECRETKEY --amazonec2-vpc-id vpc-myvpc swarm-create`
    2. Connect to the Docker Machine: `docker-machine env swarm-create`
    3. Get shell access: `eval "$(docker-machine env swarm-create)"`
    4. Run the Swarm container: `docker run swarm create`
    5. The output will have a unique id such as `b353bb30194d59ab33e4d47c012ee895`.
2. Create a [Docker Swarm](https://docs.docker.com/swarm/) 3 node cluster using [Docker Machine](https://docs.docker.com/machine/). Each node requires:
    - a root drive of >=20GB
    - 16GB RAM or more
    - A Swarm token

    1. Create Swarm Master: `docker-machine -D create --driver amazonec2 --amazonec2-access-key MYKEY --amazonec2-secret-key MYSECRETKEY --amazonec2-vpc-id vpc-myvpc --amazonec2-instance-type m3.xlarge --amazonec2-root-size 50 --swarm --swarm-master --swarm-discovery token://b353bb30194d59ab33e4d47c012ee895 swarm-master`
    2. Create Node 01: `docker-machine -D create --driver amazonec2 --amazonec2-access-key MYKEY --amazonec2-secret-key MYSECRETKEY --amazonec2-vpc-id vpc-myvpc --amazonec2-instance-type m3.xlarge --amazonec2-root-size 50 --swarm --swarm-discovery token://b353bb30194d59ab33e4d47c012ee895 swarm-node01`
    3. Create Node 02: `docker-machine -D create --driver amazonec2 --amazonec2-access-key MYKEY --amazonec2-secret-key MYSECRETKEY --amazonec2-vpc-id vpc-myvpc --amazonec2-instance-type m3.xlarge --amazonec2-root-size 50 --swarm --swarm-discovery token://b353bb30194d59ab33e4d47c012ee895 swarm-node02`
    
- note:The `-D` flag is used to look at diagnostics. It's not necessary.

#### Add A Volume to Each Node
Docker containers, by nature, are used for stateless applications. By attaching an outside volume (not the root volume), data persistence is achieved. The container running the ECS software is ephemeral and the volume containing the data can be attached to any ECS container. All ECS nodes replicate data to one another and store logs in this attached Volume. The volume needs to be >=512GB.

After the volume has been added, retrieve the mounted path. For instance, AWS uses `/dev/xvdf`. This can be retrieved by using `docker-machine ssh swarm-master "sudo fdisk -l"`, which shows the disk layout for the `swarm-master` machine.

This example shows how to attach a volume using AWS. 

  1. Login to the AWS Console
  2. Navigate to **EC2 -> Volumes -> Create Volume**
  3. Set the size to **512** or greater
  4. Set the **Availability Zone** to where your Docker Machine Hosts reside.
  5. Repeat this 2 more times so there are 3 Volumes
  6. Select 1 volume and navigate to **Actions -> Attach Volume**
  7. Attach 1 unique volume per Docker Machine Swarm host. Do not attach all three volumes to all three hosts.

#### Prepare the hosts
Each host requires a script to be ran that prepares the volumes attached as XFS and builds a multitude of folders and permissions.

1. Using Docker Machine, download the setup script from this repo
    ```
    docker-machine ssh swarm-master "curl -O https://raw.githubusercontent.com/emccode/ecs-docker/master/docker-machine-hostprep.sh"
    docker-machine ssh swarm-node01 "curl -O https://raw.githubusercontent.com/emccode/ecs-docker/master/docker-machine-hostprep.sh"
    docker-machine ssh swarm-node02 "curl -O https://raw.githubusercontent.com/emccode/ecs-docker/master/docker-machine-hostprep.sh"
    ```
2. The hosts require the IPs of all the ECS nodes in the cluster. To retrieve all the IPs use `docker-machine ls` and you will get a similar output as:
  ```
  NAME        ACTIVE   DRIVER      STATE     URL                     SWARM
  swarm-create   *   amazonec2   Running   tcp://52.4.23.123:2376
  swarm-master       amazonec2   Running   tcp://52.7.13.18:2376    swarm-master (master)
  swarm-node01       amazonec2   Running   tcp://54.12.23.16:2376   swarm-master
  swarm-node02       amazonec2   Running   tcp://52.7.19.173:2376   swarm-master
  ```
3. The command to kick-off the host preparation accepts two arguments. 
    1. all three IP addresses as comma seperated values with no spaces such as `52.7.13.18,54.12.23.16,52.7.19.173`
    2. the mounted drive path without the `/dev/` portion such as `xvdf`. If no argument is specified, `xvdf` is the default.
    ```
    docker-machine ssh swarm-master "sudo sh docker-machine-hostprep.sh 52.7.13.18,54.12.23.16,52.7.19.173 xvdf"
    docker-machine ssh swarm-node01 "sudo sh docker-machine-hostprep.sh 52.7.13.18,54.12.23.16,52.7.19.173 xvdf"
    docker-machine ssh swarm-node02 "sudo sh docker-machine-hostprep.sh 52.7.13.18,54.12.23.16,52.7.19.173 xvdf"
    ```
4. The last output line will say `Host has been successfully prepared`

#### Docker Compose Up
The `docker-compose.yml` file will create 3 ECS containers including all  volume mounts and networking. 

1. Connect to the Docker Machine Swarm instance: `docker-machine env --swarm swarm-master`
2. Get shell access to the Swarm Cluster: `eval "$(docker-machine env --swarm swarm-master)"`
3. To test if swarm is working correctly, run `docker info` and you will see an output such as:
```
Containers: 4
Strategy: spread
Filters: affinity, health, constraint, port, dependency
Nodes: 3
 swarm-master: 52.7.13.18:2376
  └ Containers: 2
  └ Reserved CPUs: 0 / 4
  └ Reserved Memory: 0 B / 15.42 GiB
 swarm-node01: 54.12.23.16:2376
  └ Containers: 1
  └ Reserved CPUs: 0 / 4
  └ Reserved Memory: 0 B / 15.42 GiB
 swarm-node02: 52.7.19.173:2376
  └ Containers: 1
  └ Reserved CPUs: 0 / 4
  └ Reserved Memory: 0 B / 15.42 GiB
```
4. Copy this repo using `git clone https://github.com/emccode/ecs-docker.git` or save the `docker-compose.yml` file to a folder.
5. Navigate to your folder such as `ecs-docker`
6. Run `docker-compose up -d`. Docker Compose will be pointing towards the registerd Docker Client machine which is our swarm cluster:
BELOW WILL BE CHANGED. This output show docker-compose will error out
```
kcoleman-mbp:ecs-docker kcoleman$ docker-compose up -d
Recreating swarm-agent...
Pulling ecsnode02 (emccode/ecsobjectsw:v2.0)...
swarm-master: Pulling emccode/ecsobjectsw:v2.0... : downloaded
swarm-node01: Pulling emccode/ecsobjectsw:v2.0... : downloaded
swarm-node02: Pulling emccode/ecsobjectsw:v2.0... : downloaded
Traceback (most recent call last):
  File "<string>", line 3, in <module>
  File "/compose/build/docker-compose/out00-PYZ.pyz/compose.cli.main", line 32, in main
  File "/compose/build/docker-compose/out00-PYZ.pyz/compose.cli.docopt_command", line 21, in sys_dispatch
  File "/compose/build/docker-compose/out00-PYZ.pyz/compose.cli.command", line 34, in dispatch
  File "/compose/build/docker-compose/out00-PYZ.pyz/compose.cli.docopt_command", line 24, in dispatch
  File "/compose/build/docker-compose/out00-PYZ.pyz/compose.cli.command", line 66, in perform_command
  File "/compose/build/docker-compose/out00-PYZ.pyz/compose.cli.main", line 470, in up
  File "/compose/build/docker-compose/out00-PYZ.pyz/compose.project", line 230, in up
  File "/compose/build/docker-compose/out00-PYZ.pyz/compose.service", line 333, in execute_convergence_plan
  File "/compose/build/docker-compose/out00-PYZ.pyz/compose.service", line 380, in recreate_container
  File "/compose/build/docker-compose/out00-PYZ.pyz/compose.service", line 214, in create_container
  File "/compose/build/docker-compose/out00-PYZ.pyz/compose.service", line 450, in _next_container_number
  File "/compose/build/docker-compose/out00-PYZ.pyz/compose.container", line 70, in number
ValueError: Container f9bef6a5b6 does not have a com.docker.compose.container-number label
```

#### Access Granted!
Will finish up when I can access the services and UI.

## Contribution
Create a fork of the project into your own reposity. Make all your necessary changes and create a pull request with a description on what was added or removed and details explaining the changes in lines of code. If approved, project owners will merge it.

Licensing
---------
ecs-docker is freely distributed under the [MIT License](http://opensource.org/licenses/MIT). See LICENSE for details.

Support
-------
Please file bugs and issues at the Github issues page. For more general discussions you can contact the EMC Code team at <a href="https://groups.google.com/forum/#!forum/emccode-users">Google Groups</a> or tagged with **EMC** on <a href="https://stackoverflow.com">Stackoverflow.com</a>. The code and documentation are released with no warranties or SLAs and are intended to be supported through a community driven process.