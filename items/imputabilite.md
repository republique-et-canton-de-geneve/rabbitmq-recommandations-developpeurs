# Imputer chaque message à un usager 

## Le problème

Comment s'assurer que le consommateur de tout message connaîtra l'origine fonctionnelle
du message qu'il traite ?

## Discussion

Il s'agit ici de sécurité fonctionnelle :
à l'État de Genève, on impose qu'une opération métier sur un système soit toujours menée au nom
d'un usager identifiable.
L'usager est, par exemple, un agent de l'État ou un citoyen.
Le producteur du message est nécessairement au courant de l'identité de cet usager ;
il lui incombe de transmettre cette information au consommateur.

## Recommandations

### a) Inclure l'identifiant de l'usager dans le message

On peut inclure cet identifiant soit les métadonnées du message, soit dans le corps du message.
Dans le second cas, si le message contient des informations chiffrées et qu'il est convenu entre le
producteur et le consommateur de tracer l'identifiant de l'usager
(cf. [tracer les messages](./tracer_les_messages.md)),
alors cet identifiant doit figurer dans la partie non chiffrée du corps du message.

### b) Être au plus près de l'usager humain

Si l'envoi du message n'est pas reliable à une action humaine, on se résoudra à fournir l'identifiant d'un
utilisateur technique.
C'est par exemple le cas de l'envoi en lot de messages dû à un événement automatique (par exemple, échéance
automatique) du cycle de vie de la donnée.

On veillera toutefois à ne considérer cette solution qu'en dernier recours, car d'autres solutions existent.
Par exemple :
- fournir l'identifiant d'un responsable de service, à l'instar de ce qui est souvent fait dans les formulaires
  papier envoyés par l'administration
- le fait que des messages soit envoyés en lots par un processus automatique n'implique pas nécessairement
  d'imputer ces messages à un utilisateur technique. L'envoi asynchrone peut n'être qu'un artifice
  de réalisation technique d'une opération déclenchée par un usager humain - agent de l'État ou citoyen

