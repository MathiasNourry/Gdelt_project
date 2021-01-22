## Configuration AWS

### 0. Premiers pas avec AWS : bucket S3 et cluster EMR
### 1. Créer un ETL Spark-Scala sur EMR
### 2. Créer un cluster d'instances EC2 sur AWS

Afin de requêter efficacement les données preprocessées par les ETL, nous allons les injecter dans une base de données non relationnelle. Nous allons créer celle-ci **from scratch**, sur la base d'un ensemble d'**instances EC2**.

Connectez vous à AWS et dirigez vous sur la page du service `EC2`, puis sélectionez le menu `Instances` et cliquez sur le bouton "Lancer des instances".
### 3. Installer et configurer Apache Cassandra sur EC2
### 4. Installer et configurer Zeppelin sur EC2
### 5. Transférer des données de AWS S3 vers Apache Cassandra
### 6. Requêter le ring Cassandra

### Configuration cluster EC2

Dans AWS, cliquer sur EC2, puis "instances", puis sur le bouton "lancer des instances".
Valider les choix dans les différentes étapes :
1) choisir un système d'exploitation pour les instances (je sélectionne ubuntu 18 : nécessaire pour l'install de Cassandra)
2) choisir la capacité des machines (t2.micro pour tester __edit__ il semble que zeppelin connecté à cassandra ne fonctionne que sur les machines Medium ...)
3) choisir le nombre d'instances (je choisis 3 pour tester et je ne change aucun autre paramètre)
4) choix du stockage : 8Go (par défaut)
5) pas d'ajout de balises
6) Ajouter l'ouverture des ports TCP 7000 et 9042 à la création du groupe de sécurité (depuis n'importe où) __primordial pour que les noeuds cassandra puisque communiquer__

Cliquer sur le bouton de lancement du cluster.

Générer une nouvelle paire de clé pour le cluster, et la télécharger en cliquant sur le bouton dédié avant de passer à la suite en cliquant sur le bouton "Lancer les instances".

Protéger la paire de clé d'éventuelles modifications en lançant la commande 
``` shell
chmod 400 \<name-of-key-pair\>.pem 
```
__Connexion aux instances__

``` shell
ssh -i \<chemin-vers-keyPair.pem\> ubuntu@\<DNS-address-of-master\>
```

### Install Apache Cassandra

#### Install Java

À noter : Java 8 est requis

Update package index
``` shell
sudo apt update
``` 
Install openjdk package
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
Le résultat devrait être
```
openjdk version "1.8.0_275"
OpenJDK Runtime Environment (build 1.8.0_275-8u275-b01-0ubuntu1~18.04-b01)
OpenJDK 64-Bit Server VM (build 25.275-b01, mixed mode)
```
#### Install Cassandra

Télécharger la dernière version de Cassandra sur le site https://cassandra.apache.org/ et décompresser l'archive obtenue.
``` shell
wget https://mirrors.ircam.fr/pub/apache/cassandra/3.11.9/apache-cassandra-3.11.9-bin.tar.gz
tar -xzf apache-cassandra-3.11.9-bin.tar.gz
rm apache-cassandra-3.11.9-bin.tar.gz
```

Vérifier le bon fonctionnement de cassandra en lançant les commandes 
``` shell
./apache-cassandra-3.11.9/bin/cassandra
```
dans un autre terminal, sur la même instance EC2
``` shell
./apache-cassandra-3.11.9/bin/nodetool status
```
qui doit retourner quelque chose similaire à 
```
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address    Load       Tokens       Owns (effective)  Host ID                               Rack
UN  127.0.0.1  70.7 KiB   256          100.0%            c180d502-ed1a-4b30-880d-84e7cbde8be3  rack1
```

À ce stade, cassandra est installé sur le noeud du cluster. Il est nécessaire de répéter les différentes étapes ci-dessus sur les autres noeuds du cluster.


#### Config Cassandra


172.31.95.67,172.31.83.243,172.31.86.222,172.31.83.28,172.31.83.237

Il semble nécessaire de lancer un 1er noeud (avec la seed affectée à lui même), avant de lancer les autres noeuds (dont les seeds sont fixées à l'IP du 1er noeud).

Nous allons modifier les fichiers :
* `cassandra.yaml`
* `cassandra-rackdc.properties`
qui se situent dans le répertoire `/apache-cassandra-3.11.9/conf/`

Modifier les fichiers avec la commande
```
sudo nano <nom_fichier>
```

##### cassandra.yaml
__Seeds__

Donner les adresses IP privées des instances du cluster.
```
seed_provider:
    - class_name: org.apache.cassandra.locator.SimpleSeedProvider
      parameters:
          # seeds is actually a comma-delimited list of addresses.
          # Ex: "<ip1>,<ip2>,<ip3>"
          - seeds: "172.31.51.42,34.207.61.42,35.175.104.58"
```

__Listen address__

Paramétrer la listen_address avec l'IP privée de l'instance.
```
listen_address: 172.31.51.42
```

__Snitch__

Etant donné que nous travaillons sur AWS et plus précisemment, sur une unique zone (obligatoire pour les comptes AWS Educate), nous pouvons passer le paramètre suivant
```
endpoint_snitch: Ec2Snitch
```

__rpc__

Changer l'adresse rpc avec l'IP privée de l'instance et le rpc_port (si besoin)
```
rpc_address: 172.31.51.42
rpc_port: 9160
```

##### cassandra-rackdc.properties

Commenter toutes les lignes du fichiers pour que les données de snitch considérées soient celles paramétrées dans le fichier précédent.

&nbsp;
Répéter les étapes précédentes pour chaque noeud du cluster.
&nbsp;

Sur le noeud dont l'IP est la seed passée dans les fichiers de config, exécuter le daemon cassandra
``` shell
./apache-cassandra-3.11.9/bin/cassandra
```
Ensuite, faire de même sur les autres noeuds du cluster

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


### Requêter sur le ring Cassandra

Dans les étapes qui suivent, Zeppelin est installée sur l'une des instances contenant cassandra, et le cluster Cassandra contient 3 noeuds. Zeppelin utilise la version native de spark (disponible depuis les packages zeppelin), et pas un cluster spark déployé from scratch.

_[Edit] : les configurations du notebook zeppelin peuvent être faite sur une version locale (plutôt que sur une instance EC2)._ 

#### Zeppelin

Pour effectuer des requêtes sur le ring cassandra, on va utiliser un notebook Zeppelin configuré avec un connecteur spark-cassandra.

__Install__

Pour installer Zeppelin, on télécharge les paquets depuis l'adresse suivante.

``` shell
wget https://downloads.apache.org/zeppelin/zeppelin-0.8.2/zeppelin-0.8.2-bin-all.tgz
tar xzf zeppelin-0.8.2-bin-all.tgz
rm zeppelin-0.8.2-bin-all.tgz
```

__Launch__
Démarrer le daemon Zeppelin sur l'instance EC2
``` shell
cd zeppelin-0.8.2-bin-all/bin/
./zeppelin-daemon.sh start
```

__Connect__
Se connecter en ssh à l'instance EC2, en redirigeant le port utilisé par zeppelin pour y avoir accès localement (attention, le daemon zeppelin doit être démarré sur l'instance EC2). Le port par défaut de zeppelin est `8080`, nous allons le rediriger sur le port local `8891`.
``` shell
ssh -L 8891:127.0.0.1:8080 -i <keyPair.pem> ubuntu@ec2-18-232-186-153.compute-1.amazonaws.com
```

Dans le navigateur local, se connecter à l'adresser `localhost:8891`.


#### Connect Zeppelin to Cassandra

_**Note** : Dans le cas d'une connexion depuis un poste local sur le ring cassandra (stocké sur un cluster EC2), il faudra remplacer les IP privées des noeuds du cluster, par les IP **publiques**_.

Pré-requis

- Installer java 8

Optionnel
- Installer cassandra

Il s'agit de modifier les configurations du `Interpreter menu` de Zeppelin (cliquer sur la roue de configuration en haut à droit de la page du notebook zeppelin) afin de faire le lien entre le cluster cassandra et le notebook utilisé.

__Spark interpreter__

Sélectionner l'interpréteur spark grâce à la barre de recherche du menu interpreter, et effectuer les modifications suivantes :
1) Ajouter un pointeur vers la liste des adresses IP privées du cluster cassandra en créant une nouvelle ligne contenant
  ```
  spark.cassandra.connection.host           <IP cassandra>
  ```
2) Ajouter le connecteur Spark-Cassandra comme dépendance dans l'interpréteur. Les chemins peuvent être modifiés selon le système d'exploitation utilisé (voir ci-dessous pour le répertoire où copier les `.jar`). Dans tous les cas, inscrire le chemin **complet**.
![dependances](dependances.png)

Si les dépendances ne sont pas disponibles, télécharger les jar aux adresses

* https://mvnrepository.com/artifact/com.datastax.spark/spark-cassandra-connector_2.11/2.0.12
* https://mvnrepository.com/artifact/com.twitter/jsr166e/1.1.0

ou directement par commande sur les machines du cluster (le répertoire `interpreter` est situé dans le dossier `zeppelin` téléchargé via fichiers binaires).

``` shell
cd ~/zeppelin-0.8.2-bin-all/interpreter/spark/dep
wget https://repo1.maven.org/maven2/com/datastax/spark/spark-cassandra-connector_2.11/2.0.12/spark-cassandra-connector_2.11-2.0.12.jar
wget https://repo1.maven.org/maven2/com/twitter/jsr166e/1.1.0/jsr166e-1.1.0.jar
```
_Note : cliquer sur le bouton "+" à droite des ajouts afin que l'interpréteur prenne bien les modifications en compte._

__Cassandra interpreter__

Sélectionner l'interpréteur cassandra grâce à la barre de recherche du menu interpreter et ajouter un pointeur vers la liste des adresses IP privées du cluster cassandra en créant la ligne
  ```
  cassandra.hosts           <IP cassandra>
  ```

### Data Analysis

Maintenant que Zeppelin est connecté à Cassandra, il est possible de charger des données sur le cluster, et de les requêter pour répondre aux problématiques du projet GDELT

Pour faciliter le traitement par l'utilisateur, on extrait la table stockée sur Cassandra au sein du Zeppelin notebook, et on transforme les données pour les lire via `spark.sql`

__Exemple de requête__

![Data analysis](data_analysis.png)

__Etat du cluster Cassandra__

Une fois les données chargées avec un `replication_factor` de 3, on peut vérifier l'état du keyspace créé sur le cluster Cassandra
``` 
ubuntu@ip-172-31-71-95:~/apache-cassandra-3.11.9/bin$ ./nodetool status gdelt_project
Datacenter: us-east
===================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address        Load       Tokens       Owns (effective)  Host ID                               Rack
UN  172.31.71.95   12.48 MiB  256          100.0%            ae0cd4e5-7f86-4502-a527-e728c32af8bc  1f
UN  172.31.75.249  12.46 MiB  256          100.0%            c97a67f3-3be5-491c-ad30-30cc7ae00776  1f
UN  172.31.79.24   12.46 MiB  256          100.0%            05a936ee-e9e8-414a-b7c5-42a0d87d1475  1f
```
__Résilience__

Après l'arrêt forcé d'un noeud, il est toujours possible de requêter depuis le Zeppelin notebook grâce à la réplication des données sur le ring Cassandra.





## Access S3 from EC2

Pour autoriser l'accès à un bucket S3 depuis un instance EC2, nous allons procéder aux étapes suivantes
1. Configuer les rôles et stratégies des buckets et instances
2. Configurer `aws cli`

### 1. Rôles et stratégies des buckets et instances

**Créer un profil d'instance IAM qui autorise l'accès à S3**

Aller sur la page du service EC2 et sélectionner l'instance qui aura accès à S3, afin d'obtenir le rôle IAM lui étant associé ; ici : `EMR_EC2_DefaultRole`

![](EC2_role.png)

Sur la page du service IAM, cliquer sur l'onglet "Rôles" puis cliquer sur le rôle de l'instance EC2 souhaitée (voir ci-dessus). Dans le menu "autorisation", attacher une nouvelle stratégie permettant l'accès à S3 : `AmazonS3FullAccess`

![](IAM_role.png)

**Valider les autorisations sur le bucket S3**

Sur la page du service S3, sélectionner le bucket à connecter à l'instance EC2, puis dans le menu "Autorisations", **désactiver** le blocage de l'accès public

![](S3_role.png)

Si besoin, retirer les instructions `Effect:Deny` de la stratégie de compartiment

Les éléments AWS sont configurés, a présent, connectons nous à l'instance EC2 afin de télécharger les données depuis S3.

### 2. Configurer `aws cli`

Données du projet GDELT :

- groupe de sécurité : **sg-0384c4f143d0535dd (launch-wizard-3-09012021)**
- bucket S3 utilisé pour le projet (configuré via lien ci-dessus): **simondelarue-s3-09012021**
- rôle IAM associé au master du cluster EC2 : **EMR_EC2_DefaultRole** (configuré avec la règle d'accès S3FullAccess)
- keyPair associée au cluster EC2 : **keyPair-09012021.pem**

**Définir les credentials du user**

Les étapes ci-dessous sont à effectuer depuis le terminal de l'instance EC2 à connecter au bucket S3.

Modifier les credentials de la session dans les configurations de `aws cli`.

a. Récupérer les credentials de la session
Retourner à la page de connexion "Vocareum" et cliquer sur `account details`
![](vocareum.png)
Puis copier **la totalité du contenu des credentials** (y compris "[default]" et le dernier "=")
![](credentials.png)
b. Ouvrir le fichier de configuration de `aws cli` avec la commande ci-dessous, et y coller les éléments récupérés à l'étape précédente
``` shell
sudo nano ~/.aws/credentials
```
Si le fichier `credentials` n'existe pas (nouvelle instance EC2 par exemple), il est nécessaire dans un premier temps d'installer `aws cli` avec la commande
``` shell
sudo apt-get install awscli
```


Ca y est ! `aws cli` est configuré. Nous pouvons à présent interagir depuis l'instance EC2 avec S3.

**Download fichiers depuis S3**

Download les fichiers depuis AWS S3 en lançant la commande
``` shell
aws s3 cp s3://<S3_bucket_name>/<S3_filename> <local_filename>
```
_Note : Cette commande peut être lancée depuis un zeppelin notebook, dans l'interpréteur shell, avec le mot clé `%sh` en début de cellule_
![](shell_cmd.png)

Pour download/upload tout le contenu d'un répertoire depuis/vers S3, utiliser l'option `--recursive` dans la commande `aws s3 cp`.

_Source : 
https://aws.amazon.com/fr/premiumsupport/knowledge-center/ec2-instance-access-s3-bucket/_
&nbsp;
&nbsp;
&nbsp;

-- Ci-dessous : EN COURS


### Spark

Download 2.4.7 version (comptabile avec zeppelin 0.8.2)
``` shell
wget https://apache.mirrors.benatherton.com/spark/spark-2.4.7/spark-2.4.7-bin-hadoop2.7.tgz
tar -xzf spark-2.4.7-bin-hadoop2.7.tgz
rm spark-2.4.7-bin-hadoop2.7.tgz
```

Modifier les variables d'environnement en ajoutant la ligne suivante dans le fichier `~/.bashrc` 
```
export SPARK_HOME=/home/ubuntu/spark-2.4.7-bin-hadoop2.7
```
Puis dans le terminal, lancer la commande
``` shell
source ~/.bashrc
```


### Master node config

Copier les 2 fichiers de configuration situés dans `/spark-2.4.7-bin-hadoop2.7/conf/` 
``` shell
cd /home/ubuntu/spark-2.4.7-bin-hadoop2.7/conf
cp spark-env.sh.template spark-env.sh 
cp spark-defaults.conf.template spark-defaults.conf 
```

A présent, modifier les deux fichiers :
* `spark-env.sh`
* `spark-default.conf`


__spark-env.sh__

Ajouter les lignes suivantes dans le fichier `spark-env.sh`
```
export SPARK_LOCAL_IP=<PRIVATE_DNS_this_NODE>
export SPARK_MASTER_HOST=<PRIVATE_DNS_this_NODE>
```

__spark-default.conf__

Ajouter les lignes suivantes dans le fichier `spark-default.conf`
```
spark.master                        spark://PRIVATE_DNS_MASTER1:7077,PRIVATE_DNS_MASTER2:7077
spark.jars.packages                 datastax:spark-cassandra-connector:2.0.0-s_2.11
spark.cassandra.connection.host     <PRIVATE_DNS_Slaves> (separated by ',')
```

### Worker node config

Copier les 2 fichiers de configuration situés dans `/spark-2.4.7-bin-hadoop2.7/conf/` 
``` shell
cd /home/ubuntu/spark-2.4.7-bin-hadoop2.7/conf
cp spark-env.sh.template spark-env.sh 
cp spark-defaults.conf.template spark-defaults.conf 
```

Les deux fichiers suivants sont à modifier :
* `spark-env.sh`
* `spark-default.conf`

__spark-env.sh__

Ajouter les lignes suivantes dans le fichier `spark-env.sh`
```
export SPARK_LOCAL_IP=<PRIVATE_DNS_this_NODE>
export SPARK_MASTER_HOST=<PRIVATE_DNS_MASTER1,PRIVATE_DNS_MASTER1>
```

__spark-default.conf__

Ajouter les lignes suivantes dans le fichier `spark-default.conf`
```
spark.master                        spark://PRIVATE_DNS_MASTER1:7077,PRIVATE_DNS_MASTER2:7077
spark.jars.packages                 datastax:spark-cassandra-connector:2.4.0-s_2.11
spark.cassandra.connection.host     <PRIVATE_DNS_Slaves> (separated by ',')
```

__Spark-Cassandra connectors__

Version du connecteur compatible avec Spark 2.4 et Cassandra 3.x
``` shell
wget https://dl.bintray.com/spark-packages/maven/datastax/spark-cassandra-connector/2.4.0-s_2.11/spark-cassandra-connector-2.4.0-s_2.11.jar
wget https://repo1.maven.org/maven2/com/twitter/jsr166e/1.1.0/jsr166e-1.1.0.jar
```



## Access cassandra from EMR

**Old method**
source : http://www.doanduyhai.com/blog/?p=2325

install git
``` shell
sudo yum install git-core
```
download spark-cassandra connector repo
``` shell
git clone https://github.com/datastax/spark-cassandra-connector/
```
install sbt (to create jar file from spark-cassandra connector repo)
source : https://www.scala-sbt.org/1.x/docs/Installing-sbt-on-Linux.html
``` shell
curl https://bintray.com/sbt/rpm/rpm | sudo tee /etc/yum.repos.d/bintray-sbt-rpm.repo
sudo yum install sbt
```
create jar file from spark-cassandra connector repo. 
_Executer la commande à l'endroit où a été téléchargé le repo spark-cassandra connector_
``` shell
sbt assembly
```

Modifier le fichier conf de spark dans EMR
``` shell
cd /etc/spark/conf
sudo nano spark-env.sh
```
ajouter la ligne
```
export SPARK_SUBMIT_OPTIONS=' --packages com.datastax.spark:spark-cassandra-connector_2.11:2.0.12'
```

Modifier le fichier conf de zeppelin dans EMR
``` shell
cd /etc/zeppelin/conf
sudo nano zeppelin-env.sh
```
ajouter/compléter la ligne avec les packages de connecteur spark-cassandra
```
export SPARK_SUBMIT_OPTIONS=' --packages com.datastax.spark:spark-cassandra-connector_2.11:2.0.12'
```

**New method**

Modifier les dépendances zeppelin dans EMR en y ajoutant les fichiers `.jar` de dépandances
``` shell
cd /usr/lib/zeppelin/interpreter/spark/dep
sudo wget https://repo1.maven.org/maven2/com/twitter/jsr166e/1.1.0/jsr166e-1.1.0.jar
sudo wget https://repo1.maven.org/maven2/com/datastax/spark/spark-cassandra-connector_2.11/2.0.12/spark-cassandra-connector_2.11-2.0.12.jar
```

Ajouter les connecteurs Spark-Cassandra comme dépendance dans l'interpréteur
![dependances](dependances.png)






