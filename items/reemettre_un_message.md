# Savoir réémettre un message

## Le problème

Comment éviter qu'un message qui n'a pas été bien traité ne soit pas perdu ?

## Discussion

De nombreuses choses peuvent se passer entre le moment où le message a été construit par le producteur
et celui où il est censé avoir été traité par le consommateur.
Voici quelques exemples :
* le matériel supportant le producteur lâche avant l'envoi
* le producteur part en dépassement de mémoire avant l'envoi
* une anomalie dans le code du producteur fait que le message n'est pas envoyé
* l'agent RabbitMQ est indisponible au moment de l'envoi
* la configuration de l'agent RabbitMQ fait que le message n'est pas délivré au bon consommateur
* le message a expiré alors que le consommateur n'a pas encore pu le lire
* le matériel supportant le consommateur lâche avant le traitement du message
* le consommateur part en dépassement de mémoire avant le traitement du message
* une anomalie dans le code du consommateur fait que le message n'est pas traité.

Il faut donc considérer comme illusoire une configuration du producteur, de RabbitMQ et du
consommateur où aucun message ne se perdrait jamais.
Il faut au contraire prévoir que le producteur soit en mesure de rejouer les messages perdus ;
lui et le consommateur auront au préalable identifié les messages perdus, par exemple via une
confrontation de leurs traces respectives, cf.
[Tracer les messages](./tracer_les_messages.md)).

Le producteur peut se permettre d'être large dans sa sélection des messages à réémettre :
envoyer un message une fois de trop est normalement sans conséquence, car le consommateur doit s'être
protégé grâce à une conception idempotente de son traitement, cf.
[Pouvoir consommer plusieurs fois le même message](./idempotence.md).

On remarque que ce besoin de rejeu des messages condamne en pratique l'émission synchrone du message
par le producteur :
en effet, si l'application productrice est codée de telle sorte qu'elle réagit de façon synchrone
à un événement métier en émettant un message, elle est sans recours si son message ensuite se perd.
Elle a donc de toute façon besoin d'un système asynchrone d'envoi de message ;
autant que ce mode asynchrone soit son seul mode d'émission.

## Recommandations

### a) Émettre les messages de façon asynchrone

Attention : ce n'est pas parce que l'on utilise un système asynchrone de traitement de messages comme
RabbitMQ que l'émission de ces messages est nécessairement asynchrone !

### b) Détecter les messages perdus
