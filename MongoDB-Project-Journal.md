# CMPE 281 - Personal NoSQL Project
## Objective : Testing the partition tolerance of one CP and one AP NoSQL Database


## Week1 (23rd Feb'19 - 2nd March'19)
#### Using MongoDB as CP NoSQL Databse

#### 1. Configuration I had before starting MongoDB Cluster
  EC2 Key Pair named : cmpe281-us-west-2 \
  VPC in region US West (Oregon) - US West 2
  	
  ```
  VPC name : cmpe281
  wizard option : public with private subnet
  ```
#### 2. Setting up MongoDB cluster on AWS 
  Setting up MongoDB AMI -

- Created a Mongo Instance :  Launced a  Ubuntu Server 16.04 LTS 
    ```AMI : Ubuntu Server 16.04 LTS(HVM)
	 Instance type : t2.micro
	 VPC : cmpe281
	 Network : Public Subnet
	 Auto Public IP : no
	 Security Group : mongodb-cluster
	 SG open ports : 22,27017
	 Key Pair : cmpe281-us-west-2
   ```
	   
- Allocating and assigning an Elastic IP to Mongo Instance
    ``` 
    Allocate Elastic IP : Scope VPC
    Associate this Elastic IP to Mongo Instance
    ```
    
- SSH into Mongo Instance \
   `ssh -i <key>.pem ubuntu@<public ip>`
      
- Installed MongoDB
    ```
    sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 9DA31620334BD75D9DCB49F368818C72E52529D4
   
   echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/4.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb.list
    
    sudo apt update
    sudo apt install mongodb-org
    ```
- For replicas of the cluster to communicate they need to authenticate themselves to the replica set. For this, 
created a key file to serve as shared password between the replicas.
    ```
    openssl rand -base64 741 > keyFile
    sudo mkdir -p /opt/mongodb
    sudo cp keyFile /opt/mongodb
    sudo chown mongodb:mongodb /opt/mongodb/keyFile
    sudo chmod 0600 /opt/mongodb/keyFile
    ```
- Configured mongod.conf \
    `sudo vi /etc/mongod.conf`
    
   - remove or comment out bindIp: 127.0.0.1 and replace with bindIp: 0.0.0.0 (binds on all ips)
     ```
     # network interfaces
     net:
        port: 27017
        bindIp: 0.0.0.0
      ```
   - Uncomment security section & add key file
     ```
     security:
        keyFile: /opt/mongodb/keyFile
     ```
   - Uncomment Replication section. Name Replica Set = cmpe281
     ```
     replication:
        replSetName: cmpe281
     ```

- Created mongod.service \
    `sudo vi /etc/systemd/system/mongod.service`
    ```
    [Unit]
        Description=High-performance, schema-free document-oriented database
        After=network.target

    [Service]
        User=mongodb
        ExecStart=/usr/bin/mongod --quiet --config /etc/mongod.conf

    [Install]
        WantedBy=multi-user.target
    ```
- Enable Mongo Service
    `sudo systemctl enable mongod.service`
    
- Restart MongoDB to apply the changes
    ```
    sudo service mongod restart
    sudo service mongod status
    ```
- Saved to AMI Image : `AMI : mongo-ami`

***

## Week 2 (3rd March'19 - 9th March'19)

Created a replication set of 5 nodes. Below is the architecture(master-slave model) of Replica set. All the write operations in replica set are recieved by primary node. All the secondary nodes replicate the dataset of primary node to reflect the primary node data.

<img src="https://github.com/nguyensjsu/cmpe281-nikitabairagi/blob/master/MongoDb_topology.png" width="65%">

**Steps :**
* Using mongo-ami image launced 5 free tier instances - Primary, Secondary1, Secondary2, Secondary3, Secondary4.
* Associate each instance with their own Elastic IPs.
* For each instance edit /etc/hosts and add local host name for Public IPs. 
  ```
  34.218.116.115  Primary
  54.69.181.161   Secondary1
  52.35.188.45    Secondary2
  54.70.181.29    Secondary3
  34.211.133.108  Secondary4
  ```
* Change the hostnames on each instance
  ```
  sudo hostnamectl set-hostname <new hostname>
  sudo hostname -f 
  reboot (to make sure the change sticks)
  ```
* Initialize the Replica Set
   ```
   mongo  
   rs.initiate( {
       _id : "cmpe281",
       members: [
          { _id: 0, host: "Primary:27017" },
          { _id: 1, host: "Secondary1:27017" },
          { _id: 2, host: "Secondary2:27017" },
          { _id: 3, host: "Secondary3:27017" },
          { _id: 4, host: "Secondary4:27017" }
       ]
    })
   ```
* Check replica Set Status \
 ` rs.status()`

* Create admin user to access the database
  ```
  mongo
  use admin
  db.createUser( {
        user: "admin",
        pwd: "*****",
        roles: [{ role: "root", db: "admin" }]
  });
  ```
  

### **EXPERIMENTS**

### **Normal Behaviour**

Now that the replication set of MongoDB instances is created we can test its partition tolerance. In MongoDB replica set all nodes maintain the same data. For testing the behavior of replica set without any partition,following steps are performed-
*  Login to Primary node as admin \
  `mongo -u admin -p <password> --authenticationDatabase admin`

* Insert a document in admin database. Also note that we can only insert the data from primary node,because its a master slave model and only master is allowed to do write operations.
  ```
  use admin
  db.products.insert({item:"envelopes",qty:100,type:"Clasp"})
  db.products.find()  //find the document
  ```
* SSH into all the secondary nodes.For allowing queries in secondary nodes we need to enable 'slave Ok' mode and authorize each node. And then find the document in each secondary nodes.
  ```
  db.getMongo().setSlaveOk()
  use admin
  db.auth('admin','*****')
  db.products.find()
  ```
All the nodes shows the document that was inserted from primary node. This shows that all the nodes are consistent and available.

### **Creating Partition**

**TestCase 1 - Partitioning Secondary2 node from the replicat set -** 

* To cause a network partition I used Linux firewall rules to drop the incoming traffic on Secondary2 node. By doing this, Secondary2 node will not accept any updates from other nodes in replica set.
  ```
  sudo iptables -A INPUT -s 34.218.116.115 -j DROP // drop traffic from Primary 
  sudo iptables -A INPUT -s 54.69.181.161 -j DROP // drop traffic from Secondary1 
  sudo iptables -A INPUT -s 54.70.181.29 -j DROP  // drop traffic from Secondary3
  sudo iptables -A INPUT -s 34.211.133.108 -j DROP  // drop traffic from Secondary4 
  sudo iptables -L --line-numbers //  list of rules

  ```

* To test that partition is created, update a document in the primary node \
  `db.products.findAndModify({update:{item:"envelopes",qty:90,type:"Clasp"}})`

* Now find the document in all the secondary nodes.It is found that all the secondary nodes shows the updated document except the Secondary2 node. Seoncdary2 node shows the stale or inconsistent data. 
* Hence, after the partition Secondary2 node becomes unreachable/unavailable.

**TestCase 2 - Partitioning Secondary2 and Primary(master) node from the replica set -** 

* Now we partitioned Primary Node as well as Secondary2 node from the replica set, using the same firewall rules, dropping the traffic from Secondary1, Secondary3, and Secondary4 node.

* It is observed that replica set decided to elect Secondary3 node as master node. So when we insert a new document in Secondary3 node, Secondary1 and Secondary4 node recieves the update. But Primary and Secondary2 node reflects stale data.

### **Partition Recovery**

* Now to recover from partition, we will allow the traffic on Secondary2 and Primary node again.
  ```
  sudo iptables -L --line-numbers
  sudo iptables -D INPUT 1 // execute 4 times to remove the 4 rules that was added
  ```
* As soon as we recover Primary and Secondary2 node from partition, these nodes get synchronized with other nodes and gets the updated data set. However **Secondary3 remains the master node.**


  | Questions | Results   | 
  | ------------- |:-------------:| 
  | How does the system function during normal mode (i.e. no partition)     | In the normal mode,when a write request in sent to primary node, the writes are replicated to all the nodes. | 
  | What happens to the master node during a partition?      | If master node is partitioned from the replica set,a new master node is elected. And previous master node becomes secondary node and becomes unreachable    |  
  | Can stale data be read from a slave node during a partition?  | We can read the stale data from slave node during partition      |  
  | What happens to the system during partition recovery?| During partition recovery the data is synchronized across all the nodes and system becomes consistent    |








 
 


  
