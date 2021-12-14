# RabbitMQ : recommandations pour les développeurs

Ce projet définit les recommandations élaborées à l'État de Genève
pour ses développeurs d'applications productices et consommatrices de messages
[RabbitMQ](https://www.rabbitmq.com).

## Présentation

L'usage de RabbitMQ à l'État de Genève provient de la nécessité du remplacement de JMS,
jugé vieillissant comme système middleware de gestion de queues.
En 2020, l'étude d'un cas en interne a mis en concurrence
[Apache Kafka](https://kafka.apache.org) et RabbitMQ.
Il en a été dégagé que les besoins de l'État de Genève étaient ceux d'une solution classique
de gestion de queues et non
d'une gestion de flux aux performances stratosphériques : (relative) simplicité, découplage, robustesse.
De plus, un système RabbitMQ complet peut être monté en open source, tandis que si le noyau de Kafka
est librement accessible, des éléments nécessaires à l'exploitation sont payants.
Aussi RabbitMQ a-t-il été retenu.

Le déploiement de RabbitMQ à l'État de Genève a été conçu selon les axes suivants :
- À chaque domaine fonctionnel son RabbitMQ.
On n'a donc pas cherché à établir un agent RabbitMQ global ou une ferme d'agents RabbitMQ globale ;
au contraire on a préféré compartimenter les domaines afin qu'un incident sur un agent d'un domaine
ne puisse pas affecter un autre agent.
- La sécurité est un élément clé de la solution.
Une application productrice ou consommatrice s'authentifie au système de gestion d'accès de l'État
(appelé Gina), puis la gestion des accès est assurée par un serveur
[UAA](https://docs.cloudfoundry.org/concepts/architecture/uaa.html)
qui récupère de Gina les rôles de l'usager via un appel LDAP.
L'accès humain à la console RabbitMQ est protégé de manière similaire, à la différence près que
le serveur UAA récupère les rôles de l'usager via un échange SAML plutôt qu'un appel LDAP.
- Le déploiement de chaque agent RabbitMQ et de l'UAA, en développement, en recette et en production,
s'effectue via des scripts Ansible et un serveur GoCD.

Pour les développeurs d'applications de l'État, en Java, en PHP ou en JavaScript,
l'usage de RabbitMQ n'est pas familier.
Aussi une série de recommandations et d'injonctions à leur intention a été rédigée ;
elle fait l'objet de ce projet Git.

## Recommandations

| Recommandation | Catégorie | Producteurs | Consommateurs |
|----------------|:---------:|:-----------:|:-------------:|
| [Prendre en main RabbitMQ](items/prendre_en_main_rabbitmq.md) | Outillage | X | X |
| [Utiliser RabbitMQ à l'État](items/utiliser_rabbitmq_a_l_etat.md) | Outillage | X | X |
| [Être au clair sur le contenu des messages](items/etre_au_clair_sur_le_contenu_des_messages.md) | Messages | X | X |
| [Échanger de petits messages lisibles](items/echanger_de_petits_messages_lisibles.md) | Messages | X | X |
| [Échanger des métadonnées sur chaque message](items/echanger_des_metadonnees_sur_chaque_message.md) | Messages | X | X |
| Gérer les erreurs de traitement | Messages | X | X |
| [Ne pas perdre de messages (acquittements)](items/acquittements.md) | Messages | X | X |
| [Prévoir les évolutions du contenu des messages](items/prevoir_les_evolutions_des_messages.md) | Messages | X | |
| [Gérer l'indisponibilité de RabbitMQ](items/gerer_l_indisponibilite.md) | Gestion opérationnelle | X | X |
| Savoir réémettre un message | Gestion opérationnelle | X | |
| [Pouvoir consommer plusieurs fois le même message (idempotence)](items/idempotence.md) | Gestion opérationnelle | | X |
| [Tracer les messages](items/tracer_les_messages.md) | Gestion opérationnelle | X | X |
| Respecter la bande passante allouée | Gestion des ressources de RabbitMQ | X | X |
| Utiliser un pool de connexions à RabbitMQ | Gestion des ressources de RabbitMQ | X | X |
| [Gérer la connexion à RabbitMQ](items/gerer_la_connexion.md) | Gestion des ressources de RabbitMQ | X | X |
| [Contrôler les envois de message selon leur importance](items/controler_selon_l_importance.md) | Suivi et surveillance | X | X |

Principaux auteurs : François Montmasson, Jérémy Meunier, Pierre Laroche, Yves Dubois-Pèlerin.