#1. Why Multiple Container Hosts?
**Install containner**

```docker
docker run -rm -d -p 3000:3000 swarmgs/nodewebstress
```

**Basic Load Test**
```bash
ab -n 1000 -c 4 http://127.0.0.1:3000/customer/1
```

A `-c 4` that means send four requests at the same time.
After running this command, you will set a result of load testing. Check `Requests per second` param.

Think about, Optimize API code or run API more than one container.

**Run two instance in same machine**

```docker
docker run --rm -d -p 3001:3000 swarmgs/nodewebstress
```

Install parallel
```
brew install parallel
```

Run load test again.
```
echo "http://127.0.0.1:3000/customer/1\nhttp://127.0.0.1:3001/customer/2" | parallel -j 2 "ab -n 1000 {.}"
```

**Connerns**
- Scale Capaciry
    - Multiple Containers
    - Load Balancing
- Container Failure
    - Restart
        ```
        docker run -d -p 3002:3000 --restart=unless-stoped swarmgs/nodewebstress
        ```

- Node Failuer
    - Redistribute Containers
    - Replace Node
    - Placement
    - Node Maintenance
- Internal Communication
    - Demo
        - Start Customer API
            ```
            docker run --rm -d -p 3000:3000 --name customer-api --restart=unless-stoped swarmgs/nodewebstress
            ```
        - Start Balance API
            ```
            docker run --rm -d -p 4000:3000 --name balance-api -e MYWEB_CUSTOMER_API=192.168.0.10:3000 swarmgs/nodewebstress
            ```
            `-e MYWEB_CUSTOMER_API`  is defined env parameter

- Defined Networks to Container
    - Sharing single address on Node
    - Create new networks
        ```
        docker network create -d=bridge company
        ```
    - Start Container
        ```
        docker run --rm -d --name customer-api --network company swarmgs/customer
        ```
        ```
        docker run --rm -d --name balance-api --network company -p 4000:3000 -e  MYWEB_CUSTOMER_API=customer-api:3000 swarmgs/balance
        ```
- docker-compose.yml
    - See a `company.yml`
    - Run `docker-compose -f c1/company.yml up -d` 
    - Check network `docker network ls`
    - Stop `docker-compose -f company.yml stop customer-api`

#2. Creating a Swarm and Running a Services

**Enable Swarm Mode**
```docker
docker info
```
check `Swarm inactive`

```docker 
docker -v
```
check Management Command

Check Swarm mode
```docker 
docker swarm -h
```

**Swarm init**
``` 
docker swarm init
```

**Docker Node Command**
``` 
docker node -h
docker node inspect self  
```

**_Special Function_**
```
docker service -h
```
vs
```
docker container -h
```

#3. Create Service Create Container
``` 
docker service create --name web --publish 8080:80 nginx
```
`--name web` : service name
`--publish` : enable publish port
`8080:80` : `8080` publish port , `80` binding port

**A Service is a Definition of an Application**
- What you would like to Run?
    - Aeb site
    - Database
    - Customer API
    - Summary : What is the application you would like to Run.
- For example : Web site services
    - image: nginx
    - publish : 8080:80
    - command, workdir, env, vars
    - memory/cpu limits
    - network
    - replicas: 2
    
**Service lead to Task**
- Task to prepare the container
- `docker service ps SERVICE_NAME` to show task ID in column ID

**Remove Service**
- `docker service rm web`    
- Don't do this in production

**Update a Service to Scale the Number of Container**
```
docker service update --replicas=3  web
```
Or
```
docker service scale web=4
```

Check service instance : `docker service ps web` or `docker ps`
Stop Container `docker stop ID`


**Task Scheduling**
- Check a diagram https://docs.docker.com/engine/swarm/how-swarm-mode-works/services/#tasks-and-scheduling

**Create Second Service**
- Create another service `docker service create --name customer-api --publish 3000:3000 swarmgs/customer`
- Check all service `docker service ls`
- Scale down to 
    - web =  2 `docker service scale web=2`
    - customer-api = 2 `docker service scale customer-api=2`
    

**Swarm Mode Routing**
- Check a diagram https://docs.docker.com/engine/swarm/ingress/#publish-a-port-for-a-service
- Can send traffic to any node that not have container running

**Testing Throughput on a Scaled Service**
- Use `ab -n 1000 -c 4 http://127.0.0.1:3000/customer/1`
- Check `Requests per second:    166.29 [#/sec] (mean)`
- Scale up `docker service scale customer-api=4` 
- Check new result `Requests per second:    272.35 [#/sec] (mean)`. this mean double Throughput(Good).
- Solt: 1 CPU = 1 Container

#4. Adding nodes
**Remove node**
- `docker swarm leave --force`. used the `--force` for self node.

**Creating and Managing VMs with docker Machine**

- We have two option for create Node
    - Using Docker Machine
        - `docker-machine create -d virtualbox  m1`
        - `docker-machine env m1`
        - `eval $(docker-machine env m1)`
        - `docker-machine ssh m1`
        - Remove `docker-machine rm m1`
    - Using Vagrant
        - Check docker-swarm-mode-getting-started/Vagrantfile
- Launching 3 VMs with `vagrant up`
    - `vagrant up m1 w1 w2`
    - `vagrant status`
    
- Accessing the Docker Engine In VMs
    - `vargrant ssh m1`
    - `export DOCKER_HOST=192.168.99.201`
    - `docker info | grep Name`
    - `docker ps`
    - `docker images ls`
    
- docker swarm init --advertise-addr command
    - `export DOCKER_HOST=192.168.99.201`
    - `docker swarm init --advertise-addr 192.168.99.201:2377 --listen-addr 192.168.99.201:2377`
    - Check `docker info | grep Swarm`. It shows `Swarm: active`
    - Reference :  [How nodes work](https://docs.docker.com/engine/swarm/how-swarm-mode-works/nodes/)

- Joining workers node
    - `export DOCKER_HOST=192.168.99.211`
    - Join 
        ```docker
        docker swarm join --token SWMTKN-1-2lvpl5x8o812d0iuqud5icrv13q8v62ssc4lsrw2plk43lzujp-8cyjkk81q4n6u6nou0ykaw8ig 192.168.99.201:2377
        ``` 
- Creating a Service to Visualize Our Cluster State
    - Crete _docker-swarm-visualizer container_
        - ```
          docker service create --name viz \
          --publish 8090:8080 \
          --mount=type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
          --constraint=node.role==manager \
           dockersamples/visualizer
           ```
        - `docker service ls`
        - `docker service ps viz`
        - `open http://192.168.99.201:8090/`. Can not uses localhost
- What happen when a Node shutdown.
    - Check Node Down `docker node ls | grep Down`
    - Check viz `open http://192.168.99.201:8090/`
    - Example. Remove a Node
        - `export DOCKER_HOST=192.168.99.212 `
        - `docker swarm leave`
        - `docker node rm w2`
    - Re-Joining Node
        - `docker swarm join-token worker`
        - `export DOCKER_HOST=192.168.99.212`
        - `docker swarm join --token SWMTKN-1-2lvpl5x8o812d0iuqud5icrv13q8v62ssc4lsrw2plk43lzujp-8cyjkk81q4n6u6nou0ykaw8ig 192.168.99.201:2377`

- Creating and Scaling service
    - `export DOCKER_HOST=192.168.99.201`
    - `docker pull  swarmgs/customer`
    - ```
      docker service create --name customer-api \
      --publish 3000:3000 \
      swarmgs/customer
      ```
    -  Scale: `docker service scale customer-api=2`
    -  Scale: `docker service scale customer-api=3`
- Spread Strategy and Test Throughput of Scale service
    - `open http://192.168.99.201:3000/customer/1`
    - Scale up to 3 container `docker service scale customer-api=3`
    - Load Test 
        - Scale down to 1 container `docker service scale customer-api=1`
        - `ab -n 1000 -c 4 http://192.168.99.201:3000/customer/1`
        - Review `Requests per second:    88.51 [#/sec] (mean)`
        - Scale up to 4 container `docker service scale customer-api=4`
        - Review `Requests per second:    271.03 [#/sec] (mean)`

- Inspecting Nodes and Clustering Gotchas
    - `docker node ls` list of the node
    - `docker node inspect w2` 
        - Ip address
        - OS
        - Resource CPU, MEM        
    - Test scale down to 1 `docker service scale customer-api=1`
        - `m1` has viz task
        - `w1` has customer-api task
        - `w2` has note any task
    - `docker ps` : show only container on self node.
    - `docker network ls` : check `ingress`
    - `docker network inspect ingress` : show self node network info. we using peer node.
    
- List Task per Nodes(If not install `viz`)
    - `docker node ps m1` : check all task running on `m1` node.
    - `docker node ps m1 w1 w2`   
- Promoting a Worker to a Manager 
    - `docker node promote w2`. `w2` 's Manager Status will change to Manager `Reachable`. It use full for some to bing manager online quickly.
    - `docker node demote w2`

- Draining a Node to Perform Maintenance
    - Check command `docker node update -h`
    - Use case : update `w1`
        - `docker node update --availability=drain w1` It will move task to another node. So we can update path etc.
        - `docker node update --availability=active w1` The task will not re-balance.
        - Re-Balance
            - `docker service scale customer-api=2`
            - `docker service scale customer-api=1`
- One Container per Node with Global Services
    - By default mode is replicated but we have difference type of mode.   
    - Use Case : Installing `cAdvisor` to every node.
        - [cAdvisor Installation Guid](https://blog.codeship.com/monitoring-docker-containers-with-elasticsearch-and-cadvisor/)
        - ```
            docker service create --mode=global --name cadvisor\
            --mount type=bind,source=/,target=/rootfs,readonly=true \
            --mount type=bind,source=/var/run,target=/var/run,readonly=false \
            --mount type=bind,source=/sys,target=/sys,readonly=true \
            --mount type=bind,source=/var/lib/docker/,target=/var/lib/docker,readonly=true \
            google/cadvisor:latest 
          ``` 
- Swarm Mode in incredibly Easy to Setup
    - [SwarmKit](https://github.com/docker/swarmkit)


#5. Ingress Routing and Publish Ports
**Published ports Provide External Access to Services**
- Swarm load balancing work behind the scene.

**Ingress Overlay Network**
- Ingress Overlay network is a technical that Swarm Load Balancer.
- There are run across between Nodes.
- `docker network ls`. See name `ingress`
- `docker network inspect ingress` to inspecting a detail of ingress network.By default subnet `10.255.0.0/16`

**Options to routing External Traffic to Nodes**
- Set up DNS 
    - Server to routing traffic to one Node but If node down we cloud have a problem.
    - We can Set  round-robin DNS or set up multiple ID for a given domain
- External LB
    - Create one IP Address for LB set out site swarms
    - LB will route available node
    - In case create new Node we will update external LB

**Ingress publish Mode Routes to a Random Cluster**
- Publish cAdvisor on 8080 port
    - Command
    ```
        docker service create --mode=global --name cadvisor\
        --mount type=bind,source=/,target=/rootfs,readonly=true \
        --mount type=bind,source=/var/run,target=/var/run,readonly=false \
        --mount type=bind,source=/sys,target=/sys,readonly=true \
        --mount type=bind,source=/var/lib/docker/,target=/var/lib/docker,readonly=true \
        --publish 8080:8080 \
        google/cadvisor:latest 
        
    ```
    - `open http://192.168.99.201:8080` It will random node. So we can't monitor a specific node.
    
**Remove a published port on an external Service**
- `docker service inspect cadvisor`. Check `Ports` and See `"PublishMode": "ingress"`
- `docker service update --publish-rm 8080 cadvisor`
- `docker service inspect cadvisor` It still ingress network

**Adding a Host Mode Published Port**
- `docker service update --publish-add mode=host,published=8080,target=8080 cadvisor`
- `docker service inspect cadvisor` It show `"PublishMode": "host"`
- `open http://192.168.99.201:8080` for `m1`
- `open http://192.168.99.211:8080` for  `w1`

**Publishing a random ports**
- Create new Service with random ports
    - `docker service create --name nginx-random -p target=80 nginx`
    - Check the random ports `docker service inspect nginx-random | grep PublishedPort`

#6. Reconciling a Desired State
**Quiz- What happen when we add a Node to the Swarm?**
- Join New Node step
    - `vagrant up w3`
    - `docker swarm join-token worker`
    - `export DOCKER_HOST=192.168.99.213`
    -  ```
        docker swarm join --token SWMTKN-1-2lvpl5x8o812d0iuqud5icrv13q8v62ssc4lsrw2plk43lzujp-8cyjkk81q4n6u6nou0ykaw8ig 192.168.99.201:2377
        ```

**Creating a Pending Service and Inspecting**
- Run container with specific node  `docker service create --name onw4 -p 9000:80 --constraint node.hostname==w3 nginx`
- It work fine.
- If Run container with specific node  `docker service create --name onw5 -p 9000:80 --constraint node.hostname==w4 nginx`
    - `w4` node not created yet.
    - If we use `docker service ps onw5`. It shows pending.
    - Or use `docker inspect ID` command. Status also pending.`message` node shows more infomate
    
**Joining a New Node to Fulfill a Pending Service**
- Create new `w4` server and join to the node.
- `docker service ps onw5 `
- `docker service inspect onw5 --pretty`

**What happen to a Service When we lose the Only Compatible Node?**
- Use Case : Shutdown `w4` node
    - `vagrant destroy -f w4`
    - `docker service ls`
    - We will up node Asap.

**Cleaning up Nodes that have Failed**
- `docker node rm w4`. This can use only the node that not have container in site.

**Remove Vestigial Service**
- `docker service rm onw5`

**If Your app fails Then the corresponding Task will be shutdown**
- Use Case : 
    - `docker service create --name exploder -p 9001:3000 swarmgs/nodewebstress`. This image have a API to terminating a Application.
    - `docker service ps exploder`
    - Swarm will shutdown container and create new task automatically( Not guarantee running on same node)
    
**Scale a Service to Zero to Stop it Without Removing It** 
- `docker service scale exploder=0`   