# Oracle Database on Docker
Sample Docker build files to facilitate installation, configuration, and environment setup for DevOps users. For more information about Oracle Database please see the [Oracle Database Online Documentation](https://docs.oracle.com/en/database/oracle/oracle-database/index.html).

## About
This is a modified Dockerfile of fuzziebrain user [Docker 18.4.0](https://github.com/fuzziebrain/docker-oracle-xe)

The problem with oracle images is extremely slow start. On the first start database is created, and this step can take several minutes.

I decided divide a runOracle.sh file into two files:
+ PREPARE_FILE="prepareOracle.sh" is responsible for create a database.


```bash
#!/bin/bash
# Original source from: https://github.com/oracle/docker-images/blob/master/OracleDatabase/SingleInstance/dockerfiles/18.3.0/runOracle.sh

# LICENSE UPL 1.0
#
# Copyright (c) 1982-2018 Oracle and/or its affiliates. All rights reserved.
# 
# Since: November, 2016
# Author: gerald.venzl@oracle.com
# Description: Runs the Oracle Database inside the container
# 
# DO NOT ALTER OR REMOVE COPYRIGHT NOTICES OR THIS HEADER.
# 

########### Move DB files ############
function moveFiles {

  if [ ! -d $ORACLE_BASE/oradata/dbconfig/$ORACLE_SID ]; then
    mkdir -p $ORACLE_BASE/oradata/dbconfig/$ORACLE_SID/
  fi;

  # Replace the container's hostname with 0.0.0.0
  sed -i 's/'$(hostname)'/0.0.0.0/g' $ORACLE_HOME/network/admin/listener.ora
  sed -i 's/'$(hostname)'/0.0.0.0/g' $ORACLE_HOME/network/admin/tnsnames.ora

  mv $ORACLE_HOME/dbs/spfile$ORACLE_SID.ora $ORACLE_BASE/oradata/dbconfig/$ORACLE_SID/
  mv $ORACLE_HOME/dbs/orapw$ORACLE_SID $ORACLE_BASE/oradata/dbconfig/$ORACLE_SID/
  mv $ORACLE_HOME/network/admin/sqlnet.ora $ORACLE_BASE/oradata/dbconfig/$ORACLE_SID/
  mv $ORACLE_HOME/network/admin/listener.ora $ORACLE_BASE/oradata/dbconfig/$ORACLE_SID/
  mv $ORACLE_HOME/network/admin/tnsnames.ora $ORACLE_BASE/oradata/dbconfig/$ORACLE_SID/

  # oracle user does not have permissions in /etc, hence cp and not mv
  cp /etc/oratab $ORACLE_BASE/oradata/dbconfig/$ORACLE_SID/
   
  symLinkFiles;
}

########### Symbolic link DB files ############
function symLinkFiles {

  if [ ! -L $ORACLE_HOME/dbs/spfile$ORACLE_SID.ora ]; then
    ln -s $ORACLE_BASE/oradata/dbconfig/$ORACLE_SID/spfile$ORACLE_SID.ora $ORACLE_HOME/dbs/spfile$ORACLE_SID.ora
  fi;
   
  if [ ! -L $ORACLE_HOME/dbs/orapw$ORACLE_SID ]; then
    ln -s $ORACLE_BASE/oradata/dbconfig/$ORACLE_SID/orapw$ORACLE_SID $ORACLE_HOME/dbs/orapw$ORACLE_SID
  fi;
   
  if [ ! -L $ORACLE_HOME/network/admin/sqlnet.ora ]; then
    ln -s $ORACLE_BASE/oradata/dbconfig/$ORACLE_SID/sqlnet.ora $ORACLE_HOME/network/admin/sqlnet.ora
  fi;

  if [ ! -L $ORACLE_HOME/network/admin/listener.ora ]; then
    ln -s $ORACLE_BASE/oradata/dbconfig/$ORACLE_SID/listener.ora $ORACLE_HOME/network/admin/listener.ora
  fi;

  if [ ! -L $ORACLE_HOME/network/admin/tnsnames.ora ]; then
    ln -s $ORACLE_BASE/oradata/dbconfig/$ORACLE_SID/tnsnames.ora $ORACLE_HOME/network/admin/tnsnames.ora
  fi;

  # oracle user does not have permissions in /etc, hence cp and not ln 
  cp $ORACLE_BASE/oradata/dbconfig/$ORACLE_SID/oratab /etc/oratab

}

########### SIGINT handler ############
function _int() {
  echo "Stopping container."
  echo "SIGINT received, shutting down database!"
  runuser oracle -s /bin/bash -c "${ORACLE_BASE}/scripts/${SHUTDOWN_FILE} immediate"
}

########### SIGTERM handler ############
function _term() {
  echo "Stopping container."
  echo "SIGTERM received, shutting down database!"
  runuser oracle -s /bin/bash -c "${ORACLE_BASE}/scripts/${SHUTDOWN_FILE} immediate"
}

########### SIGKILL handler ############
function _kill() {
  echo "SIGKILL received, shutting down database!"
  runuser oracle -s /bin/bash -c "${ORACLE_BASE}/scripts/${SHUTDOWN_FILE} abort"
}

###################################
# !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!! #
############# MAIN ################
# !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!! #
###################################

# Check whether container has enough memory
# Github issue #219: Prevent integer overflow,
# only check if memory digits are less than 11 (single GB range and below) 
if [ `cat /sys/fs/cgroup/memory/memory.limit_in_bytes | wc -c` -lt 11 ]; then
  if [ `cat /sys/fs/cgroup/memory/memory.limit_in_bytes` -lt 2147483648 ]; then
    echo "Error: The container doesn't have enough memory allocated."
    echo "A database container needs at least 2 GB of memory."
    echo "You currently only have $((`cat /sys/fs/cgroup/memory/memory.limit_in_bytes`/1024/1024/1024)) GB allocated to the container."
    exit 1;
  fi;
fi;

# Set SIGINT handler
trap _int SIGINT

# Set SIGTERM handler
trap _term SIGTERM

# Set SIGKILL handler
trap _kill SIGKILL

# Commands
ORACLE_CMD=/etc/init.d/oracle-xe-18c

# Check whether database already exists
if [ -d $ORACLE_BASE/oradata/$ORACLE_SID ]; then
  echo Database exists
   
  # Make sure audit file destination exists
  if [ ! -d $ORACLE_BASE/admin/$ORACLE_SID/adump ]; then
    mkdir -p $ORACLE_BASE/admin/$ORACLE_SID/adump
    chown -R oracle.oinstall $ORACLE_BASE/admin
  fi;
  
  symLinkFiles;
   
  # Start database
  ${ORACLE_CMD} start
  
else
  echo Database does not exists, configuring
 
  mkdir -p ${ORACLE_BASE}/oradata
  chown oracle.oinstall ${ORACLE_BASE}/oradata

  ${ORACLE_CMD} configure

  # Enable EM remote access
  runuser oracle -s /bin/bash -c "${ORACLE_BASE}/scripts/${EM_REMOTE_ACCESS} ${EM_GLOBAL_ACCESS_YN:-N}"

  # Move database operational files to oradata
  moveFiles;
   
fi;
```



+ RUN_FILE="runOracle.sh" responsible only for start a prepared database.


```bash
#!/bin/bash
# Original source from: https://github.com/oracle/docker-images/blob/master/OracleDatabase/SingleInstance/dockerfiles/18.3.0/runOracle.sh

# LICENSE UPL 1.0
#
# Copyright (c) 1982-2018 Oracle and/or its affiliates. All rights reserved.
# 
# Since: November, 2016
# Author: gerald.venzl@oracle.com
# Description: Runs the Oracle Database inside the container
# 
# DO NOT ALTER OR REMOVE COPYRIGHT NOTICES OR THIS HEADER.
# 

########### Symbolic link DB files ############
function symLinkFiles {

  if [ ! -L $ORACLE_HOME/dbs/spfile$ORACLE_SID.ora ]; then
    ln -s $ORACLE_BASE/oradata/dbconfig/$ORACLE_SID/spfile$ORACLE_SID.ora $ORACLE_HOME/dbs/spfile$ORACLE_SID.ora
  fi;
   
  if [ ! -L $ORACLE_HOME/dbs/orapw$ORACLE_SID ]; then
    ln -s $ORACLE_BASE/oradata/dbconfig/$ORACLE_SID/orapw$ORACLE_SID $ORACLE_HOME/dbs/orapw$ORACLE_SID
  fi;
   
  if [ ! -L $ORACLE_HOME/network/admin/sqlnet.ora ]; then
    ln -s $ORACLE_BASE/oradata/dbconfig/$ORACLE_SID/sqlnet.ora $ORACLE_HOME/network/admin/sqlnet.ora
  fi;

  if [ ! -L $ORACLE_HOME/network/admin/listener.ora ]; then
    ln -s $ORACLE_BASE/oradata/dbconfig/$ORACLE_SID/listener.ora $ORACLE_HOME/network/admin/listener.ora
  fi;

  if [ ! -L $ORACLE_HOME/network/admin/tnsnames.ora ]; then
    ln -s $ORACLE_BASE/oradata/dbconfig/$ORACLE_SID/tnsnames.ora $ORACLE_HOME/network/admin/tnsnames.ora
  fi;

  # oracle user does not have permissions in /etc, hence cp and not ln 
  cp $ORACLE_BASE/oradata/dbconfig/$ORACLE_SID/oratab /etc/oratab

}

########### SIGINT handler ############
function _int() {
  echo "Stopping container."
  echo "SIGINT received, shutting down database!"
  runuser oracle -s /bin/bash -c "${ORACLE_BASE}/scripts/${SHUTDOWN_FILE} immediate"
}

########### SIGTERM handler ############
function _term() {
  echo "Stopping container."
  echo "SIGTERM received, shutting down database!"
  runuser oracle -s /bin/bash -c "${ORACLE_BASE}/scripts/${SHUTDOWN_FILE} immediate"
}

########### SIGKILL handler ############
function _kill() {
  echo "SIGKILL received, shutting down database!"
  runuser oracle -s /bin/bash -c "${ORACLE_BASE}/scripts/${SHUTDOWN_FILE} abort"
}

###################################
# !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!! #
############# MAIN ################
# !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!! #
###################################

# Check whether container has enough memory
# Github issue #219: Prevent integer overflow,
# only check if memory digits are less than 11 (single GB range and below) 
if [ `cat /sys/fs/cgroup/memory/memory.limit_in_bytes | wc -c` -lt 11 ]; then
  if [ `cat /sys/fs/cgroup/memory/memory.limit_in_bytes` -lt 2147483648 ]; then
    echo "Error: The container doesn't have enough memory allocated."
    echo "A database container needs at least 2 GB of memory."
    echo "You currently only have $((`cat /sys/fs/cgroup/memory/memory.limit_in_bytes`/1024/1024/1024)) GB allocated to the container."
    exit 1;
  fi;
fi;

# Set SIGINT handler
trap _int SIGINT

# Set SIGTERM handler
trap _term SIGTERM

# Set SIGKILL handler
trap _kill SIGKILL

# Commands
ORACLE_CMD=/etc/init.d/oracle-xe-18c

# Check whether database already exists
if [ -d $ORACLE_BASE/oradata/$ORACLE_SID ]; then
  echo Database exists
   
  # Make sure audit file destination exists
  if [ ! -d $ORACLE_BASE/admin/$ORACLE_SID/adump ]; then
    mkdir -p $ORACLE_BASE/admin/$ORACLE_SID/adump
    chown -R oracle.oinstall $ORACLE_BASE/admin
  fi;
  
  symLinkFiles;
   
  # Start database
  ${ORACLE_CMD} start
fi;

#Check whether database is up and running
$ORACLE_BASE/scripts/$CHECK_DB_FILE
if [ $? -eq 0 ]; then
  echo "#########################"
  echo "DATABASE IS READY TO USE!"
  echo "#########################"
  
else
  echo "#####################################"
  echo "########### E R R O R ###############"
  echo "DATABASE SETUP WAS NOT SUCCESSFUL!"
  echo "Please check output for further info!"
  echo "########### E R R O R ###############" 
  echo "#####################################"
fi;

# Tail on alert log and wait (otherwise container will exit)
echo "The following output is now a tail of the alert.log:"
tail -f ${ORACLE_BASE}/diag/rdbms/*/*/trace/alert*.log &
childPID=$!
wait ${childPID}

# TODO workaround
tail -f /dev/null

```


In Dockerfile we have additional layer which is responsible for run a prepareOracle.sh



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
  
# As this takes a long time to run you can keep track of the initial installation by running:
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











