# Performance at scale with AWS.

**Topics**

- ASW CFM template
- Minimize errors on scaling systems
- Catching strategies with AWS elastic cache

  

# CloudFormation stack that implements EC2 autoscaling group and EFS FileSystem
    
## Auto scaling template requisites
 
- [AWS Account](https://aws.amazon.com)

- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)

- [AWS CLI CONF](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html)

 ## Creating stacks

Download the YAML file and use the AWS CloudFormation Console to run the templates. Click the "Create Stack" button in the upper left corner of the console, then under "Choose a template", select "Upload a template to Amazon S3" and click "Browse" to find your local fork of this repository and choose the template you want to run.

## Note

To create a file system, the only requirement is that you create a token to ensure idempotent operation. If you use the console, it generates the token for you. For more information, see [CreateFileSystem](https://docs.aws.amazon.com/efs/latest/ug/API_CreateFileSystem.html).
  
### EFS mount targets:

The default target is Default:

```sh
“nfs.example.com:/data”
```

### If you want to change the target:

AWS Linux/ Redhead/CentOS

  ```sh
sudo yum -y install nfs-utils
sudo mkdir /mnt/efs
sudo mount -t nfs -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport $mount-target-ip-address:/ /mnt/efs
```
Debian/ UBUNTU

```sh
sudo apt-get install -y nfs-common
sudo mkdir /mnt/efs
sudo mount -t nfs
-o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport $mount-target-ip-address:/ /mnt/efs
```

  

# Minimize errors on scaling systems.

## Auto scaling group to responding to rapid and dramatic changes in load.

  

*1* **Resource Monitoring**

  We need to identify how busy in terms of CPU, network traffic and other metric our instances are, in order to make scaling decisions, we can use AWS cloud watch to help us to measure and track our instances.  

**2** Now that we are monitoring the metrics on our instances, we need to know when to dictate a scale out or scale in operation. This also cloud watch can handle it.

  

![enter image description here](https://user-images.githubusercontent.com/12648295/97328004-764a1300-186d-11eb-871e-866749f68023.png)

  

**3** **Scaling Actions**

This is the final step and takes action when the alarm is raised and is handled by auto-scaling, as directed by cloud watch alarm to create systems that can do a better job of responding to rapid and dramatic changes in load.

  

![enter image description here](https://user-images.githubusercontent.com/12648295/97328084-8e219700-186d-11eb-8b4f-a9c8990187a9.png)

  
For example if you try to keep average CPU below 50%  

You can have standard response for modest branch (50% - 60%) 

Two more for some bigger breaches (60% - 70%)  

And a super aggressive one for utilization that exceed (80%)  

In this case you can have a fixed number of instances to the group (1, 2, 4, 8), increasing the count by 50%, 100%, 150% and 200%. 

Take action instances:


1 instances when 50 <- CPU utilization 60

2 instances when 60 <- CPU utilization 70

4 instances when 70 <- CPU utilization 80

8 instances when 80 <- CPU utilization infinity

(Instances need 300 second to warm up after each step)

And you can also define a similar set of increasingly aggressive policies for scaling down.
Step policies continuously evaluate the alarms during a scaling activity and while unhealthy instances are being replaced with new ones. This allows for faster response to changes in demand. Let’s say the CPU load increases and the first step in the policy is activated. During the specified warm up period (300 seconds in this example), the load might continue to increase and a more aggressive response might be appropriate.

**Conclusion**

Fortunately, Auto Scaling is in violent agreement with this sentiment and will switch in to high gear (and use one of the higher steps) automatically. If you create multiple step scaling policies for the same resource (perhaps based on CPU utilization and inbound network traffic) and both of them fire at approximately the same time, Auto Scaling will look at both policies and choose the one that results in the change of the highest magnitude

# Whats Amazon ElastiCache for Redis.

Deploying Redis makes use of familiar concepts such as clusters and nodes. However, Redis has a few important differences compared with Memcached:
  
• Redis data structures cannot be horizontally shared. As a result, Redis ElastiCache clusters are always a single node, rather than the multiple nodes we saw with Memcached.

• Redis supports replication, both for high availability and to separate read workloads from write workloads. A given ElastiCache for Redis primary node can have one or more replica nodes. A Redis primary node can handle both reads and writes from the app. Redis replica nodes can only handle reads, similar to Amazon RDS Read Replicas.

• Because Redis supports replication, you can also fail over from the primary node to a replica in the event of failure. You can configure ElastiCache for Redis to automatically fail over by using the Multi-AZ feature.

• Redis supports persistence, including backup and recovery. However, because Redis replication is asynchronous, you cannot completely guard against data loss in the event of a failure. We will go into detail on this topic in our discussion of Multi-AZ.
 
## Architecture.

An ElastiCache for Redis replication group consists of a primary and up to five read replicas. Redis 
asynchronously replicates the data from the primary to the read replicas. Because Redis supports persistence, it is technically possible to use Redis as your only data store. In practice, customers find that a managed database such as Amazon DynamoDB or Amazon RDS is a better fit for most use cases of long-term data storage. ElastiCache for Redis has the concept of a primary endpoint, which is a DNS name that always points to the current Redis primary node. If a failover event occurs, the DNS entry will be updated to point to the new Redis primary node. To take advantage of this functionality, make sure to configure your Redis client so that it uses the primary endpoint DNS name to access your Redis cluster.


![enter image description here](https://user-images.githubusercontent.com/12648295/97328179-aa253880-186d-11eb-9349-b0fae4e21926.png)
  

## Distributing Reads and Writes.

Using read replicas with Redis, you can separate you read and write workloads. This separation lets you scale reads by adding additional replicas as your application grows. In this pattern, you configure your application to send writes to the primary endpoint. Then you read from one of the replicas, as shown in the following diagram. With this approach, you can scale your read and write loads independently, so your primary node
only has to deal with writes

  

![enter image description here](https://user-images.githubusercontent.com/12648295/97328145-9ed20d00-186d-11eb-9ff7-bbf31a2ff181.png)
  

## Multi-AZ with Auto-Failover.

Amazon ElastiCache can be configured to automatically detect the failure of the primary node, select a read replica, and promote it to become the new primary. ElastiCache auto-failover will then update the DNS primary endpoint with the IP address of the promoted read replica. If your application is writing to the primary node endpoint as recommended earlier, no application change will be needed. Depending on how in-sync the promoted read replica is with the primary node, the failover process can take several minutes. First, ElastiCache needs to detect the failover, then suspend writes to the primary node, and finally complete the failover to the replica. During this time, your application cannot write to the Redis ElastiCache cluster.
Architecting your application to limit the impact of these types of failover events will ensure greater overall availability.

  

## Sharding with Redis.

Redis has two categories of data structures: simple keys and counters, and multidimensional sets, lists, and hashes. The bad news is the second category cannot be shared horizontally. But the good news is that simple keys and counters can. In the simplest case, you can treat a single Redis node just like a single Memcached node. Just like you might spin up multiple Memcached nodes, you can spin up multiple Redis clusters, and each Redis cluster is responsible for part of the shared dataset.

  

![enter image description here](https://user-images.githubusercontent.com/12648295/97328223-b3160a00-186d-11eb-9247-482c71b00492.png)

  

You can also combine horizontal sharing with split reads and writes. In this setup, you have two or more Redis clusters, each of which stores part of the key space. You configure your application with two separate sets of Redis handles, a write handle that points to the shared masters and a read handle that points to the shared replicas. Following is an example architecture, this time with Amazon DynamoDB rather than MySQL, just to illustrate that you can use either one:

![enter image description here](https://user-images.githubusercontent.com/12648295/97328244-bc06db80-186d-11eb-95a6-f3e928dbaa34.png)  

## Recommendation Engines.

 Some algorithms, such as **Slope One**, are simple and effective but require in-memory access to every item ever rated by anyone in the system. Even if this data is kept in a relational database, it has to be loaded in memory somewhere to run the algorithm.

Redis data structures are a great fit for recommendation data. You can use Redis counters used to increment or decrement the number of likes or dislikes for a given item. You can use Redis hashes to maintain a list of everyone who has liked or disliked that item, which is the type of data that SlopeOne requires.

  [SlopeOne](https://www.wikiwand.com/en/Slope_One)

## Monitoring Cache Efficiency.

To begin, see the Monitoring Use with CloudWatch topic for Redis and Memcached,as well as the Which Metrics Should I Monitor? 

Most importantly, watch CPU usage. A consistently high CPU usage indicates that a node is overtaxed, either by too many concurrent requests, or by performing dataset operations in the case of Redis. For Redis, ElastiCache provides two different types of metrics for monitoring CPU usage: **CPUUtilization** and **EngineCPUUtilization**. 

Because Redis is single-threaded, you need to multiply the CPU percentage by the number of cores to get an accurate measure of CPUUtilization. For smaller node types with one or two vCPUs, use the CPUUtilization metric to monitor your workload. 

**Note:**

For larger node types with four or more vCPUs, its recommend monitoring the EngineCPUUtilization metric, which reports the percentage of usage on the Redis engine core.

  

## Memcached Memory Optimization.
 
When you launch an ElastiCache cluster, the max_cache_memory parameter is set for you automatically, along with several other parameters. The key parameters to keep in mind are chunk_size and chunk_size_growth_factor, which work together to control how memory chunks are allocated.

## Cluster Scaling and Auto Discovery.

Scaling your application in response to changes in demand is one of the key benefits of working with AWS.  
Configuring their client with a list of node DNS endpoints for ElastiCache works perfectly fine. But let's look at how to scale your ElastiCache Memcached cluster while your application is running, and how to set up your application to detect changes to your cache layer dynamically.

## Auto Scaling Cluster Nodes.

Amazon ElastiCache does not currently support using Auto Scaling to scale the number of cache nodes in a cluster. To change the number of cache nodes, you can use either the AWS Management Console or the AWS API to modify the cluster. You usually don't want to regularly change the number of cache nodes in your Memcached cluster. Any change to your cache nodes will result in some percentage of cache keys being remapped to new (empty) nodes, which means a performance impact to your application. Even with consistent hashing, you will see an impact on your application when adding or removing nodes.

## Client Libraries and Consistent Hashing.

[PHP phpredis](https://github.com/phpredis/phpredis)

[PHP Predis ](https://github.com/predis/predis)

**Note**:

Redis libraries to support consistent hashing. Redis libraries rarely support consistent hashing because the advanced data types that we discussed preceding cannot simply be horizontally shared across multiple Redis nodes.

Redis as a technology cannot be horizontally scaled easily. Redis can only scale up to a larger node size, because its data structures must reside in a single memory image in order to Perform properly.


**Conclusion**

The proper use of in memory caching can result in an application that performs better and cost less at scale.

Using the cache strategies in mention before to increase the performance and the resiliency of the application

Chancing the configuration on Elastic Cache to add or remove nodes as your app need to change over time in order to get most of the performance of you in memory.  

License.
----
MIT

# References.

  

- [AWS Elastic ache Redis](https://aws.amazon.com/elasticache/redis/)

- [AWS NFS](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-redis-server-as-a-session-handler-for-php-on-ubuntu-14-04)

- [AWS Autoscaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ts-as-capacity.html)
