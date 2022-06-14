# Gérer les erreurs de traitement

## Le problème

Comment faire lorsque le consommateur doit-il faire face à une erreur,
comme un message mal formé, un contenu de message invalide ou une erreur technique lors du traitement ?

## Discussion

Le cas nominal de traitement d'un message est celui où le consommateur réussit son traitement et éventuellement
envoie un acquittement à l'agent RabbitMQ.
Or le traitement peut mal se passer, par exemple, en Java, quand le consommateur lance une exception.
À ce stade, habituellement le consommateur va rendre à l'agent RabbitMQ un acquittement négatif (*nack*)
et RabbitMQ va remettre le message dans la queue.
Si la queue ne contient que ce message, il sera donc immédiatement re-soumis, repartira en erreur,
retournera dans la queue, etc., indéfiniment et à toute vitesse.
Il faut donc un mécanisme pour "calmer le jeu" ;
les boîtes aux lettres mortes (dead letter queues ou DLQs) servent à cela.

### Types d'erreurs

On peut distinguer deux types d'erreurs :
- les erreurs de producteur. Quelques exemples :
  le message est mal formé (mauvais JSON, mauvais XML) ;
  le message ne contient pas une donnée obligatoire ;
  la combinaison des données du message est fonctionnellement invalide (par exemple, pays=Suisse et ville=Paris) ;
  le type de media (*media type*) du message n'est pas pris en charge.
- les erreurs de consommateur. Quelques exemples :
  une anomalie s'est produite dans le code du consommateur ;
  la mémoire est épuisée ;
  la connexion à la base de données est interrompue ;
  un service REST dont dépend le consommateur est indisponible.
  On notera que ces erreurs ne sont pas forcément dues au consommateur,
  mais il lui revient dans tous les cas de les prendre en charge.

### Queues de réponse

Au nom de la fiabilité,
il est tentant de créer une queue à destination du producteur dans laquelle le consommateur créerait
un court message d'acquittement indiquant que le traitement du message a réussi ;
elle permettrait de producteur de savoir précisément quels messages ont été traités et quels messages semblent
perdus.
Il est cependant recommandé de résister à cette tentation :
le traitement d'une telle queue par le producteur serait lui-même une nouvelle source de complexité
et de non-fiabilité (par exemple, cette queue a-t-elle à son tour besoin d'une DLQ ?).
Il vaut mieux s'en tenir au mode unidirectionnel *fire and forget* (aux erreurs de traitement près),
et ne pas chercher à recréer un mode requête-réponse pseudo-asynchrone.
Si vraiment le producteur tient à recevoir des acquittements métier, alors il faut considérer l'abandon
de RabbitMQ et l'utilisation d'un protocole synchrone classique comme les services REST. 

On notera que ces messages d'acquittement métier n'ont rien à voir avec les acquittements techniques
de RabbitMQ décrits dans la page
[Ne pas perdre de messages](./acquittements.md).


### Identifiant de corrélation

Le producteur du message original voudra en général croiser le message d'erreur avec le message original.
D'emblée, ce croisement ne se fait pas, car techniquement le message d'erreur n'est pas la réponse
synchrone de l'envoi du message original, mais est un nouveau message envoyé de manière asynchrone
dans l'autre sens.
Le producteur doit ainsi inclure un identifiant de corrélation dans son message.

Voir aussi la page [Échanger des métadonnées sur chaque message](./echanger_des_metadonnees.md).

## Recommandations

### a) Organiser la gestion des incidents en coordonnant les équipes

Organiser l'entreprise de telle sorte que les équipes
(développement d'application, RabbitMQ, réseau, sécurité, exploitation)
se sentent impliquées, se parlent et coordonnent leurs efforts quand une erreur difficile à
diagnostiquer se produit. 

### b) Établir une distinction parmi les erreurs

Le consommateur doit distinguer les erreurs "de producteur" des erreurs "de consommateur",
comme définies plus haut.
Les actions ci-dessous en dépendent.

Exemple avec Java et Spring.
N'inclut pas la recommandation ci-dessus d'utiliser les codes HTTP de retour (c'est à faire !).

Classe du consommateur. Un try-catch distingue les traitements réussis des traitements en erreur :
```
/**
 * Le principal point d'entree de l'application : consommation d'un message RabbitMQ.
 */
@RabbitListener(queues = "SOME_QUEUE")
public void consume(Message message) {
    try {
        ...   // le traitement du message
    } catch (Exception e) {
        responseHandler.handleKo(e, message);
    }
}
```

Classe `ResponseHandler`.
On distingue les erreurs dues au producteur (→ queue de réponse)
des erreurs dues au consommateur (→ boîte aux lettres mortes) :
```
/**
 * Si erreur due au producteur, le message est rejeté dans la DLQ du producteur.
 * Si erreur due à ce consommateur, le message est rejeté dans la queue d'erreur du consommateur.
 */
public void handleKo(Exception e, Message message) {
    if (e instanceof ValidationException) {
        message.getMessageProperties().setHeader(ERROR, e.getMessage());
        log.info("Envoi a la DLQ producteur du message [{}], cause : {}", message, e.getMessage());
        dlq1Template.send(dlx1, "", message);
    } else {
        message.getMessageProperties().setHeader(TECHNICAL_ERROR, e.getMessage());
        log.error("Envoi a la DLQ consommateur du message [{}], cause : {}", message, e);
        dlq2Template.send(dlx2, "", message);
    }
}
```

### c) Créer des boîtes aux lettres mortes

Créer trois boîtes aux lettres mortes (DLQ) :

| Boîte | Propos | Remplie par | Destinée à | Dépouillée par |
| :---: | :----- | :---------: | :--------: | :------------: |
| DLQ 1 | Erreurs de producteur (par ex., une donnée manquante dans le message) | consommateur | producteur | un humain |
| DLQ 2 | Erreurs temporaires de consommateur (par ex., un épuisement de mémoire) | consommateur | consommateur | consommateur |
| DLQ 3 | Erreurs fatales de consommateur (par ex., une NullPointerException) | consommateur | consommateur | un humain |


Détails :
- créer chaque fois une paire échange + une boîte aux lettres mortes (DLX et DLQ)
- techniquement, la DLQ 1 peut être définie comme la DLQ (forcément unique) de la queue en question
- pour la DLQ 3, l'équipe du consommateur doit se coordonner avec l'équipe du producteur,
  ne serait-ce que pour l'avertir du retard possible du traitement.
  Techniquement, le rejeu du message peut être soit à la charge du producteur (voir page
  [Savoir réemettre un message](./reemettre_un_message.md)),
  soit à la charge du consommateur
- ajouter dans un en-tête du message une description de l'erreur constatée

### d) Nommer soigneusement les échanges et les queues

- réfléchir à l'intention précise des chaque échange et de chaque queue
- une convention à l'État de Genève est de suffixer les noms des échanges par "-x"
  et les noms des queues par "-q". Cela facilite la lecture

### e) Ne pas créer de queue de réponse

Voir le paragraphe "Queues de réponse" dans la discussion ci-dessus.

### f) Prévoir un identifiant de corrélation

Le producteur inclut dans son message (habituellement, dans la propriété `correlation_id` plutôt que
dans un en-tête) un identifiant de corrélation.
Cet identifiant peut être n'importe quelle valeur, elle doit juste être unique.

### g) Utiliser les standards du Web

Utiliser les codes HTTP standard de retour : 201 (Created), 403 (Forbidden), etc.

### h) Installer des sondes

Faire installer des sondes par chaque équipe ; ces sondes portent notamment sur le nombre de messages
traités et le nombre d'erreurs relevées.
Ces sondes doivent englober chaque maillon du système, à savoir non seulement les fonctions de soumission
de messages des producteurs et de traitement de messages des consommateurs,
mais aussi le serveur RabbitMQ lui-même et les boîtes mortes (DLQ) des consommateurs.

### i) Réconcilier les relevés des sondes

Agréger les résultats des sondes et vérifier le bon état fonctionnel du système, c'est-à-dire essentiellement
la bonne consommation des messages.
Si nécessaire, faire rejouer des messages par les producteurs.
Cette tâche est à assurer par le propriétaire de RabbitMQ
(voir les rôles dans [gouvernance](./gouvernance.md)).

Baser cette chaîne de sondes et de réconciliation sur des outils externes à RabbitMQ,
afin de ne pas surveiller le système par le système lui-même. 
À l'État de Genève, les sondes sont basées sur l'outil Splunk qui agrège les différents fichiers
de traces.
