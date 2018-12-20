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

**Create Service Create Container**
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

#3. Adding nodes
**Remove node**
- `docker swarm leave --force`. used the `--force` for last node.

**Creating and Managing VMs with docker Machine**
We have two option for create Node
- Using Docker Machine
    - `docker-machine create -d virtualbox  m1`
    - `docker-machine env m1`
    - `eval $(docker-machine env m1)`
    - `docker-machine ssh m1`
    - Remove `docker-machine rm m1`
- Using Vagrant
    - Check Vagrantfile
    
    
    













