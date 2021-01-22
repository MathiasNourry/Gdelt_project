<p align="center">
  <img src="https://github.com/MathiasNourry/Gdelt_project/blob/main/img/gdelt_global.png" width="700" />
</p>

<center>
  <h1>Gdelt - NoSQL Big data implementation from scratch</h1>
</center>

## Evolution de la pandémie COVID19 via son impact média

_Contributeurs : Vincent, Bardonnet, Alexandre Bréboin, Simon Delarue, Mathias Nourry, Valentin Pannier_

> _" The Global Database of Events, Language, and Tone (GDELT) monitors the world’s broadcast, print, and web news from nearly every corner of every country in over 100 languages and identifies the people, locations, organizations, themes, sources, emotions, counts, quotes, images and events driving our global society every second of every day, creating a free open platform for computing on the entire world_

**Objectif**

|L'objectif de ce projet est d'analyser l'évolution de la pandémie COVID, en fonction de la couverture médiatique associée. Pour cela, nous utilisons le jeu de données du **[projet Gdelt](https://blog.gdeltproject.org/gdelt-2-0-our-global-world-in-realtime/_)**.|
| --- |

### 1. Données

Le Gdelt Project vise à réunir les articles de presse du monde entier sous un même endroit. Au-delà d'un simple travail de récolte de données, ces dernières sont analysés pour produire des informations au sujet des thèmes, des sources, des lieux ou encore du ton de l'article.
In fine, 3 sources de données sont mises à disposition :
- export : données relatives à une publication
- mentions : données relatives aux mentions de chaque publication 
- gkg : données relatives aux lieux, acteurs et ton de chaque publication   

Ces 3 sources sont générées par intervalle de 15 minutes et sont indexées selon deux fichiers principaux :
- [Masterfile : English](http://data.gdeltproject.org/gdeltv2/masterfilelist.txt) : relatif aux publications écrites en anglais
- [Masterfile : Translation](http://data.gdeltproject.org/gdeltv2/masterfilelist-translation.txt) : relatif aux publications dans leur langue originale (hors anglais)

L'étude d'un an de données correspond à environ **500Go** à traiter. Les choix d'architecture pour l'analyse ont donc un impact majeur sur les performances finales du modèle !


### 2. Architecture

Pour répondre aux besoins de l'analyse, nous avons mis en place une architecture basée déployée sur **Amazon Web Services** (AWS), au travers de technologies distribuées comme **Spark** et d'une base de données NoSQL, **Cassandra**.

<p align="center">
  <img src="https://github.com/MathiasNourry/Gdelt_project/blob/main/img/modele.png" width="700" />
</p>

Le choix de la technologie de base de données sélectionné s'est fait sur la base du triangle CAP :
- ***C***onstistency
- ***A***vailability
- ***P***artition tolerance

<p align="center">
  <img src="https://github.com/MathiasNourry/Gdelt_project/blob/main/img/CAP.png" width="500" />
</p>

Le theorème CAP suggère qu'il faille faire un choix entre les trois caractéristiques de notre base.
Dans notre cas d'étude, nous avons décidé de ne pas sélectionner la caractéristique ***C***onsistency qui assure normalement de toujours obtenir des données à jour au moment d'une lecture de la base. En effet, les données stockées correspondront à l'année 2020 et représentent donc une image des données du Gdelt pour cette année. Les données stockées ne sont donc pas amenées à être modifiées.   
Au final on souhaite donc une technologie permettant : 
- une fault tolerance (accessibilité aux données malgré que certains noeuds tombent) = ***P***artition tolerance 
- que le temps de requêtage soit relativement faible = ***A***vaibility


#### EMR - ETLs

Un cluster de 7 machines m4.xlarge à été mis en place, chaque machine m4.xlarge possède 4 CPUs, ce qui totalise 28 CPUs. Notre compte AWS educate permettant d'utiliser un maximum de 32 CPUs en simultané. 
A partir de notre cluster EMR, nous avons lancé un notebook Zeppelin. Sur ce Zeppelin, nous avons devellopé L'ETL 1 qui permet de télécharger les données brutes Gdelt (masterfile & masterfile-translation) sous formes de .zip dans le dossier raw-data sur un bucket S3. 
Dans un second temps, nous avons devellopé le second ETL qui permet de récupérer les données brutes et de les transformer en DataFrame permettant de répondre aux problématique du projet. Ces données sont sauvegardées en .parquet dans le dossier cleaned-data sur le bucket S3

La charge de téléchargement à été répartie entre 3 personnes, qui ont lancé chacune un cluster de machine, executés les 2 ETLs sur 4 mois de données Gdelt.

#### EC2 - Cassandra

Afin de stocker les données préprocessées dans un format exploitable facilement par l'utilisateur, tout en respectant notre souhait d'architecture résiliante, nous avons installé Cassandra **from scratch** sur un ensemble d'instances EC2.

_Note : Pour configurer cette architecture chez vous, suivez ce [tutoriel](lien tuto)_

**Configurations**
* **EC2 instances** : M4Large 
* **Replication factor** = 3
* **Snitch** : Ec2Snitch
Ce paramètre est optimal pour une utilisation du cluster au sein d'une même région (ce qui est une contrainte imposée par l'utilisation d'un compte AWS étudiant).
* **Read/Write consistency** = ONE/LOCAL_QUORUM
Ces choix nous permettent en effet d'offrir à l'utilisateur la possibilité de requêter les données si certains noeuds sont _down_, tout en assurant une consistance raisonnable au moment du chargement des données.
* **Load** ~1Go de données par noeud

<p align="center">
  <img src="https://github.com/MathiasNourry/Gdelt_project/blob/main/img/nodetool_status.png" width="700" />
</p>

Les données **export** et **mentions** pour une année entière sont injectées dans 2 tables distinctes, permettant à l'utilisateur un requêtage simple et efficace.

### 3. Zeppelin - Analyse des données

Les données sont à présent sur le ring Cassandra, et l'utilisateur peut y accéder en utilisant un **Notebook Zeppelin**, configuré spécifiquement pour dialoguer avec Cassandra (via l'installation d'un connecteur spark-cassandra).

Il existe deux façons distinctes d'obtenir les données :
* via `CQL` le langage de requête natif de Cassandra. Ce langage impose cependant des contraintes fortes sur la manipulation des données (notamment sur les agrégations et jointures)
* via `Spark-SQL`, en important les tables depuis Cassandra et en les injectant dans des _vues_ destinées à faciliter l'analyse. C'est cette méthode que nous retenons (exemple ci-dessous)
``` scala
val mentions_from_cass = spark.read.cassandraFormat("mentions", "gdelt_project").load()
mentions_from_cass.createOrReplaceTempView("mentions")
```

<p align="center">
  <img src="https://github.com/MathiasNourry/Gdelt_project/blob/main/img/export_table.png" width="500" />
</p>

**Nombre d'événements médiatiques relatifs au COVID, par date et pays**

Il peut être intéressant d'observer l'évolution du nombre d'articles relatifs au COVID sur l'année 2020, en détaillant par pays d'origine de l'article.  
On note qu'avant le mois de mars, la couverture médiatique était relativement basse, et principalement concentrée sur la Chine. Après le début de la "première vague", tous les pays - et majortairement les US - ont participé à la production d'articles.

``` sql
select country
    ,days
    ,count(globaleventid) as nb_event 
from export
where country <> "null"
group by
    country
    ,days
order by nb_event desc
```

<p align="center">
  <img src="https://github.com/MathiasNourry/Gdelt_project/blob/main/img/COVID_events.png" width="1000" />
</p>

**Nombre d'événements médiatiques relatifs au COVID, par pays et par langue**

En observant le nombre d'articles relatifs au COVID par pays et par langue, on peut noter les différences de proportion dans les langues utilisées pour la production d'articles ; ci-dessous des exemples pour la France et l'Italie.

``` sql
select country
    ,days
    ,language
    ,count(globaleventid) as nb_event 
from export
where country="${Country=France,Belgium|France|Canada|China|Germany|Italy|India|Mexico|Spain|United States|United Kingdom}"
    and days <= 20200531
    and days >= 20200430
group by
    country
    ,days
    ,language
order by nb_event desc
```
<p align="center">
  <img src="https://github.com/MathiasNourry/Gdelt_project/blob/main/img/COVID_events_France.png" width="1000" />
</p>

<p align="center">
  <img src="https://github.com/MathiasNourry/Gdelt_project/blob/main/img/COVID_events_India.png" width="1000" />
</p>

**Nombre de mentions des évenements, par pays**

Finalement, au delà du nombre d'articles relatifs au COVID publiés dans une région, nous pouvons regarder les événements qui ont suscité le plus de mentions dans d'autres sources d'information.

``` sql
select export.url, count(mentions.mention) as nb_mentions
from export as export
left join mentions as mentions
    on export.url = mentions.mention
where export.country = "${Country=United States,Belgium|France|Canada|China|Germany|Italy|India|Mexico|Spain|United States|United Kingdom}"
    and days = "20201208"
group by export.url
order by nb_mentions desc
limit 15
```

<p align="center">
  <img src="https://github.com/MathiasNourry/Gdelt_project/blob/main/img/COVID_mentions.png" width="1000" />
</p>

### 4. Limites et contraintes du modèle


    a) Contraintes liées à Cassandra

        Cassandra ne supporte ni les jointures ni les groupby car les jointures sur des machines différentes dans un système distribué ne sont pas performantes. Pour résoudre cela, nous avons dû passer par Spark SQL ce qui rajoute des étapes supplémentaires.   

    b) Cases vides

        Certains articles n’ont pas de Pays associés. Lors d’une requête par Pays, il y aura probablement une sous-évaluation du nombre d’articles

    c) Mise en forme des données

        Certains articles sont référencés avec un numéro d’article, sachant que notre filtre se fait sur l'URL, ce dernier est potentiellement un article sur le Covid non détecté.

    d) Limitations du compte "Educate"

        Nous sommes limitées dans le nombre de machines réservées à un même cluster ce qui augmente le temps de téléchargement => 32 CPUs environ

        Nécessité de répartir le téléchargement de l'ensemble des données entre plusieurs personnes. Chaque compte ayant 100 crédits et l'ensemble du projet coûtant plus que cela. Nous n'aurions pu tout effectuer sur un seul compte.

        Du fait que nous ayons récupéré toutes les données sur une année, il aurait fallut pouvoir allouer des machines plus puissantes que m4.large à l'EC2 afin d'avoir un requêtage plus efficace.



