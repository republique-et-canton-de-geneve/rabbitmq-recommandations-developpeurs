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

Comme expliqué plus haut, il ne faut compter que sur les traces ajoutée par le développeur
dans le système producteur et dans le système consommateur.
RabbitMQ ne fournit que l'information instantanée, à savoir les messages actuellement en attente
de traitement ; c'est insuffisant.

### b) Tracer les messages à la production et à la consommation

Le producteur et le consommateur doivent tracer les messages émis et reçus.

Si les acquittements (publisher confirms et consumer acknowledgements) sont en place, tracer
l'acquittement (ack ou nack).


### c) Tracer les informations pertinentes, sans dévoiler le contenu du message

Ni le producteur, ni le consommateur ne doivent tracer le contenu (body) du message, qui peut
véhiculer des informations sensibles.
Il leur est recommandé de tracer les informations suivantes, prises dans les métadonnées du message :

| Métadonnée | Producteur | Consommateur | Trace obligatoire | But de la trace |
|------------|:----------:|:------------:|:-----------------:|-----------------|
| échange RabbitMQ | X | | oui |  Suivre l'aiguillage du message dans RabbitMQ |
| clef de routage RabbitMQ | X | | oui | Suivre l'aiguillage du message dans RabbitMQ |
| queue RabbitMQ | | X | oui | Suivre l'aiguillage du message dans RabbitMQ |
| identifiant de corrélation | X | X | oui | Identifier précisément le message |
| identifiant de l'émetteur  | X | X | oui | Identifier l'application émettrice |
| horodatage (timestamp) | X | X | non | UTILE ? |
| utilisateur métier | X | X | non | Imputer l'initiative du message à une personne |
| empreinte (hash) | X | X | oui | Contrôler l'intégrité du message |
| type de media (media_type) | X | X | non | Disposer d'un complément d'informations |
| codage du contenu (content_encoding) | X | X | non | Disposer d'un complément d'informations |
| mode d'envoi (delivery mode) | X | | non | Disposer d'un complément d'informations en cas de perte d'un message |

Voir aussi :
[Échanger des métadonnées sur chaque message](./echanger_des_metadonnees.md).

Exemple en Java :

Code du producteur
```
    @Override
    public void run(String... args) {
        String messageBody = "title: Un joli message (" + LocalDateTime.now().get(MINUTE_OF_HOUR) + ")";
        String appId = "10989";
        String correlationId = UUID.randomUUID().toString();
        String idResponsibleUser = "AGENT-XXX";
        Date timestamp = new Date();
        String hashAlgorithm = "SHA256";
        String hashValue = DigestUtils.sha256Hex(messageBody);
        String mediaType = "application/silly-message-v1.0+json";
        String contentEncoding = "UTF-8";

        rabbitTemplate.convertAndSend(
                EXCHANGE,
                ROUTING_KEY,
                messageBody,
                message -> {
                    message.getMessageProperties().setAppId(appId);
                    message.getMessageProperties().setCorrelationId(correlationId);
                    message.getMessageProperties().setHeader("idResponsibleUser", idResponsibleUser);
                    message.getMessageProperties().setTimestamp(timestamp);
                    message.getMessageProperties().setHeader("hashAlgorithm", hashAlgorithm);
                    message.getMessageProperties().setHeader("hashValue", hashValue);
                    message.getMessageProperties().setHeader("mediaType", mediaType);
                    message.getMessageProperties().setContentEncoding(contentEncoding);
                    message.getMessageProperties().setDeliveryMode(PERSISTENT);
                    log.info("Production : message [{}] envoye a echange [{}], clef de routage [{}]",
                            message.getMessageProperties(), EXCHANGE, ROUTING_KEY);
                    return message;
                }
        );
    }
```

Exécution du producteur

```
14:37:54.813 INFO  : Production : message [MessageProperties [headers={mediaType=application/silly-message-v1.0+json, idResponsibleUser=AGENT-XXX, hashAlgorithm=SHA256, hashValue=4d292223b1f9b60a3a47bfa0acb839740dca2a0d678adf1e5178e6403a0491bc}, timestamp=Mon Feb 07 14:37:54 CET 2022, appId=10989, correlationId=b98e026a-526a-438f-aa1c-4ab326b0ee01, contentType=text/plain, contentEncoding=UTF-8, contentLength=27, deliveryMode=PERSISTENT, priority=0, deliveryTag=0]] envoye a echange [exchange1], clef de routage [queue1]
```

Code du consommateur

```
    @RabbitListener(queues = QUEUE)
    public void receiveMessage(Message message) {
        log.info("Consommation : message [{}] recu", message.getMessageProperties());
        // ...
    }
```

Exécution du consommateur

```
Consommation : message [MessageProperties [headers={mediaType=application/silly-message-v1.0+json, idResponsibleUser=AGENT-XXX, hashAlgorithm=SHA256, hashValue=4d292223b1f9b60a3a47bfa0acb839740dca2a0d678adf1e5178e6403a0491bc}, timestamp=Mon Feb 07 14:37:54 CET 2022, appId=10989, correlationId=b98e026a-526a-438f-aa1c-4ab326b0ee01, contentType=text/plain, contentEncoding=UTF-8, contentLength=0, receivedDeliveryMode=PERSISTENT, priority=0, redelivered=false, receivedExchange=exchange1, receivedRoutingKey=queue1, deliveryTag=1, consumerTag=amq.ctag-eU8ajmGoG4_SY1Hiejfg6A, consumerQueue=queue1]] recu
```

Note interne à l'État de Genève :
le code est disponible dans le projet GitLab `rabbitmq-traces`.

### d) Coordonner les traces du producteur et du consommateur

La liste ci-dessus des éléments tracés doit être discutée entre le producteur et le consommateur.
En effet, il ne s'agit pas seulement que chacun dans son coin ait une trace des messages envoyés ou
reçus, il s'agit aussi qu'en cas de problème, producteur et consommateur aient
(dans le système de traces Splunk) le maximum d'éléments *communs* sur lesquels s'appuyer.
