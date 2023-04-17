# Découverte de Prometheus

Prometheus est un outil de surveillance applicative très connu dans le monde de l'observabilité.
Je présenterai dans cet article son fonctionnement global en insistant sur la partie requêtage de données avec le langage PromQL.

<br>

![Prometheus Schéma global](./img/prometheus.png)

<br>

Prometheus fonctionne en mode `Pull`, il collecte les métriques à interval régulier auprès des applications à surveiller.<br>
Ces métriques sont enregistrées dans une base de données temporelles. Le langage **PromQL** permettra de requêter les données par le biais d'une API Rest.

On retrouve dans Prometheus :

- un outil pour collecter les métriques.
- une base de données temporelles (Time Series DataBase).
- un langage de requête (PromQL).
- une API Rest pour exécuter les requêtes et gérer la plateforme.
- une interface graphique pour visualiser les données.

Il est recommandé d'utiliser **Grafana** pour l'affichage des données. 
L'interface minimaliste de Prometheus sera utilisée uniquement dans la phase de développement pour tester les requêtes PromQL.

## Les métriques

Une métrique est une mesure numérique d'un élement de l'application, par exemple la quantité de mémoire RAM utilisée. Elles se déclinent  principalement en quatre types.

### Compteur

La valeur d'un compteur peut **uniquement augmenter**.

<u>Exemple</u> : Nombre de requêtes HTTP executées par un serveur web.

<br>

![Compteur](./img/compteur.png)

<br>

### Jauge

La valeur d'une jauge peut **augmenter ou diminuer**.

<u>Exemple</u> : Température d'un processeur.

<br>

![Jauge](./img/jauge.png)

<br>

### Histogramme

Un histogramme compte le nombre de données selon certaines catégories prédéfinies.

<u>Exemple</u> : Durée d'exécution des requêtes HTTP.

<br>

![Jauge](./img/histogramme.png)

<br>

Ici, 10 requêtes ont un temps d'exécution compris entre 0,5s et 1s.

Le nombre de requêtes de chaque catégorie augmentera au cours du temps. Comme pour les compteurs, cette valeur ne pourra pas diminuer.

### Résumé

Un résumé calcule la valeur de certains quantiles.

<u>Exemple</u> : Durée d'exécution des requêtes HTTP.

<br>

![Résumé](./img/resume.png)

<br>

Au vu du dernier résultat, on déduit que 99% des requêtes s'exécutent dans un temps inférieur ou égal à 650ms.

## Format des métriques

Les applications à surveiller doivent exposer leurs métriques par le biais d'un service HTTP de type GET.<br> 
Ce service retournera l'intégralité des métriques dans un format texte compréhensible par Prometheus.

<b>Counter</b>

![Résumé](./img/compteur_format.png)

Chaque ligne (excepté les en-têtes) définit une nouvelle série temporelle qui sera identifiée par le nom de la métrique et les libellés de type clé-valeur.

Par convention, le nom d'un compteur se terminera par `_total`.

<b>Jauge</b>

<b>Histogram</b>

<b>Summary</b>


## Exposition des métriques

Pour obtenir les métriques d'une application existante, on peut :

- utiliser un exporter qui va venir se greffer à celle-ci et exposer un ensemble de métriques utiles à son observation.

- définir ses propres métriques en ajoutant du code. Prometheus fournit des libraries clientes dans plusieurs langages pour les gérer.

Il est donc possible grâce aux exporters d'observer une application existante sans avoir à ajouter le moindre code.

On retrouve par exemple le `Node Exporter` qui expose les métriques d'un système Linux, le `JMX Exporter` qui expose les métriques d'une application Java en se basant sur les données disponibles via JMX.

Notons que le serveur Prometheus expose lui-même un ensemble de métriques accessibles via le path `/metrics`, permettant ainsi d'observer son fonctionnement.

