# Prendre en main RabbitMQ

1 LIEN 

## Le problème

Comment se frotter à RabbitMQ quand on n'en a pas l'habitude ?

## Discussion

L'État met à disposition des développeurs des serveurs RabbitMQ de développement.
Cependant, comme ceux-ci sont conçus pour être répliqués en recette et en production, ils sont déjà
munis d'un attirail nécessaire, mais qui est gênant pour le développeur novice :
connexion via TLS, virtual hosts, authentification via Gina et UAA.
Pour s'habituer à créer des messages et à les consommer, il vaut mieux commencer par quelque chose de
plus simple.

Pour la connexion à RabbitMQ avec les contraintes de l'État, voir la XXX page A2 - Utiliser RabbitMQ à l'État.

## Recommandations

### a) Utiliser un conteneur Docker

C'est étonnamment simple : on lance une commande Docker et un serveur RabbitMQ est prêt à recevoir des
messages ; de plus une console RabbitMQ est automatiquement lancée.

Powershell :

```docker run -it -p 5672:5672 -p 15672:15672 rabbitmq:3.8-management```

Git bash :

```winpty docker run -it --rm -p 5672:5672 -p 15672:15672 rabbitmq:3.8-management```

Console : http://localhost:15672.
Le démon Docker - par exemple Docker Desktop - doit être au préalable lancé.

Référence : https://www.rabbitmq.com/download.html.

Référence interne :
une autre procédure de prise en main avec Docker, utilisant Docker Compose, est disponible
dans le [wiki](<URL wiki>/pageId=1832222965).
