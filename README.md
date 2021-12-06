# HSA L9.1: Redis Cluster

## Overview
This is an example project to show how to master-slave redis cluster and implement probabilistic cache clearing.

## Getting Started

### Preparation
Run the docker containers.
```bash
  docker-compose up -d
```

Be sure to use ```docker-compose down -v``` to cleanup after you're done with tests.

## Test scenario
#### 1. Make 3 requests to show the work of probabilistic cache
##### 1.1 User controller requests data from Redis, but as it's first call Redis database is empty, so the data is generated and saved to Redis.
```bash
curl -s -i http://localhost:8080/user
HTTP/1.1 201 Created
Server: nginx
Date: Mon, 06 Dec 2021 13:36:18 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
X-Powered-By: PHP/8.0.13

{
    "data": {
        "uuid": "4afa80ff-065a-3497-b81f-77b1b5d9b9b0",
        "username": "berta.schaefer",
        "email": "Kiehn.Elouise@hotmail.com",
        "first_name": "Lela",
        "last_name": "Turcotte"
    },
    "redis_cache_info": "Redis used memory: 1.85M\r\nRedis keyspace: {\"db0\":{\"keys\":\"1\",\"expires\":\"1\",\"avg_ttl\":\"0\"}}\r\n"
}
```
##### 1.2 User controller requests data from Redis. Data isn't expired (avg_ttl=9183).
```bash
curl -s -i http://localhost:8080/user
HTTP/1.1 201 Created
Server: nginx
Date: Mon, 06 Dec 2021 13:36:19 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
X-Powered-By: PHP/8.0.13

{
    "data": {
        "uuid": "4afa80ff-065a-3497-b81f-77b1b5d9b9b0",
        "username": "berta.schaefer",
        "email": "Kiehn.Elouise@hotmail.com",
        "first_name": "Lela",
        "last_name": "Turcotte"
    },
    "redis_cache_info": "Redis used memory: 1.85M\r\nRedis keyspace: {\"db0\":{\"keys\":\"1\",\"expires\":\"1\",\"avg_ttl\":\"9183\"}}\r\n"
}
```
##### 1.3 User controller requests data from Redis, but the data was expired. New data is generated.
```bash
curl -s -i http://localhost:8080/user
HTTP/1.1 201 Created
Server: nginx
Date: Mon, 06 Dec 2021 13:40:18 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
X-Powered-By: PHP/8.0.13

{
"data": {
"uuid": "78cde189-ce98-3131-9591-0f6ae0ce1301",
"username": "moore.sydnie",
"email": "Tom69@hotmail.com",
"first_name": "Peyton",
"last_name": "Greenfelder"
},
"redis_cache_info": "Redis used memory: 1.85M\r\nRedis keyspace: {\"db0\":{\"keys\":\"1\",\"expires\":\"1\",\"avg_ttl\":\"0\"}}\r\n"
}
```

#### 2. Check that data is shared between master and slave Redis clusters. Let make a simple request.
```bash
curl -s -i http://localhost:8080/user
```

##### 2.1 Redis Master.
```bash
docker exec -it redis-master redis-cli get test_user
"[{\"uuid\":\"647986da-93e5-37e7-92fe-e313c44ab161\",\"username\":\"maggio.woodrow\",\"email\":\"Bashirian.Vella@Lesch.com\",\"first_name\":\"Destany\",\"last_name\":\"Stamm\"},0,1638798418]"
```

##### 2.2 Redis Slave.
```bash
docker exec -it redis-slave redis-cli get test_user
"[{\"uuid\":\"647986da-93e5-37e7-92fe-e313c44ab161\",\"username\":\"maggio.woodrow\",\"email\":\"Bashirian.Vella@Lesch.com\",\"first_name\":\"Destany\",\"last_name\":\"Stamm\"},0,1638798418]"
```
