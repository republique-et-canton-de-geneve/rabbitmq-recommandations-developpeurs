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

### a) Le consommateur doit être capable de traiter plusieurs fois le même message (idempotence)

Exemple de mise en œuvre. Dans le code du consommateur :
- Placer un filtre au début du traitement.
  Le filtre écarte un message qu'il considère comme un doublon, tout en lançant éventuellement une trace ;
  sinon le filtre laisse passer le message pour qu'il soit traité
- Contenu du filtre : calculer l'empreinte (hash) du message et la comparer avec des empreintes précédentes
- L'empreinte porte sur une sélection des champs du message ; on prend ordinairement quasiment toutes
  les données fonctionnelles, éliminant par exemple certains horodatages
- Les empreintes précédentes peuvent être stockées dans une petite base de données - SQL ou non - avec trois
  champs : l'empreinte, l'identifiant de corrélation (pour l'audit) et le délai d'expiration
- Afin d'éviter un engorgement, une tâche planifiée supprime régulièrement de la base de données les entrées
  dont le délai d'expiration est passé.

On notera que l'identifiant de corrélation n'est pas un bon critère de détection d'un doublon, notamment du
fait qu'il y a de fortes chances que le producteur change d'identifiant de corrélation lorsqu'il produit
un doublon.
Il faut vraiment se baser sur des données fonctionnelles.
