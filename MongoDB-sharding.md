
## MongoDB Sharding
### Architecture - 
<img src = "https://github.com/nikitabairagi/Personal-Project/blob/master/MongoDb_sharded_cluster_topology.png" width="65%">
### Steps -

### 1. Creating MongoDb AMI Image :

* **Launched Linux AMI instance -**

   ```
   Amazon Linux AMI 
   T2 Micro Instance
   VPC: default vpc
   Public Subnet
   Auto Assign Public IP
   Security Group: mongo-sharding
   Open Ports: 22, 27017-27019(Source: id of security group mongo-sharding)
   ```

* **SSH into instance**
  
  `ssh -i <key>.pem ec2-user@<public ip>`

* **Install MongoDB:**
   
   1. Configure the package management system (yum)
   
    sudo vi /etc/yum.repos.d/mongodb-org-4.0.repo

    ``` 
    [mongodb-org-4.0]
    name=MongoDB Repository
    baseurl=https://repo.mongodb.org/yum/amazon/2013.03/mongodb-org/4.0/x86_64/
    gpgcheck=1
    enabled=1
    gpgkey=https://www.mongodb.org/static/pgp/server-4.0.asc

    ```
   2. Install MongoDB packages 
   ```
    sudo yum -y update
    sudo yum install -y mongodb-org
    sudo chkconfig mongod on
    ```  
   3. Save to AMI Image
    ``` AMI: mogodb-ami ```
 
### 2. Launch  config servers, shard servers and mongos :

```
Using mongodb-ami launch - 
Config Servers : config-server1, config-server2
Shard1 : shard-server1.1, shard-server1.2
Shard2 : shard-server2.1, shard-server1.2
```


### 3. Create Config Server Replica Set
* SSH in to configServer1
* Create data directory to store config server metadata:

  ```
   sudo mkdir -p /data/db
   sudo chown -R mongod:mongod  /data/db
   ```
* Configure mongod.conf 

  ```
  dbPath : /data/db

  net:
   port:27019
   bindIp: 0.0.0.0

  replication:
   replSetName: csr 

  sharding:
    clusterRole: configsvr
  ```

*  Start mongod daemon 
  ```
  sudo mongod --config  /etc/mongod.conf --logpath  /var/log/mongodb/mongod.log
  ```
* Check the process 
  ``` ps -aux | grep mongod```

* **Repeat all the above steps for config-server2.**

* On config-server1 open mongo shell

  ``` mongo -port 27019 ```

* Initiate replica set
  ```
  rs.initiate( {
       _id : "csr",
       members: [
          { _id: 0, host: "config-server1:27019" },
          { _id: 1, host: "config-server2:27019" }
       ]
    })
  ```
* Check replica set status

  ``` rs.status() ```

### 4. Create Sharded Replica Set 1

* SSH into shard-server1.1
* Ceate data directry
  ```
   sudo mkdir -p /data/db
   sudo chown -R mongod:mongod  /data/db
   ```
* Configure mongod.conf 

  ```
  dbPath : /data/db

  net:
   port:27018
   bindIp: 0.0.0.0

  replication:
   replSetName: rs0 

  sharding:
    clusterRole: shardsvr
  ```

*  Start mongod daemon 
   ```
   sudo mongod --config  /etc/mongod.conf --logpath  /var/log/mongodb/mongod.log
   ```
* Check the process 
  ``` ps -aux | grep mongod```

* **Repeat all the above steps for shard-server1.2.**

* On shard-server1.1 open mongo shell

  ``` mongo -port 27018 ```

* Initiate replica set
  ```
  rs.initiate( {
       _id : "rs0",
       members: [
          { _id: 0, host: "shard-server1.1:27018" },
          { _id: 1, host: "shard-server1.2:27018" }
       ]
    })
  ```
* Check replica set status

  ``` rs.status() ```
 
### 5. Create Sharded Replica Set 2
* SSH into shard-server2.1
* Ceate data directry
  ```
   sudo mkdir -p /data/db
   sudo chown -R mongod:mongod  /data/db
   ```
* Configure mongod.conf 

  ```
  dbPath : /data/db

  net:
   port:27018
   bindIp: 0.0.0.0

  replication:
   replSetName: rs1 

  sharding:
    clusterRole: shardsvr
  ```

*  Start mongod daemon 
   ```
   sudo mongod --config  /etc/mongod.conf --logpath  /var/log/mongodb/mongod.log
   ```
* Check the process 
  ``` ps -aux | grep mongod```

* **Repeat all the above steps for shard-server2.2.**

* On shard-server1.1 open mongo shell

  ``` mongo -port 27018 ```

* Initiate replica set
  ```
  rs.initiate( {
       _id : "rs1",
       members: [
          { _id: 0, host: "shard-server2.1:27018" },
          { _id: 1, host: "shard-server2.2:27018" }
       ]
    })
  ```
* Check replica set status

  ``` rs.status() ```
  
### 6. Configure Mongos
* SSH into mongos
* Configure mongod.conf 

  ```
  Comment Storage section
  net:
   port:27017
   bindIp: 0.0.0.0

  sharding:
    configDB: csr/config-server1:27019,config-server2:27019
  ```

*  Start mongos daemon 
   ```
   sudo mongos --config  /etc/mongod.conf --logpath  /var/log/mongodb/mongod.log
   ```

* Open mongo shell 

  ``` mongo -port 27017 ```

* Add the 2 shard replica set
  
  ```
  sh.addShard("rs0/shard-server1.1:27018,shard-server1.2:27018");
  sh.addShard("rs1/shard-server2.1:27018,sahrd-server2.2:27018");
  ```
* To list shards
  ``` db.adminCommand({listShards:1}) ```

* Now we can see that we have two shards.**Sharded Cluster for our replica set is created successfully.**

### Create Test Data and Enable sharding

* Create a database testdb
* Using admin database, enable sharding on testdb

  ``` db.runCommand({enablesharding:"testdb"}); ```

* To enable sharding on a collection we need shard key. Shard key determines the distribution of collection among shard
* We are using Hash based sharding : In this method collection data is paritioned into chunks and then based on the hashed shard key values in each chunk is assigned to a range.
* To create Hash shard key

  ``` db.userInfo.ensureIndex({_id:"hashed"})```

* To create shards 

  ``` sh.shardCollection("userInfo",{_id:"hashed"}) ```

* Now insert the sample data in userInfo.
* Check the distribution of sharded data 

  ``` db.users.getShardDistribution() ```


