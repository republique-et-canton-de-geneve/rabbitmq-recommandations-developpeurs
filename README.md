# RabbitMQ : recommandations aux développeurs

Ce projet définit les recommandations élaborées à l'État de Genève
pour ses développeurs d'applications productrices et consommatrices de messages
[RabbitMQ](https://www.rabbitmq.com).

## Recommandations

| Recommandation | Catégorie | Producteur | Consommateur |
|----------------|:---------:|:----------:|:------------:|
| [Coordonner les équipes](items/gouvernance.md) | Gouvernance | X | X |
| [Prendre en main RabbitMQ](items/prendre_en_main_rabbitmq.md) | Outillage | X | X |
| [Utiliser RabbitMQ à l'État](items/utiliser_rabbitmq_a_l_etat.md) | Outillage | X | X |
| [Être au clair sur le contenu des messages](items/etre_au_clair_sur_le_contenu_des_messages.md) | Messages | X | X |
| [Échanger de petits messages lisibles](items/echanger_de_petits_messages_lisibles.md) | Messages | X | X |
| [Échanger des métadonnées sur chaque message](items/echanger_des_metadonnees.md) | Messages | X | X |
| [Relier chaque message à un usager (imputabilité)](items/imputabilite.md) | Messages | X | |
| [Prévoir les évolutions du contenu des messages](items/prevoir_les_evolutions_des_messages.md) | Messages | X | |
| [Tracer les messages](items/tracer_les_messages.md) | Gestion opérationnelle | X | X |
| [Gérer les erreurs de traitement](items/gerer_les_erreurs.md) | Gestion opérationnelle | X | |
| [Ne pas perdre de messages (acquittements)](items/acquittements.md) | Gestion opérationnelle | X | X |
| [Savoir réémettre un message](items/reemettre_un_message.md) | Gestion opérationnelle | X | |
| [Pouvoir consommer plusieurs fois le même message (idempotence)](items/idempotence.md) | Gestion opérationnelle | | X |
| Savoir traiter les messages dans le bon ordre | Gestion opérationnelle | | X |
| [Gérer l'indisponibilité de RabbitMQ](items/gerer_l_indisponibilite.md) | Gestion opérationnelle | X | X |
| Respecter la bande passante allouée | Gestion des ressources de RabbitMQ | X | X |
| Utiliser un pool de connexions à RabbitMQ | Gestion des ressources de RabbitMQ | X | X |
| [Gérer la connexion à RabbitMQ](items/gerer_la_connexion.md) | Gestion des ressources de RabbitMQ | X | X |
| [Contrôler les envois de message selon leur importance](items/controler_selon_l_importance.md) | Suivi et surveillance | X | X |

## Présentation

L'usage de RabbitMQ à l'État de Genève provient de la nécessité du remplacement de JMS,
jugé vieillissant comme système middleware de gestion de queues.
En 2020, l'étude d'un cas en interne a mis en concurrence
[Apache Kafka](https://kafka.apache.org) et RabbitMQ.
Il en a été dégagé que le besoin de l'État de Genève n'était pas ceux d'une gestion de flux aux performances
stratosphériques, mais ceux d'une solution plus classique de gestion de queues : (relative) simplicité,
découplage, robustesse.
De plus, un système RabbitMQ complet peut être monté en open source, tandis que si le noyau de Kafka
est librement accessible, des éléments nécessaires à l'exploitation sont payants.
Aussi RabbitMQ a-t-il été retenu.

Le déploiement de RabbitMQ à l'État de Genève a été conçu selon les axes suivants :
- À chaque domaine fonctionnel son RabbitMQ.
  On n'a donc pas cherché à établir un agent RabbitMQ global ou une ferme d'agents RabbitMQ globale ;
  au contraire on a préféré compartimenter les domaines afin qu'un incident sur un agent d'un domaine
  ne puisse pas affecter un autre agent.
- La sécurité est un élément-clé de la solution :
  - Une application productrice ou consommatrice de messages utilise le protocole OpenID Connect pour
    obtenir un jeton (access token) d'un serveur
    [UAA](https://docs.cloudfoundry.org/concepts/architecture/uaa.html),
    lequel s'authentifie au système de général de gestion des accès de l'État (appelé Gina) et en
    récupère les rôles de l'usager via un appel LDAP.
  - l'accès humain à la console RabbitMQ, pour des raisons historiques, est protégé un peu différemment :
    l'usager s'authentifie à Gina, puis le serveur UAA récupère les rôles de l'usager,
    via un échange SAML et sans appel LDAP.
- Le déploiement de chaque agent RabbitMQ et de l'UAA, en développement, en recette et en production,
  s'effectue via des scripts Ansible et un serveur GoCD.

Pour les développeurs d'applications de l'État, en Java, en PHP ou en JavaScript,
l'usage de RabbitMQ n'est pas familier.
Aussi cette série de recommandations et d'injonctions à leur intention a été rédigée.

Principaux auteurs : François Montmasson, Jérémy Meunier, Pierre Laroche, Yves Dubois-Pèlerin.
