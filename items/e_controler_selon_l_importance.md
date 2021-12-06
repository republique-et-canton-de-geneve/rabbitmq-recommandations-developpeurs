# Contrôler les envois de message selon leur importance

## Le problème

Comment surveiller la production en tenant compte des différences d'importance entre les messages ?  

## Discussion

L'utilisation d'un système d'échange de messages asynchrones permet de réaliser des échanges
d'information unitaires rapides et d'en traiter un volume important, tout en s'affranchissant de
certaines contraintes de disponibilité et de routage physique.
Par exemple, l'internet des objets (IOT) utilise fréquemment la messagerie pour transmettre à
intervalles réguliers les mesures de capteurs.
De même, les sites Internet utilisent souvent la messagerie pour capter les clics des utilisateurs
et recueillir ainsi d'utiles informations statistiques.

Tous les usages ne sont pas égaux : il faut définir un système de contrôle adapté à l'importance de
chaque type de message échangé.

Références documentaires :
- documentation RabbitMQ : [métriques disponibles sur les queues](https://www.rabbitmq.com/monitoring.html#queue-metrics)
- documentation RabbitMQ : [boîtes aux lettres mortes (dead letter exchanges)](https://www.rabbitmq.com/dlx.html)
- documentation RabbitMQ : [acquittements asynchrones](https://www.rabbitmq.com/confirms.html)

## Recommandations

Les contrôles à mettre en oeuvre sont cumulatifs :

| Importance du message | Exemple d'usage | Contrôles à mettre en place |
|-----------------------|-----------------|-----------------------------|
| Faible ("je peux perdre des messages") | IOT, statistiques | *Contrôle obligatoire* : gestion d'exception pour l'écriture et la lecture de message <br/> <br/> *Surveillance obligatoire* : surveillance du nombre de messages écrits et lus ; alertes sur le remplissage des queues d'erreur (DLX et DLQ) <br/> <br/> *Surveillance complémentaire* : alerte en cas d’allongement de la date de dernière lecture de message par les consommateurs |
| Moyenne ("les messages devraient arriver") | Demande de l'usager | *Contrôle obligatoire* : utilisation des acquittements asynchrones de bout en bout ; les consommateurs développent un suivi des messages acquittés permettant de rechercher et de consulter les erreurs, puis de réémettre les messages perdus <br/> <br/> *Contrôles complémentaires* : identifier chaque message, tracer la production, tracer la consommation, tracer les transferts et routages, centraliser les traces |
| Important ("je ne peux pas perdre de messages") | Comptabilité, dossiers métier avec délais légaux | *Contrôle obligatoire* : développer un système de contrôle indépendant du système d'échange (data warehouse, par exemple) <br/> <br/> *Contrôles complémentaires* : mettre en place des messages de retour, remonter les erreurs des consommateurs au producteur <br/> <br/> *Surveillance complémentaire* : tableau de bord Splunk de suivi des messages |
