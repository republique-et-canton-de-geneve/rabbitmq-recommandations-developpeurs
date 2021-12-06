# Gérer l'indisponibilité de RabbitMQ

## Le problème

Que doit faire le producteur ou le consommateur pour que les conséquences d'une coupure de la connexion
à RabbitMQ soient automatiquement surmontées quand la connexion est rétablie ?

## Discussion

Le découplage est le principal avantage recherché avec la mise en place d'un système asynchrone.
Il se traduit par le fait que les producteurs et consommateurs de messages peuvent travailler à des
moments différents et à leur rythme, c'est le découplage temporel ;
ou par le fait que producteurs et consommateurs n'ont pas d'informations l'un sur l'autre,
c'est le découplage physique : ils se contentent d'avoir un accord sur les formats des messages échangés.

Les producteurs et consommateurs de messages doivent s'affranchir de la disponibilité de l'agent RabbitMQ.
Le trafic doit s'interrompre si une anomalie est détectée et reprendre lorsque les systèmes redeviennent
disponibles.

Référence documentaire :
[Connection Recovery](https://www.rabbitmq.com/consumers.html#connection-recovery).

## Recommandations

### a) Mettre en attente les messages sortants

Concerne : les producteurs.

Architecture découragée :

```
Système principal :
1. traitement métier
2. production du message RabbitMQ
```

Architecture recommandée :

```
Système principal :
1. traitement métier

Système asynchrone :
2. production du message RabbitMQ
```

La première architecture est du type "ça passe ou ça casse".
Son couplage la rend très sensible aux erreurs pouvant survenir dans l'envoi du message à RabbitMQ,
qui peuvent bloquer la transaction du traitement métier.
Elle ne permet pas de réémettre le message à RabbitMQ en cas d'erreur.

La seconde architecture est à privilégier.
Le système asynchrone peut travailler à la pièce ou par lots (batch).
Il faut une communication minimale entre le système principal et le système asynchrone, afin que
celui-ci puisse identifier les messages à envoyer ;
la base de données, s'il y en a une, est un bon candidat.


### b) Reprendre l'activité dès la détection de la disponibilité de RabbitMQ

Concerne : les producteurs et les consommateurs.

Exemple. Un test avec Java et Spring RabbitMQ avec la configuration par défaut montre une reprise
automatique de l'activité interrompue, dès la remise à disposition de RabbitMQ.
Le test est celui-ci :

```
1. Lancer RabbitMQ
2. Envoyer un message 1. Résultat attendu : OK
3. Arrêter RabbitMQ
4. Envoyer un message 2. Résultat attendu : KO (exception), car RabbitMQ est arrêté
5. Relancer RabbitMQ
6. Envoyer un message 3. Résultat attendu : OK, car RabbitMQ est à nouveau disponible
```

Code :
```
public void run(String... args) throws InterruptedException {
    setupCallbacks();

    // Prealable : RabbitMQ doit etre lance

    // envoyer un message 1 : OK car RabbitMQ est lance
    sendMessage(1);
    wait(2, "Attente de l'acquittement");

    // arreter RabbitMQ a la main ici
    wait(30, "ATTENTION : VOUS AVEZ 30 SECONDES POUR ARRETER RABBIT !");

    // envoyer un message 2 : KO car RabbitMQ est arrete
    try {
        sendMessage(2);
    } catch(Exception e) {
        log.info("Exception obtenue :", e);
    }

    // relancer RabbitMQ a la main ici
    wait(30, "ATTENTION : VOUS AVEZ 30 SECONDES POUR RELANCER RABBIT !");

    // attendre un peu que RabbitMQ soit vraiment pret
    wait(5, "Quelques secondes supplementaires d'attente... un peu de patience...");

    // envoyer un message 3 : OK car RabbitMQ est reparti
    sendMessage(3);

    // attendre un peu, sinon on prend un nack pour cause de "clean channel shutdown"
    wait(2, "Attente de l'acquittement");
}
```

Résultat :
```
17:13:31 INFO  : Message envoye : (Body:'Message 1' MessageProperties [headers={}, contentType=text/plain, contentEncoding=UTF-8, contentLength=9, deliveryMode=PERSISTENT, priority=0, deliveryTag=0])
17:13:31 INFO  : Attempting to connect to: [localhost:5672]
17:13:31 INFO  : Created new connection: rabbitConnectionFactory#6272c96f:0/SimpleConnection@1cb37ee4 [delegate=amqp://guest@127.0.0.1:5672/, localPort= 51787]
17:13:31 INFO  : Attente commence. Attente de l'acquittement
17:13:31 INFO  : Confirmation : ack (cause : [null]) pour correlation [CorrelationData [id=Correlation for message 1]]
17:13:33 INFO  : Attente commence. ATTENTION : VOUS AVEZ 30 SECONDES POUR ARRETER RABBIT !
17:14:03 INFO  : Message envoye : (Body:'Message 2' MessageProperties [headers={}, contentType=text/plain, contentEncoding=UTF-8, contentLength=9, deliveryMode=PERSISTENT, priority=0, deliveryTag=0])
17:14:03 INFO  : Attempting to connect to: [localhost:5672]
17:14:07 INFO  : Exception obtenue :
org.springframework.amqp.AmqpConnectException: java.net.ConnectException: Connection refused: connect
...
org.springframework.amqp.rabbit.connection.CachingConnectionFactory$ChannelCachingConnectionProxy.createChannel(CachingConnectionFactory.java:1413)
...
Caused by: java.net.ConnectException: Connection refused: connect
17:14:07 INFO  : Attente commence. ATTENTION : VOUS AVEZ 30 SECONDES POUR RELANCER RABBIT !
17:14:37 INFO  : Attente commence. Quelques secondes supplementaires d'attente... un peu de patience...
17:14:42 INFO  : Message envoye : (Body:'Message 3' MessageProperties [headers={}, contentType=text/plain, contentEncoding=UTF-8, contentLength=9, deliveryMode=PERSISTENT, priority=0, deliveryTag=0])
17:14:42 INFO  : Attempting to connect to: [localhost:5672]
17:14:42 INFO  : Created new connection: rabbitConnectionFactory#6272c96f:2/SimpleConnection@1639f93a [delegate=amqp://guest@127.0.0.1:5672/, localPort= 51857]
17:14:42 INFO  : Attente commence. Attente de l'acquittement
17:14:42 INFO  : Confirmation : ack (cause : [null]) pour correlation [CorrelationData [id=Correlation for message 3]]
```

On obtient bien l'envoi réussi de Message 3, suite à une reconnexion automatique après la relance
de RabbitMQ.

Référence interne à l'État de Genève :
le code complet est disponible dans le projet GitLab `rabbitmq-producer-disconnected`.

### c) Mettre en œuvre la réémission ultérieure des messages bloqués

Cette question est traitée dans la page "Savoir réémettre un message" <à faire>.
