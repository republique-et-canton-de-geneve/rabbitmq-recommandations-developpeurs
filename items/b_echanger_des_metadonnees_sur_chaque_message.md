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
| Identification de l'émetteur | Un identifiant de l'application productrice | Cet identifiant devrait être connu du consommateur du message. Si ce n'est pas le cas, le consommateur pourrait refuser de traiter le message. | `app_id` |
| Identification du destinataire | Un identifiant de l'application consommatrice | Cet identifiant devrait correspondre au consommateur du message. Si ce n'est pas le cas, le consommateur devrait refuser de traiter le message qui n'est a priori pas pour lui. <br />(CECI EST À VÉRIFIER : cette métadonnée n'est peut-être pas nécessaire, car le producteur n'est pas sensé connaître le destinataire. Le destinataire peut d'ailleurs être multiple) | - |
| Identification du message	| Un identifiant unique généré par le producteur lors de la construction du message | Cet identifiant permet *éventuellement* de déterminer si un message a déjà été traité par le consommateur, potentiellement en conjonction avec l'empreinte du contenu. Dans tous les cas, recevoir 2 messages ayant même identifiant devrait alerter le consommateur ! <br /> Par ailleurs, cet identifiant sert à corréler au message l'éventuelle réponse du consommateur (ou DLQ). | `correlation_id` |
| Identification de l'utilisateur à l'origine de l'envoi du message | Le code de l'utilisateur qui est à l'origine de la création du message | Le code de l'utilisateur peut être utilisé en conjonction avec d'autres métadonnées dans l'activité de support. <br/><br/> La Sécurité estime essentiel que tout message puisse être fonctionnellement imputable à une personne (agent de l'État ou usager). | - |
| Horodatage du message | Un timestamp généré par le producteur à la construction du message | Le timestamp peut également être utilisé en conjonction avec d'autres métadonnées dans l'activité de support. | `timestamp` |
| Empreinte du contenu du message | Le hash du contenu métier du message généré par le producteur à la création du message | Comme indiqué plus haut, l'empreinte du contenu du message permet d'aider à déterminer si un message a déjà été traité ou non par le consommateur. <br /> Pour un exemple de code, voir ci-dessous. | - |
| Type de média du message | Le type de média (anciennement : type MIME) du contenu du message | Si plusieurs formats de contenu peuvent être envoyés, cela aide le consommateur à savoir comment décoder le contenu. | `content_type` |

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
Les développeurs du producteur et du consommateur doivent se coordonner sur l'algorithme utilisé,
afin que les empreintes correspondent.
