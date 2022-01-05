# Gérer les erreurs de traitement

## Le problème

Comment faire lorsque le consommateur rencontre une erreur de traitement,
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
  le type de media (*media type*) du message n'est pas prévu.
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
cela permet de producteur de savoir précisément quels messages ont été traités et quels messages semblent
perdus.

Il est cependant recommandé de résister à cette tentation :
le traitement d'une telle queue par le producteur serait lui-même une nouvelle source de complexité
et de non-fiabilité (a-t-elle besoin à son tour de DLQ ?).
Il vaut mieux s'en tenir au mode unidirectionnel *fire and forget* (aux erreurs de traitement près),
et ne pas chercher à recréer un mode requête-réponse pseudo-asynchrone.

Noter que ces messages d'acquittement métier n'ont rien à voir avec les acquittements techniques
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

### a) Établir une distinction parmi les erreurs

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
 * Si erreur due au producteur, une réponse est renvoyée au producteur (queue de réponse).
 * Si erreur due a ce consommateur, le message est rejete dans la queue d'erreur (DLQ).
 */
public void handleKo(Exception e, Message originalMessage) {
    if (e instanceof ValidationException) {
        log.info("Envoi a RabbitMQ du message d'erreur client suivant : {}", e.getMessage());
        originalMessage.getMessageProperties().setHeader(ERROR, e.getMessage());
        replyTemplate.send(replyExchange, "", originalMessage);
    } else {
        log.error("Envoi a RabbitMQ d'un message d'erreur serveur (DLQ), suite a ", e);
        originalMessage.getMessageProperties().setHeader(TECHNICAL_ERROR, e.getMessage());
        dlqTemplate.send(deadLetterExchange, "", originalMessage);
    }
}
```

### b) Créer des boîtes aux lettres mortes

Créer plusieurs boîtes aux lettres mortes :

| Propos | Remplie par | Destinée à | Dépouillée par |
| :----- | :---------: | :--------: | :------------: |
| Erreurs de producteur | consommateur | producteur | un humain |
| Erreurs temporaires de consommateur (par ex., un épuisement de mémoire) | consommateur | consommateur | consommateur |
| Erreurs fatales de consommateur (par ex., une NullPointerException) | consommateur | consommateur | un humain |

Détails :
- créer chaque fois un échange et une boîte aux lettres mortes (DLX et DLQ)
- ajouter l'erreur constatée dans un en-tête du message

### b) Ne pas créer de queue de réponse

Voir le paragraphe "Queues de réponse" dans la discussion ci-dessus.

### c) Prévoir un identifiant de corrélation

- le producteur inclut dans son message (habituellement, dans la propriété `correlation_id` plutôt que
  dans un en-tête) un identifiant de corrélation.
  Cet identifiant peut être n'importe quelle valeur, elle doit juste être unique.
- le consommateur inclut l'identifiant de corrélation dans son message d'erreur.

### d) Utiliser les standards du Web

Utiliser les codes HTTP standard de retour, comme 403 (Forbidden), 201 (Created), etc.