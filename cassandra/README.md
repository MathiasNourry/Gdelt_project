<center>
  <h1>Apache Cassandra on EC2 from scratch</h1>
</center>

Once the EC2 instances are active (see corresponding tutorial if needed), we can install Apache Cassandra on each of them and establish their communication, in order to create a "Cassandra Ring".

We will use:
* AWS with security group configured
* EC2 Instances: Ubuntu 18 
* Java 8
* Apache Cassandra 3.11.9

>The steps below are to be repeated for each node of the cassandra cluster (i.e. each EC2 instance)


## Java

Cassandra runs on a JVM, so it is necessary to install Java before you can install and configure Cassandra. We use `Java 8` for compatibility with our version of Cassandra.

Update the package index on the EC2 instance :
``` shell
$ sudo apt update
``` 

Install Java version 8 :
``` shell
$ sudo apt install openjdk-8-jre-headless
```

Update the environment variables by adding the following lines to the `~/.bashrc` file :
```
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
export JRE_HOME=$JAVA_HOME/jre
export PATH=$PATH:$JAVA_HOME/bin:$JAVA_HOME/jre/bin
```

Then run the following command in the terminal :
``` shell
$ source ~/.bashrc
```

Check the version of Java installed with the command below :
``` shell
$ java -version
```

The result should be similar to the one below :
```
openjdk version "1.8.0_275"
OpenJDK Runtime Environment (build 1.8.0_275-8u275-b01-0ubuntu1~18.04-b01)
OpenJDK 64-Bit Server VM (build 25.275-b01, mixed mode)
```


## Cassandra - installation
Download the latest version of Cassandra from https://cassandra.apache.org/ and unpack the resulting archive. In our case, we will use the version `3.11.9` (January 2021) :
``` shell
$ wget https://mirrors.ircam.fr/pub/apache/cassandra/3.11.9/apache-cassandra-3.11.9-bin.tar.gz
$ tar -xzf apache-cassandra-3.11.9-bin.tar.gz
$ rm apache-cassandra-3.11.9-bin.tar.gz
```

Run the Cassandra daemon with the following command :
``` shell
$ ./apache-cassandra-3.11.9/bin/cassandra
```

In another terminal, on the same EC2 instance, check the status of the node thus created with the command :
``` shell
$ ./apache-cassandra-3.11.9/bin/nodetool status
```

The expected result is as follows :
```
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address    Load       Tokens       Owns (effective)  Host ID                               Rack
UN  127.0.0.1  70.7 KiB   256          100.0%            c180d502-ed1a-4b30-880d-84e7cbde8be3  rack1
```

**Congratulations! The `UN` symbol means that cassandra is active on this node** (UN = Up and Normal). It remains to configure the communication between the nodes.


## Cassandra - communication configuration

In order to ensure good communication between nodes, it may be necessary to follow a certain order in launching cassandra daemons:
- configure a first node (node 1) (installation of cassandra + configuration of the communication)
- run the cassandra daemon on this node 1, and leave the daemon active
- install cassandra and configure the communication on node 2
- run the cassandra daemon on node 2
- check the good communication between nodes 1 and 2

To configure the communication between the different nodes of the cluster, let's set up Cassandra on the instances. For this, 2 files, which are located in the `/apache-cassandra-3.11.9/conf/` directory, will be useful to us :
* `cassandra.yaml`
* `cassandra-rackdc.properties`

To modify files with admin rights, use the following command :
``` shell
$ sudo nano <nom_fichier>
```

_Note: Save the file with `ctrl+x` then confirm with `y` then `Enter`_


### 1. cassandra.yaml
__Seeds__

When a Cassandra node joins the ring, it uses the data of a current node to instantiate itself based on it. 
The first node will have its own (private) IP address as seed.  
The following nodes will initially have the (private) IP address of this first node.  
Then, once the first communications are established, we will be able to modify this list of addresses in order to add the (private) IP addresses of several nodes in the cluster.
Example of configuration for node 1 :
```
seed_provider:
    - class_name: org.apache.cassandra.locator.SimpleSeedProvider
      parameters:
          # seeds is actually a comma-delimited list of addresses.
          # Ex: "<ip1>,<ip2>,<ip3>"
          - seeds: "172.31.51.42,34.207.61.42,35.175.104.58"
```

__Listen address__

Set the instance's listening port, passing the instance's private IP to the `listen_address` field :
```
listen_address: 172.31.51.42
```

__Snitch__

Since we are working on AWS and more precisely, on a single zone (mandatory for AWS Educate accounts), we can specify the `Snitch` parameter :
```
endpoint_snitch: Ec2Snitch
```

__rpc__

Change the rpc address again with the private IP of the instance and the rpc_port (if needed) :

```
rpc_address: 172.31.51.42
rpc_port: 9160
```


### 2. cassandra-rackdc.properties

Comment all the lines of the file so that the snitch data considered are those set in the previous file.
&nbsp;
**Repeat the previous steps for each node of the cluster**
&nbsp;

On each node of the cluster, run the cassandra daemon :
``` shell
$ ./apache-cassandra-3.11.9/bin/cassandra
```

Check the state of the cluster, to make sure that the nodes are communicating with each other with the following command :
``` shell
$ ./apache-cassandra-3.11.9/bin/nodetool status
```
The expected result is : 
```
Datacenter: us-east
===================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address        Load       Tokens       Owns (effective)  Host ID                               Rack
UN  172.31.63.156  70.68 KiB  256          68.4%             d124eab5-7a17-497d-8a24-e85b629b194d  1e
UN  172.31.50.107  70.77 KiB  256          68.5%             8dd5ad48-0466-429b-bfc7-21b8d6234000  1e
UN  172.31.61.170  70.72 KiB  256          63.1%             a43831e3-d90a-4952-865d-d728e6ed4d64  1e
```

**Some useful commands**   
Checking the cassandra launch log :
``` shell
$ tail /var/log/cassandra/system.log
```

Killing a cassandra process on a node :
``` shell
$ pkill -f 'java.*cassandra'
```

After the cassandra process is stopped, the status of the node in question should appear `DN` when the `nodetool status` command is executed :
```
Datacenter: us-east
===================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address        Load       Tokens       Owns (effective)  Host ID                               Rack
UN  172.31.71.95   12.48 MiB  256          100.0%            ae0cd4e5-7f86-4502-a527-e728c32af8bc  1f
DN  172.31.75.249  12.46 MiB  256          100.0%            c97a67f3-3be5-491c-ad30-30cc7ae00776  1f
DN  172.31.79.24   12.46 MiB  256          100.0%            05a936ee-e9e8-414a-b7c5-42a0d87d1475  1f
```

_Sources : 
https://linuxize.com/post/how-to-install-apache-cassandra-on-ubuntu-18-04/_
