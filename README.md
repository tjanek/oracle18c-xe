# Oracle Database on Docker
Sample Docker build files to facilitate installation, configuration, and environment setup for DevOps users. For more information about Oracle Database please see the [Oracle Database Online Documentation](https://docs.oracle.com/en/database/oracle/oracle-database/index.html).

## About
This is a modified Dockerfile of fuzziebrain user [docker-oracle-xe](https://github.com/fuzziebrain/docker-oracle-xe)

The problem with oracle images is extremely slow start. On the first start database is created, and this step can take several minutes.

I decided divide a runOracle.sh file into two files:
+ PREPARE_FILE="prepareOracle.sh" is responsible for create a database.
+ RUN_FILE="runOracle.sh" responsible only for start a prepared database.

In Dockerfile we have additional layer which is responsible for run a prepareOracle.sh

This image is available on docker hub:
+ docker pull kamiljedrzejuk/oracle18c-xe-initialized:latest

## Build Image

```bash
-- Clone repo
git clone https://github.com/KamilJedrzejuk/oracle18c-xe

-- Set the working directory to the project folder
cd docker-oracle-xe

-- Copy the RPM to docker-odb18c-xe/files
cp ~/Downloads/oracle-database-xe-18c-1.0-1.x86_64.rpm files/

-- Build Image
docker build -t oracle-xe:18c .
```
## Run Container

```bash
docker run -d \
  -p 32118:1521 \
  -p 35518:5500 \
  --name=oracle-xe \
  --volume ~/docker/oracle-xe:/opt/oracle/oradata \
  --network=oracle_network \
  oracle-xe:18c
  
docker logs oracle-xe
```

Run parameters:

Name | Required | Description 
--- | --- | ---
`-p 1521`| Required | TNS Listener. `32118:1521` maps `32118` on your laptop to `1521` on the container.
`-p 5500`| Optional | Enterprise Manager (EM) Express. `35518:5500` maps `35518` to your laptop to `5500` on the container. You can then access EM via https://localhost:35518/em 
`--name` | Optional | Name of container. Optional but recommended
`--volume /opt/oracle/oradata` | Optional | (recommended) If provided, data files will be stored here. If the container is destroyed can easily rebuild container using the data files.
`--network` | Optional | If other containers need to connect to this one (ex: [ORDS](https://github.com/martindsouza/docker-ords)) then they should all be on the same docker network.
`oracle-xe:18c` | Required | This is the `name:tag` of the docker image that was built in the previous step

## Container Commands

```bash
# Status:
# Look under the STATUS column for "(health: ...".
docker ps

# Start container
docker start oracle-xe

# Stop container
docker stop -t 200 oracle-xe
```











