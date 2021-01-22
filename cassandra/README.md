# Apache Cassandra sur EC2 from scratch

Une fois les instances EC2 actives (voir tutoriel correspondant si besoin), nous pouvons installer Apache Cassandra sur chacunes d'elles et établir leur communication, afin de créer un "Ring Cassandra".

Nous utiliserons :
* AWS avec groupe de sécurité configuré
* Instances EC2 : Ubuntu 18 
* Java 8
* Apache Cassandra 3.11.9

>Les étapes ci-dessous sont à répéter pour chaque noeud du cluster cassandra (i.e chaque instance EC2)

## Pré-requis

### Java

Cassandra s'exécute sur une JVM, il est donc nécessaire d'installer Java avant de pouvoir installer et configurer Cassandra. Nous utilisons `Java 8` dans un souci de compatibilité avec notre version de Cassandra.

Mettre à jour l'index des packages sur l'instance EC2
``` shell
sudo apt update
``` 
Installer la version 8 de Java
``` shell
sudo apt install openjdk-8-jre-headless
```
Mettre à jour les variables d'environnement en ajoutant les lignes ci dessous dans le fichier `~/.bashrc`
```
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
export JRE_HOME=$JAVA_HOME/jre
export PATH=$PATH:$JAVA_HOME/bin:$JAVA_HOME/jre/bin
```
Puis lancer la commande suivante dans le terminal
``` shell
source ~/.bashrc
```
Vérifier la version de Java installée grâce à la commande ci-dessous.
``` shell
java -version
```
Le résultat doit être semblable à celui ci-dessous
```
openjdk version "1.8.0_275"
OpenJDK Runtime Environment (build 1.8.0_275-8u275-b01-0ubuntu1~18.04-b01)
OpenJDK 64-Bit Server VM (build 25.275-b01, mixed mode)
```
## Cassandra - installation

Télécharger la dernière version de Cassandra sur le site https://cassandra.apache.org/ et décompresser l'archive obtenue. Dans notre cas, nous utiliserons la version `3.11.9` (Janvier 2021).
``` shell
wget https://mirrors.ircam.fr/pub/apache/cassandra/3.11.9/apache-cassandra-3.11.9-bin.tar.gz
tar -xzf apache-cassandra-3.11.9-bin.tar.gz
rm apache-cassandra-3.11.9-bin.tar.gz
```

Exécuter le daemon Cassandra avec la commande :
``` shell
./apache-cassandra-3.11.9/bin/cassandra
```
Dans un autre terminal, sur la même instance EC2, vérifier l'état du noeud ainsi créé. La commande :
``` shell
./apache-cassandra-3.11.9/bin/nodetool status
```
doit retourner quelque chose similaire à 
```
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address    Load       Tokens       Owns (effective)  Host ID                               Rack
UN  127.0.0.1  70.7 KiB   256          100.0%            c180d502-ed1a-4b30-880d-84e7cbde8be3  rack1
```

**Bravo ! Le sumbole `UN` signifie que cassandra est bien actif sur ce noeud** (UN = Up and Normal). Il reste à configurer la communication entre les noeuds.


## Cassandra - configuration de la communication

Afin de garantir la bonne communication entre les noeuds, il peut être nécessaire de suivre un certain ordre dans le lancement des daemons cassandra :
- configurer un premier noeud (noeud 1) (installation de cassandra + configuration de la communication)
- exécuter le daemon cassandra sur ce noeud 1, et laisser le daemon actif
- installer cassandra et configurer la communication sur le noeud 2
- exécuter le daemon cassandra sur le noeud 2
- vérifier la bonne communication entre les noeuds 1 et 2

Pour configurer la communication entre les différents noeuds du cluster, paramétrons Cassandra sur les isntances. Pour cela, 2 fichiers nous seront utiles :
* `cassandra.yaml`
* `cassandra-rackdc.properties`
qui se situent dans le répertoire `/apache-cassandra-3.11.9/conf/`

Pour modifier les fichiers avec les droits admin, utiliser la commande ci-dessous
``` shell
sudo nano <nom_fichier>
```
_Note : Enregistrer le fichier avec `ctrl+x` puis confirmer avec `y` puis `Entrée`_

#### 1. cassandra.yaml
__Seeds__

Lorsqu'un noeud Cassandra rejoint le "ring", il utilise les données d'un noeud présent afin de s'instancier sur la base de celui-ci. 
Le premier noeud aura pour "seed" sa propre adresse IP (privée).  
Les noeuds suivants auront dans un 1er temps l'adresse IP (privée) de ce premier noeud.  
Puis une fois les premières communications établies, nous pourrons modifier cette liste d'adresses afin d'y ajouter les IP (privées) de plusieurs noeuds du cluster.
Exemple de configuration pour le noeud 1 :
```
seed_provider:
    - class_name: org.apache.cassandra.locator.SimpleSeedProvider
      parameters:
          # seeds is actually a comma-delimited list of addresses.
          # Ex: "<ip1>,<ip2>,<ip3>"
          - seeds: "172.31.51.42,34.207.61.42,35.175.104.58"
```

__Listen address__

Paramétrer le port d'écoute de l'instance, en passant l'IP privée de l'instance de celle-ci au champ `listen_address`
```
listen_address: 172.31.51.42
```

__Snitch__

Etant donné que nous travaillons sur AWS et plus précisemment, sur une unique zone (obligatoire pour les comptes AWS Educate), nous pouvons spécifier le paramètre de `Snitch`
```
endpoint_snitch: Ec2Snitch
```

__rpc__

Changer l'adresse rpc à nouveau avec l'IP privée de l'instance et le rpc_port (si besoin)
```
rpc_address: 172.31.51.42
rpc_port: 9160
```

#### 2. cassandra-rackdc.properties

Commenter toutes les lignes du fichiers pour que les données de snitch considérées soient celles paramétrées dans le fichier précédent.
&nbsp;
**Répéter les étapes précédentes pour chaque noeud du cluster**
&nbsp;
Sur chacun des noeuds du cluster, exécuter le daemon cassandra
``` shell
./apache-cassandra-3.11.9/bin/cassandra
```


Vérifier l'état du cluster, pour s'assurer que les noeuds communiquent bien entre eux.
La commande 
``` shell
./apache-cassandra-3.11.9/bin/nodetool status
```
doit retourner quelque chose du style
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
**Utiles**
Si besoin, checker la log de lancement de cassandra
``` shell
tail /var/log/cassandra/system.log
```
Killer un processus cassandra sur un noeud
``` shell
pkill -f 'java.*cassandra'
```
Après l'arrêt du processus cassandra, l'état du noeud en question doit apparaître `DN` lors de l'exécution de la commande `nodetool status`
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