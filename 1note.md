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
