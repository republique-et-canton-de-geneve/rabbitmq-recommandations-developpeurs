# Échanger de petits messages lisibles

## Le problème

Quand des informations lourdes, comme des fichiers, doivent être échangées, comment éviter un
engorgement du système ?

## Discussion

La taille des messages (payload) devrait être le plus petite possible. En voici les raisons :

* un petit message transitera plus rapidement du producteur au consommateur.
* dans le cas d'un grand nombre de messages, les ressources nécessaires (mémoire, disque, CPU) au
  fonctionnement du système ne pourront être maîtrisées qu'avec de petits messages.
* un MOM est fait pour l'envoi d'information sous forme de (petits) messages et pas pour le transfert
  de (gros) fichiers.

Le contenu des messages qui sont échangés devraient être lisibles.
Dans le cadre d'une communication inter SI, cela simplifie grandement le débogage en cas de problème
si on peut facilement comprendre le contenu d'un message.
La recherche de traces dans Splunk sera plus simple.
Cela n'empêche pas de chiffrer une partie du contenu si le secret de certaines informations est nécessaire.

## Recommandations

### a) Échanger de petits messages

La taille maximale des messages autorisée sur les agents RabbitMQ de l'État est de **128 ko**.


### b) Utiliser le format JSON

Il convient en tout cas d'utiliser un format texte, notamment pour les traces.

Par rapport au format XML, le format JSON est plus compact et plus lisible.


### c) Ne pas échanger de fichiers

L'échange de fichiers via RabbitMQ est proscrit.

Si des échanges de fichiers sont nécessaires, il convient de passer par la GED (gestion électronique de 
documents).
La séquence est la suivante :
1. le producteur sauve son document dans la GED
2. le producteur envoie un message à RabbitMQ. Ce message contient les identifiants GED du document (pas
   le contenu du document)
4. le consommateur reçoit le message
5. le consommateur va rechercher le document dans la GED à partir des identifiants trouvés dans le message
