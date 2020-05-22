## Week3 (10th March 19 - 16th March'19)
#### Using RiakDB as AP NoSQL Databse

- Decided to create Riak cluster of 5 nodes across two VPC's in two different regions.
- In order to connect nodes in two VPC's, we need to do vpc peering.
- Researched on how to create VPC peering and how establish connection between private nodes in two different regions.

***

## Week 5 (24th March'19 - 30 March'19)

Architecture used for creating Riak Cluster -
<img src="https://github.com/nikitabairagi/Personal-Project/blob/master/RiakDB_cluster_topology.png" width="90%">

## Steps

### 1. Setting up Jumpbox -
Launced a new EC2 instance in vpc1-cmpe281.

  ```
  Amazon Linux AMI
  T2 micro instance
  VPC : vpc1-cmpe28
  Subnet: Public subnet
  Auto Assign Public IP : Enable
  Security Group: jumpbox
  open ports: 22
  ```

### 2. Setting up 3 Riak nodes in vpc1-cmpe281(Oregon region) and 2 Riak nodes in vpc2-cmpe281(N.Virgina region)
Launched 5 Riak instatnces

  ```
  AMI : Riak KV 2.2 Series
  Instance Type : t2.micro
  VPC : vpc1-cmpe281(for Oregon region) / vpc2-cmpe281(for N.Virginia region)
  Network : Private Subnet
  Auto Assign Public IP : No
  Security Groups : riak cluster
  open ports : 22, 8087, 8089
  key-pair : cmpe281-us-west-1
  ```
### 3. VPC Peering 
In order to connect the vpc's in 2 regions we need to create a peering connection.
* Got to VPC Dashboard and select Peering Connections -> Create Peering Connection

  ``` 
  Peering connection name tag : RiakLink
  VPC (Requester)* : vpc1-cmpe281
  Region : Another region (N.virginia)
  VPC (Accepter)* :  vpc id of vpc2-cmpe281 in N.Virgina Region
  ```
* Now go to VPC Dashboard in N.Virginia region. Select Peering Connections. You will see the peering connection that we just created. Go to Actions->Accept request. 
* Peering connection is now established.

### 4. Now to be able to SSH from our jumpbox to private instance in vpc2, we need to update the route tables in both regions 
* Select the route tables -> select route table for public subnet -> Edit routes.
* Add the following route to route table

  |Destination | Target|
  | ------------- |:-------------:| 
  |172.0.0.0/16 (CIDR range for vpc2-cmpe281)  | RiakLink | 

* In the same way we need to update the route table for vpc2 private subnet

  |Destination | Target|
  | ------------- |:-------------:| 
  |10.0.0.0/16 (CIDR range for vpc1-cmpe281)  | RiakLink | 

### 5. Also to we need instances in vpc1 private subnet to communicate with instances in vpc2 private subnet.
* Add the following routes in vpc1 private subnet route table

  |Destination | Target|
  | ------------- |:-------------:| 
  |172.0.1.0/24 (CIDR range for private subnet of vpc2-cmpe281)  | RiakLink | 

* Add the following routes in vpc2 private subnet route table

  |Destination | Target|
  | ------------- |:-------------:| 
  |10.0.1.0/24 (CIDR range for private subnet of vpc1-cmpe281)  | RiakLink | 

### 6.We also need to update the security groups

**Update Security group 'riak-cluster' in Oregon region -**

  |Type | Port Range| Source | Description |
  | ------------- |:-------------:| :-------------:| :-------------:| 
  |All ICMP - IPv4| N/A | 172.0.1.0/24| To ping instances in vpc2 private subnet|
  |Custom TCP Rule| 8098 | 172.0.1.0/24 | Http Interface | 
  |All TCP| 0-65535 | riak-cluster id | For communication between riak nodes in vpc1 |
  |All TCP| 0-65535 | 172.0.1.0/24 | For communication between riak nodes in vpc1 and riak nodes in vpc2 |

**Update Security group 'riak-cluster' in N.Virginia region -**

  |Type | Port Range| Source | Description |
  | ------------- |:-------------:| :-------------:| :-------------:| 
  |All ICMP - IPv4| N/A | 10.0.1.0/24| To ping instances in vpc1 private subnet|
  |Custom TCP Rule| 8098 | 10.0.1.0/24 | Http Interface | 
  |All TCP| 0-65535 | 10.0.1.0/24 | For communication between riak nodes in vpc1 and riak nodes in vpc2 |
  |SSH| 22 | 10.0.0.0/16 | For SSH through jumpbox in vpc1 |
  

### 7. Set up the Riak Cluster 
* SSH into riak instance via Jumpbox -
```
ssh -i <key>.pem ec2-user@<public ip>  (access jump box)
ssh -i <key>.pem ec2-user@<private ip> (access riak node)

10.0.1.42    Riak1
10.0.1.168   Riak2
10.0.1.62    Riak3
171.0.1.249  Riak4
172.0.1.51   Riak5
```
* Configure the  all Riak nodes - 
```
In /etc/riak, you should see the following settings in riak.conf after deploying the AMI.

Node 1 Config (riak.conf):

listener.http.internal = 10.0.1.42:8098
listener.protobuf.internal = 10.0.1.42:8087
nodename = riak@10.0.1.42

Start Node 1:

sudo riak start
sudo riak ping
sudo riak-admin status
```
* Do the same for all the riak nodes
* To create the cluster we will connect Riak2, Riak3, Riak4, Riak5 nodes to Riak1 node. 
* Execute following command on Riak2, Riak3, Riak4, Riak5 nodes:

  ```sudo riak-admin cluster join riak@<ip.of.first.node>```

* After all of the nodes are joined, execute the following:

   ``` 
   sudo riak-admin cluster plan
   sudo riak-admin cluster status
   ```
* If this looks good

   ``` sudo riak-admin cluster commit```

* To check the status of the cluster :

  ```sudo riak-admin member_status```
   
***
### EXPERIMENTS
### Normal Behavior

* To test the behavior of riak cluster without any partition, we will insert a key-value pair in Riak1 node.

  ```curl -v -XPUT -d '{"name":"nikita"}' http://riak-node2:8098/buckets/bucket/keys/name?returnbody=true ```
  
*  Now we will check if the recently inserted data is available in all nodes of the cluster
  ```curl -i http://riak-node3:8098/buckets/bucket/keys/name```
  
All the nodes shows the data that was inserted in first node. 

### Creating Partition

* To create the partition we will break the vpc peering connection. We will delete the following route from private route table of vpc1. 

  ```172.0.1.0/24 - vpc2-cmpe281``` 
 
* Now the riak nodes in vpc2 will become unreachable.
* Insert the new data in first node

  ```curl -v -XPUT -d '{"role":"student"}' http://riak-node2:8098/buckets/bucket/keys/role?returnbody=true ```
  
* Now we will check the data in all the nodes. 
* It is observed that the updated data was available in Riak2 and Riak3 nodes. But Riak4 and Riak5 nodes didn't show recent data.

### Partition Recovery

* To recover from partition we will again establish the peering connection by adding the route again:

  ```172.0.1.0/24 - vpc2-cmpe281``` 
 
* Now we will check the data in all the nodes again.

  ```curl -v -XPUT -d '{"role":"student"}' http://riak-node2:8098/buckets/bucket/keys/role?returnbody=true ```
  
* It is observed that Riak4 and Riak5 nodes now show the updated data.


  | Questions | Results   | 
  | ------------- |:-------------:| 
  | How does the system function during normal mode (i.e. no partition)     | During normal mode when data is inserted in first node, all the other nodes gets the updated data. And we were able to read updated data in all riak nodes.  |
  | What happens to the master node during a partition?      | Primary node remains unaffected    |  
  | Can stale data be read from a slave node during a partition?  | Stale data can be read from slave nodes. But recent data that was inserted in raik1 node after partition, was not available in other riak nodes.       |  
  | What happens to the system during partition recovery?| After partition recovery, the updated data eventually becomes available in all nodes. |








 
 


  
