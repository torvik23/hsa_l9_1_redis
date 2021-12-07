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

#### 3. Eviction strategies comparison.
Let's check Redis eviction strategies performance with [memtier_benchmark](https://github.com/RedisLabs/memtier_benchmark)
- Maxmemory: 100mb
- Threads: 4 (default)
- Requests per client: 100000

Amount of clients (concurrency) per thread will be changed on each test scenario.

| Eviction strategy | 10 | 25 | 50 | 100 | 1000 |
|:---:|---|---|---|---|---|
| volatile-lru | Ops/sec - 342868.36<br>Hits/sec - 128.06<br>Misses/sec - 311570.14<br>Avg. Latency - 1.16<br>KB/sec - 14547.06 | Ops/sec - 378966.93<br>Hits/sec - 276.65<br>Misses/sec - 344238.40<br>Avg. Latency - 2.63<br>KB/sec - 16083.12 | Ops/sec - 332423.67<br>Hits/sec - 242.67<br>Misses/sec - 301960.37<br>Avg. Latency - 6.02<br>KB/sec - 14107.85 | Ops/sec - 311303.76<br>Hits/sec - 227.25<br>Misses/sec - 282775.88<br>Avg. Latency - 12.83<br>KB/sec - 13211.53 | Error Response: OOM command not allowed when used memory > 'maxmemory'. |
| allkeys-lru | Ops/sec - 305042.71<br>Hits/sec - 112.26<br>Misses/sec - 277199.02<br>Avg. Latency - 1.20<br>KB/sec - 12942.15 | Ops/sec - 389684.23<br>Hits/sec - 284.47<br>Misses/sec - 353973.57<br>Avg. Latency - 2.56<br>KB/sec - 16537.95 | Ops/sec - 352878.14<br>Hits/sec - 257.60<br>Misses/sec - 320540.38<br>Avg. Latency - 5.66<br>KB/sec - 14975.92 | Ops/sec - 380001.14<br>Hits/sec - 277.40<br>Misses/sec - 345177.84<br>Avg. Latency - 10.51<br>KB/sec - 16127.01 | Error Response: OOM command not allowed when used memory > 'maxmemory'. |
| volatile-lfu | Ops/sec - 317663.14<br>Hits/sec - 116.98<br>Misses/sec - 288667.41<br>Avg. Latency - 1.27<br>KB/sec - 13477.61 | Ops/sec - 362152.66<br>Hits/sec - 264.37<br>Misses/sec - 328964.99<br>Avg. Latency - 2.75<br>KB/sec - 15369.53 | Ops/sec - 412479.99<br>Hits/sec - 301.11<br>Misses/sec - 374680.32<br>Avg. Latency - 4.84<br>KB/sec - 17505.39 | Ops/sec - 382088.20<br>Hits/sec - 278.92<br>Misses/sec - 347073.64<br>Avg. Latency - 10.46<br>KB/sec - 16215.58 | Error Response: OOM command not allowed when used memory > 'maxmemory'. |
| allkeys-lfu | Ops/sec - 306335.36<br>Hits/sec - 113.42<br>Misses/sec - 278372.99<br>Avg. Latency - 1.21<br>KB/sec - 12997.02 | Ops/sec - 372935.10<br>Hits/sec - 272.24<br>Misses/sec - 338759.33<br>Avg. Latency - 2.66<br>KB/sec - 15827.13 | Ops/sec - 382758.69<br>Hits/sec - 279.41<br>Misses/sec - 347682.68<br>Avg. Latency - 5.27<br>KB/sec - 16244.04 | Ops/sec - 401847.64  <br>Hits/sec - 293.35<br>Misses/sec - 365022.32<br>Avg. Latency - 9.91<br>KB/sec - 17054.16 | Error Response: OOM command not allowed when used memory > 'maxmemory'. |
| volatile-random | Ops/sec - 311266.75<br>Hits/sec - 114.39<br>Misses/sec - 282855.10<br>Avg. Latency - 1.28<br>KB/sec - 13206.22 | Ops/sec - 356578.18<br>Hits/sec - 260.30<br>Misses/sec - 323901.35<br>Avg. Latency - 2.79<br>KB/sec - 15132.95 | Ops/sec - 392895.65<br>Hits/sec - 286.81<br>Misses/sec - 356890.69<br>Avg. Latency - 5.09<br>KB/sec - 16674.24 | Ops/sec - 390637.22<br>Hits/sec - 285.17<br>Misses/sec - 354839.22<br>Avg. Latency - 10.21<br>KB/sec - 16578.40 | Error Response: OOM command not allowed when used memory > 'maxmemory'. |
| allkeys-random | Ops/sec - 346176.03<br>Hits/sec - 252.71<br>Misses/sec - 314452.46<br>Avg. Latency - 1.25<br>KB/sec - 14691.49 | Ops/sec - 369686.03<br>Hits/sec - 269.87<br>Misses/sec - 335808.00<br>Avg. Latency - 2.69<br>KB/sec - 15689.24 | Ops/sec - 429609.76<br>Hits/sec - 313.62<br>Misses/sec - 390240.32<br>Avg. Latency - 4.62<br>KB/sec - 18232.37 | Ops/sec - 386027.28<br>Hits/sec - 281.80<br>Misses/sec - 350651.74<br>Avg. Latency - 10.36<br>KB/sec - 16382.75 | Error Response: OOM command not allowed when used memory > 'maxmemory'. |
| volatile-ttl | Ops/sec - 284302.30<br>Hits/sec - 207.54<br>Misses/sec - 258248.84<br>Avg. Latency - 1.40<br>KB/sec - 12065.61 | Ops/sec - 338610.42<br>Hits/sec - 247.19<br>Misses/sec - 307580.16<br>Avg. Latency - 2.94<br>KB/sec - 14370.41 | Ops/sec - 372284.44<br>Hits/sec - 271.77<br>Misses/sec - 338168.30<br>Avg. Latency - 5.34<br>KB/sec - 15799.52 | Ops/sec - 373302.81<br>Hits/sec - 272.51<br>Misses/sec - 339093.34<br>Avg. Latency - 10.70<br>KB/sec - 15842.73 | Error Response: OOM command not allowed when used memory > 'maxmemory'. |
| noeviction | Ops/sec - 290587.27<br>Hits/sec - 212.13<br>Misses/sec - 263957.85<br>Avg. Latency - 1.37<br>KB/sec - 12332.34 | Ops/sec - 341292.58<br>Hits/sec - 249.14<br>Misses/sec - 310016.53<br>Avg. Latency - 2.92<br>KB/sec - 14484.24 | Ops/sec - 366471.58<br>Hits/sec - 267.52<br>Misses/sec - 332888.13<br>Avg. Latency - 5.46<br>KB/sec - 15552.82 | Ops/sec - 382505.00<br>Hits/sec - 279.23<br>Misses/sec - 347452.24<br>Avg. Latency - 10.48<br>KB/sec - 16233.26 | Error Response: OOM command not allowed when used memory > 'maxmemory'. |