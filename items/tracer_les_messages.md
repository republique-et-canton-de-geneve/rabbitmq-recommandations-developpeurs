# Tracer les messages

## Le problème

Comment s'assurer qu'en cas de problème sur le traitement d'un message, on disposera de l'information
nécessaire pour enquêter ?

## Discussion

L'idéal serait que pour chaque message toute la chaîne soit tracée : émission par le producteur,
réception par RabbitMQ, émission par RabbitMQ, réception par le consommateur.
Or des essais ont établi que RabbitMQ, paradoxalement, ne garantissait pas d'emblée de traçage
et que tout système de traçage ajouté dans RabbitMQ avait de grande chance de ne pas être fiable.

Pour mémoire, RabbitMQ propose deux modes de traçage :

| Mode | Effet | Commande d'activation |
|------|-------|-----------------------|
| [Firehose](https://www.rabbitmq.com/firehose.html) | Ajout de la possibilité de voir tous les messages, dans des queues ad hoc | `rabbitmqctl trace_on -p <VHOST>` |
| Plugin [rabbitmq_tracing](https://github.com/rabbitmq/rabbitmq-server/tree/master/deps/rabbitmq_tracing) | Ajout de la possibilité de tracer les messages dans des fichiers, via l'ajout à la console RabbitMQ d'un onglet Admin > Tracing | `rabbitmq-plugins enable rabbitmq_tracing` |

Mode Firehose :

- Un arrêt-relance du conteneur Docker vide les queues de traces (pas compris pourquoi) et bloque
  la création de nouveaux messages dans ces queues ;
  il faut alors relancer la commande d'activation.
  À plus forte raison, un redéploiement via Ansible vide les queues de traces

Mode plugin rabbitmq_tracing :

- "The rabbitmq_tracing plugin builds on top of the tracer and provides a GUI to capture traced messages
  and log them in text or JSON format files" ([réf.](https://www.rabbitmq.com/firehose.html))
- "This plugin is intended to be used in development and QA environments. It will increase RAM
  consumption and CPU usage of a node" ([réf.](https://github.com/rabbitmq/rabbitmq-tracing))
- Un arrêt-relance du conteneur Docker supprime les "traces" dans Admin > Traces ;
  le fichier est conservé.
  Il faut alors redéfinir la "trace", soit dans la console, soit via curl

Bref, compliqué et pas fiable, probablement à dessein.

Référence interne à l'État de Genève :
fiche Jira RABBITMQ-148.

## Recommandations

### a) Ne pas compter sur RabbitMQ pour obtenir les traces

Comme expliqué plus haut, il ne faut compter que sur les traces dans le système producteur et dans
le système consommateur.
RabbitMQ ne fournit que l'information instantanée, à savoir les messages actuellement en attente
de traitement.

### b) Tracer les messages à la production et à la consommation

Le producteur et le consommateur doivent tracer les messages émis et reçus.

Si les acquittements (publisher confirms et consumer acknowledgements) sont en place, tracer
l'acquittement (ack ou nack).


### c) Tracer les informations pertinentes, sans dévoiler le contenu du message

Ni le producteur, ni le consommateur ne doivent tracer le contenu (body) du message, qui peut
véhiculer des informations sensibles.
Il leur est recommandé de tracer les informations suivantes, prises dans les métadonnées du message :

| Métadonnée | Producteur | Consommateur | Trace obligatoire |
|------------|:----------:|:------------:|:-----------------:|
| échange | X | | oui |
| clef de routage | X | | oui |
| queue | | X | oui |
| horodatage (1) | X | X | oui |
| identifiant de corrélation | X | X | oui |
| utilisateur métier | X | X | non |
| empreinte (hash) | X | X | oui |
| type de media | X | X | non |

(1) souvent fourni d'emblée par le système de traces (par ex. en Java : SLF4J)

Voir aussi :
[Échanger des métadonnées sur chaque message](./echanger_des_metadonnees_sur_chaque_message.md).

### d) Coordonner les traces du producteur et du consommateur

La liste ci-dessus des éléments tracés doit être discutée entre le producteur et le consommateur.
En effet, il ne s'agit pas seulement que chacun dans son coin ait une trace des messages envoyés ou
reçus, il s'agit aussi qu'en cas de problème, producteur et consommateur aient
(dans le système de traces Splunk) le maximum d'éléments *communs* sur lesquels s'appuyer.
