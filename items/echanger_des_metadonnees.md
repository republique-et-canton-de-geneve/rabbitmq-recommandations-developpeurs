# Échanger des métadonnées sur chaque message

## Le problème

Comment s'assurer que le suivi du traitement de chaque message (traces, identification de la réception
ou non-réception du message) sera possible ?

## Discussion

Indépendamment du bon codage du producteur et du consommateur, il importe que chaque message contienne,
en plus de ses données métier, certaines informations techniques.
Celles-ci peuvent être introduites dans les en-têtes (headers) du message.

## Recommandations

Voici les métadonnées qu'il est recommandé d'inclure dans chaque message :

| Métadonnée | Contenu | Utilité | Propriété AMQP (1) |
|------------|---------|---------|--------------------|
| Identification de l'émetteur | Un identifiant de l'application productrice | Cet identifiant devrait être connu du consommateur du message. Si ce n'est pas le cas, le consommateur pourrait refuser de traiter le message. <br/> Le code applicatif à 5 chiffres est un bon candidat. | `app_id` |
| Identification de corrélation du message | Un identifiant unique généré par le producteur lors de la construction du message | Cet identifiant de corrélation permet *éventuellement* de déterminer si un message a déjà été traité par le consommateur, potentiellement en conjonction avec l'empreinte du contenu. Dans tous les cas, recevoir 2 messages ayant même identifiant devrait alerter le consommateur. <br /> Par ailleurs, cet identifiant sert à corréler au message l'éventuelle réponse du consommateur (ou DLQ). | `correlation_id` |
| Identification de l'utilisateur à l'origine de l'envoi du message | Le code de l'utilisateur qui est à l'origine de la création du message | Le code de l'utilisateur peut être utilisé en conjonction avec d'autres métadonnées dans l'activité de support. <br/><br/> Cette métadonnée fait l'objet d'une discussion distincte, cf. [imputabilité](./imputabilite.md). | - |
| Horodatage du message | Un timestamp généré par le producteur à la construction du message | Le timestamp peut également être utilisé en conjonction avec d'autres métadonnées dans l'activité de support. <br/> Cette propriété semble automatiquement ajoutée par le protocole AMQP. | `timestamp` |
| Empreinte du contenu du message | Le hash du contenu métier du message généré par le producteur à la création du message | Cette empreinte sert à garantir l'intégrité du message. <br /> Afin de lever toute incertitude pour le consommateur, il est recommandé d'utiliser deux en-têtes pour cette empreinte, par exemple "hash_algorithm" pour l'algorithme utilisé (par ex. "SHA256") et "hash_value" pour l'empreinte elle-même. <br /> Pour un exemple de code, voir ci-dessous. | - |
| Type de média du message | Le type de média (anciennement : type MIME) du contenu du message | Si plusieurs formats de contenu peuvent être envoyés, cela aide le consommateur à savoir comment décoder le contenu. | `content_type` |
| Codage de contenu | Un ou plusieurs types de codage | Permet au consommateur de désérialiser correctement le message. Exemples de valeurs : `UTF-8`, `gzip`. Voir aussi [ici](https://www.rabbitmq.com/publishers.html). | `content_encoding` |

(1) si la propriété AMQP n'existe pas, il faut utiliser dans le message un en-tête (header) plutôt qu'une
propriété.


*Exemple de code pour la génération de l'empreinte (hash)*

En Java, sans Spring, avec la bibliothèque `commons-codec` pour obtenir la classe `DigestUtils`.

Producteur :

```
String hash = DigestUtils.sha256Hex(message);
```

Consommateur :

```
String hash = DigestUtils.sha256Hex(new String(delivery.getBody(), StandardCharsets.UTF_8));
```

On obtient la même valeur de l'empreinte chez le producteur et chez le consommateur.
