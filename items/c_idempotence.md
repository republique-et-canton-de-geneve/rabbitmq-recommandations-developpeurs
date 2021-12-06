# Pouvoir consommer plusieurs fois le même message (idempotence)

## Le problème

Comment faire pour qu'un message puisse être envoyé sans dommage plusieurs fois au consommateur ?

## Discussion

Pour de multiples raisons, un message peut être délivré plusieurs fois au consommateur.
En voici quelques-unes :
- une anomalie dans le producteur ou une action utilisateur de renvoi fait que le même message est
  envoyé plusieurs fois.
- un message dont la réception n'a pas été acquittée par le consommateur est renvoyé par RabbitMQ.
- une erreur de configuration de RabbitMQ fait que le même message arrive au consommateur par des
  canaux différents.
- une anomalie dans le consommateur fait que le message est traité plusieurs fois.

Dans tous ces cas, il est important que le deuxième traitement ainsi que tous les autres qui pourraient
suivre aboutissent au même état du système d'information que le premier traitement.
Ceci peut se faire au moins de deux façons différentes :
- le traitement est rejoué et les modifications apportées au système d'information sont identiques à
  celles qui ont été effectuées la première fois.
- le consommateur parvient à déterminer que le message a déjà été traité et n'exécute donc pas le
  traitement de nouveau.

## Recommandations

### a) Le consommateur doit être capable de consommer plusieurs fois le même message (idempotence)

À FAIRE : critères d'idempotence
Rejeu ou blocage (détection).
