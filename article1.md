# Surveillance applicative avec Prometheus

## Introduction

Prometheus est un outil de surveillance applicative très connu dans le monde de l'observabilité.

<br>

![Prometheus Schéma global](./img/prometheus.png)

<br>

Prometheus fonctionne en mode `Pull`, il collecte les métriques à intervalle régulier auprès des applications à surveiller.<br>
Ces métriques sont ensuite enregistrées dans une base de données temporelles.<br>
Le langage **PromQL** permettra de requêter les données par le biais d'une API Rest.

On retrouve dans Prometheus :

- un outil pour collecter les métriques.
- une base de données temporelles (Time Series DataBase).
- un langage de requête (PromQL).
- une API Rest pour exécuter les requêtes et gérer la plateforme.
- une interface graphique pour visualiser les données.

Il est recommandé d'utiliser **Grafana** pour l'affichage des données. 
L'interface minimaliste de Prometheus sera utilisée uniquement dans la phase de développement pour tester des requêtes PromQL.

## Types de métriques

Une métrique est une mesure numérique d'un élement de l'application, par exemple la quantité de mémoire RAM utilisée. <br>
Elles se déclinent  principalement en quatre types.

**Compteur**

La valeur d'un compteur peut **uniquement être incrémentée**.

<u>Exemple</u> : Nombre de requêtes HTTP executées par un serveur web.

<br>

![Compteur](./img/compteur.png)

<br>

**Jauge**

La valeur d'une jauge peut **augmenter ou diminuer**.

<u>Exemple</u> : Température d'un processeur en degré celcius.

<br>

![Jauge](./img/jauge.png)

<br>

**Histogramme**

Un histogramme compte le nombre de données selon certaines catégories prédéfinies.

<u>Exemple</u> : Durée d'exécution des requêtes HTTP.

<br>

![Jauge](./img/histogramme.png)

<br>

10 requêtes ont un temps d'exécution compris entre 0,5 et 1 seconde.

Le nombre de requêtes de chaque catégorie augmentera au cours du temps. Comme pour un compteur, cette valeur ne pourra pas diminuer.

**Résumé**

Un résumé permet de calculer la valeur de certains quantiles.

<u>Exemple</u> : Durée d'exécution des requêtes HTTP.

<br>

![Résumé](./img/resume.png)

<br>

Au vu du dernier résultat, on déduit que le temps d'exécution d'une requête est inférieur ou égal à 650ms pour au moins 99% d'entre elles. <br>
Le temps d'exécution médian est égal à 189ms.

Un quantile de rang q (0 <= q <= 1) d'une série ordonnée d'éléments permet d'évaluer la valeur d'un élément à un rang donné.<br>

Par exemple, le quantile 0,5 également appelé médiane permet de séparer une série en deux parties. Sa valeur, contrairement à la moyenne, ne sera pas affectée par des valeurs disproportionnées présentes dans le jeu de données.

<br>

![Médiane](./img/mediane.png)


## Format des métriques

Les applications doivent exposer leurs métriques par le biais d'un service HTTP de type GET.<br> 
Ce service retournera l'intégralité des métriques dans un format texte compréhensible par Prometheus.

<b>Compteur</b>

![Compteur format](./img/compteur_format.png)

Chaque ligne (excepté les en-têtes) définit une nouvelle série temporelle qui est identifiée par son **nom** et éventuellement des **libellés** de type clé-valeur. Une métrique correspond donc à un ensemble de séries temporelles.

Par convention, le nom d'un compteur se terminera toujours par `_total`.

<b>Jauge</b>

![Jauge format](./img/jauge_format.png)

La première jauge `process_resident_memory_size` indique la taille mémoire occupée par le processus. Cette valeur pourra augmenter ou diminuer.

Une jauge peut également servir à stocker une constante, c'est le cas de `process_start_time_seconds` qui indique la date de démarrage du processus.

Une métrique doit obligatoirement avoir un nom. Elle peut en revanche ne pas avoir de libellés, c'est le cas pour ces deux jauges.   

<b>Histogramme</b>

![Histogramme format](./img/histogramme_format.png)

Un histogramme est composé de plusieurs séries temporelles. 

On a dans un premier temps les séries correspondantes aux catégories de l'histogramme. Leur nom se termine par `_bucket`. Chacune de ces séries contient obligatoirement le libellé `le` (lower or equal).<br>

La première catégorie nous indique que 309 requêtes ont eu une durée d'exécution inférieure ou égale 0,1s.<br>

Les valeurs présentent dans ces catgégores sont cummulatives, on en déduit que 2 requêtes ont eu une durée d'exécution comprise entre 0,1s et 0,2s.<br>
La derniere catégorie possède obligatoirement le libellé `le="+Inf"`.

On trouve enfin deux séries contenant la somme et le nombre des valeurs enregistrées. 311 requêtes ont été exécutées pour une durée totale de 1,55s. Ces données nous permet de calculer la moyenne.

Chaque service (identifié par le libellé `handler`) contiendra ce même ensemble de séries. J'ai affiché ici uniquement celles liées au service `/metrics`.

<b>Résumé</b>

![Résumé format](./img/resume_format.png)

Chaque série liée au calcul d'un quantile possède le libellé `quantile`. Prometheus est configuré par défaut pour récupérer les métriques toutes les 15 secondes, les valeurs des quantiles pour cet intervalle sont donc très proches de 15.

Comme pour l'histogramme, deux séries contiennent la somme et le nombre de valeurs enregistrées.

Les quantiles sont calculés par l'application cliente à chaque nouvel enregistrement.<br> 
Pour obtenir leur valeur exacte, il faudrait garder en mémoire l'intégralité du jeu de données, ce qui est inenvisageable pour une application ayant une durée de vie importante.<br> 

Un algorithme est utilisé pour éliminer certaines données au fil du temps. Les calculs des quantiles sont donc effectués en intégrant une marge d'erreur.

## Exposition des métriques

Pour obtenir les métriques d'une application, on peut :

- utiliser un exporter qui va venir s'intégrer à celle-ci.

- déclarer ses propres métriques en ajoutant du code à l'application.<br> Prometheus fournit des libraries dans plusieurs langages pour les créer et les exporter.

Les exporters permettent d'observer un système existant sans avoir à ajouter le moindre code.

* `Node exporter` expose les métriques d'un système Linux.
* `JMX exporter` expose les métriques d'une application Java en se basant sur les données disponibles via JMX.
* `PostgreSQL exporter` expose les métriques d'une base PostgreSQL

À noter que le serveur Prometheus expose des métriques, il peut donc s'observer lui-même.

![Exporteur](./img/exporteur.png)

## Collecte des métriques

Le fichier de configuration `prometheus.yml` permet de déclarer les applications à observer.

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: "node"
    static_configs:
      - targets: ["localhost:9100"]
```

`job_name` correspond au nom de l'application qui peut être déployée sur plusieurs instances présentes dans le champ `targets`.

Dans cet exemple, Prometheus collectera toutes les 15 secondes :
- ses propres métriques => `http://localhost:9090/metrics` 
- les métriques d'un serveur linux => `http://localhost:9100/metrics`

Pour une utilisation plus poussée, il serait plus adéquat d'utiliser la découverte de service pour récupérer la liste des instances dynamiquement. 

Prometheus enregistre les données retournées par le services http dans sa base sous forme de séries temporelles.<br>

```
# HELP prometheus_http_requests_total Counter of HTTP requests.
# TYPE prometheus_http_requests_total counter
prometheus_http_requests_total{code="200",handler="/-/ready"} 3
prometheus_http_requests_total{code="200",handler="/api/v1/query"} 1
prometheus_http_requests_total{code="200",handler="/graph"} 1
prometheus_http_requests_total{code="200",handler="/metrics"} 98
prometheus_http_requests_total{code="302",handler="/"} 1
```

Chaque ligne (hors commentaire) représente une nouvelle série temporelle, le compteur ci-dessus en aura donc 5. Prometheus ajoute automatiquement les libellés `node` et `instance` à chaque série.

Toutes les 15 secondes un nouvel enregistrement (représenté par une date en millisecondes et une valeur) sera sauvegardé pour chaque série.<br>

<u>Représentation d'une série temporelle :</u>

![Série temporelle](./img/serie_temporelle.png)

Concernant la récupération des métriques, il est primordial de définir un intervalle de temps assez court afin de ne pas passer à côté certaines valeurs importantes, ce qui est particulèrement vrai pour une jauge dont la valeur peut augmenter ou diminuer.

Par défaut, Prometheus conservera les données pendant 15 jours, cette valeur est configuration en utilisant l'option `--storage.tsdb.retention.time` au démarrage du serveur.

## Lecture des métriques

La lecture des données est essentielle pour évaluer la santé de nos applications. C'est la partie la plus complexe à appréhender, le langage PromQL (Prometheus Query Language) bien que très puissant reste assez difficile à maîtriser.

L'évaluation des requêtes PromQL s'effectuera via des appels http sur l'API REST exposée par Prometheus.

![API Prometheus](./img/prometheus_api.png)

### API Query

Le service `query` évalue une requête pour une date donnée.

Il existe principalement deux types de requêtes dans Prometheus.

<b>1) <u>requête sur un moment précis</u></b> (`instant_query`)

![Instant Query](./img/instant_query.png)

Le résultat contient une seule valeur par série. On choisit l'élément le plus récent par rapport à la date d'évaluation. 

<u>Appel REST</u>

**GET /api/query**
* query=prometheus_http_requests_total
* time=1682358304.676

*La date courante est utilisée si le paramètre time n'est pas renseigné.*

<pre>
{
  "status":"success",
  "data":{
    "resultType":"vector",
    "result":[
    {
      "metric":{
        "__name__":"prometheus_http_requests_total",
        "code":"200",
        "handler":"/-/ready",
        "instance":"localhost:9090",
        "job":"prometheus"
      },
      "value":[1682358304.676,"1"]
    },
    {
      "metric":{
        "__name__":"prometheus_http_requests_total",
        "code":"200",
        "handler":"/api/v1/query",
        "instance":"localhost:9090",
        "job":"prometheus"
      },
      "value":[1682358304.676,"2"]
    },
    ...
    ]
  }
}
</pre>

`value` est un tableau contenent seulement deux éléments, le premier correspondant à l'horadatage et le second à la valeur.

Le temps associé à chaque valeur sera égal à la date d'évaluation, la date réelle d'enregistrement n'est pas retournée. Toutes les valeurs présentes dans le résultat de la requête auront donc le même horadatage.

<u>Affichage dans Prometheus</u> (`http://localhost:9090/graph`)

![Affichage Prometheus intant query](./img/instant_query_prometheus.png)
![Appel réseau instant query](./img/instant_query_network.png)

<b>2) <u>requête sur un intervalle de temps</u></b>  (`range_query`)

![Range Query](./img/range_query.png)

Le résultat contient pour chaque série l'ensemble des valeurs présentes sur un intervalle de temps donné.

<u>Appel REST</u>

**GET /api/query**
* query=prometheus_http_requests_total [1m]
* time=1682778791.976

<pre>
{
  "status":"success",
  "data":{
    "resultType":"matrix",
    "result":[
    {
      "metric":{
        "__name__":"prometheus_http_requests_total",
        "code":"200",
        "handler":"/-/ready",
        "instance":"localhost:9090",
        "job":"prometheus"
      },
      "values":[
        [1682778742.532,"1"],
        [1682778757.532,"1"],
        [1682778772.532,"1"],
        [1682778787.532,"1"]
      ]
    },
    ...
    ]
  }
}
</pre>

`values` est un tableau à deux dimensions, contenant un ensemble de tuple sous la forme `[date, valeur]`. Cette fois-ci, les dates associées aux valeurs correspondent aux dates d'enregistrement présentes en base.

<u>Affichage dans Prometheus</u> (`http://localhost:9090/graph`)

![Affichage Prometheus range query](./img/range_query_prometheus.png)
![Appel réseau range query](./img/range_query_network.png)

#### <u>Filtre</u>

**prometheus_http_requests_total{handler="/metrics", code="200", job=~".\*theus"}**

```
prometheus_http_requests_total{code="200", handler="/metrics", instance="localhost:9090", job="prometheus"} 11
```
#### <u>Aggrégation</u>

**count(prometheus_http_requests_total)**
```
{}  12
```

**count by (code) (prometheus_http_requests_total)**
```
{code="200"}  10
{code="302"}  1
{code="400"}  1
```

**sum(prometheus_http_requests_total)**
```
{}  67
```
