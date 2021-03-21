<center>
  <h1>Apache Zeppelin - Cassandra</h1>
</center>

Zeppelin Notebook offers a very practical environment for data analysis. Several connectors and modular interpreters can be added, in order to use Spark, SQL, Scala or Cassandra.

We will use the version `0.8.2`, which ensures compatibility with the version of Cassandra used in the NoSQL environment.

__Connection to instances__

Connect to the EC2 instance (Ubuntu 18 in our case) on which to install Apache Zeppelin via an `ssh` tunnel, redirecting the listening port to allow access to the notebook via a local web browser :

``` shell
$ ssh -L 8890:127.0.0.1:8080 -i \<chemin-vers-keyPair.pem\> ubuntu@\<DNS-address-of-master\>
```

> We redirect the port `8080` of the EC2 instance to the port `8890` of the local machine.


## Query the Cassandra ring via Apache Zeppelin

In the following steps, Zeppelin is installed on one of the instances containing cassandra, and the Cassandra cluster contains 3 nodes. Zeppelin uses the native version of spark (available from the zeppelin packages), and not a spark cluster deployed from scratch.

_[Edit] : zeppelin notebook configurations can be done on a local version (rather than on an EC2 instance)._


### Zeppelin

To make requests on the cassandra ring, we will use a Zeppelin notebook configured with a **spark-cassandra connector**.

__Install__

To install Zeppelin, we download the packages with the following command :
``` shell
$ wget https://downloads.apache.org/zeppelin/zeppelin-0.8.2/zeppelin-0.8.2-bin-all.tgz
$ tar xzf zeppelin-0.8.2-bin-all.tgz
$ rm zeppelin-0.8.2-bin-all.tgz
```

__Launch__

Start the Zeppelin daemon on the EC2 instance :
``` shell
$ cd zeppelin-0.8.2-bin-all/bin/
$ ./zeppelin-daemon.sh start
```

__Connect__

The redirection of the listening port was done at the time of the connection to the EC2 instance. So you just have to go in a browser and enter the address below :
```
localhost:8890
```

<p align="center">
  <img src="img/zeppelin_web.png" width="700" />
</p>

> **Bravo !** Apache Zeppelin is installed. However, it is now necessary to configure the connection with cassandra.


### Connect Zeppelin to Cassandra

_ **Note** : In the case of a connection from a local computer to the cassandra ring (stored on an EC2 cluster), you will have to replace the private IPs of the cluster nodes by the **public** IPs._

It is a question of modifying the configurations of the `Interpreter menu` of Zeppelin (click on the wheel of configuration in top on the right of the page of the notebook zeppelin) in order to make the link between the cluster cassandra and the notebook used.

__Spark interpreter__

Select the spark interpreter thanks to the search bar of the interpreter menu, and make the following modifications :
1. Add a pointer to the list of private IP addresses of the cassandra cluster by creating a new line containing :
    ```
    spark.cassandra.connection.host           <IP cassandra>
    ```
2. Add the Spark-Cassandra connector as a dependency in the interpreter. The paths can be modified depending on the operating system used (see below for the directory where to copy the `.jar`). In all cases, enter the **complete** path.

<p align="center">
  <img src="img/dependances.png" width="700" />
</p>

If the dependencies are not available, download the jar at the following addresses :
* https://mvnrepository.com/artifact/com.datastax.spark/spark-cassandra-connector_2.11/2.0.12
* https://mvnrepository.com/artifact/com.twitter/jsr166e/1.1.0

or directly via a terminal on the cluster machines (the `interpreter` directory is located in the `zeppelin` folder downloaded via binary files) :
``` shell
$ cd ~/zeppelin-0.8.2-bin-all/interpreter/spark/dep
$ wget https://repo1.maven.org/maven2/com/datastax/spark/spark-cassandra-connector_2.11/2.0.12/spark-cassandra-connector_2.11-2.0.12.jar
$ wget https://repo1.maven.org/maven2/com/twitter/jsr166e/1.1.0/jsr166e-1.1.0.jar
```

_**Note** : click on the "+" button on the right of the additions so that the interpreter takes the modifications into account._

__Cassandra interpreter__

Select the cassandra interpreter thanks to the search bar of the interpreter menu and add a pointer to the list of private IP addresses of the cassandra cluster by creating the line :
```
cassandra.hosts           <IP cassandra>
```

__Librairires__

To finalize the connection between spark and cassandra, in a notebook cell, import the libraries related to the installed connectors :
``` scala
import com.datastax.spark.connector._
import org.apache.spark.sql.cassandra._
```


> **Et voilà !** Apache Zeppelin is installed and configured to communicate with your Cassandra ring.
