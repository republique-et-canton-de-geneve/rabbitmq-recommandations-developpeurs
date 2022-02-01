# Imputer chaque message à un usager 

## Le problème

Comment s'assurer que le consommateur de tout message connaîtra l'origine fonctionnelle
du message qu'il traite ?

## Discussion

Il s'agit ici de sécurité fonctionnelle :
à l'État de Genève, on impose qu'une opération métier sur un système soit toujours menée au nom
d'un usager identifiable.
L'usager est, par exemple, un agent de l'Etat ou un citoyen.
Le producteur du message est nécessairement au courant de l'identité de cet usager ;
il lui incombe de transmettre cette information au consommateur.

## Recommandations

### a) Inclure l'identifiant de l'usager dans le message

<i>
Cette recommandation est à clarifier :
Comment est définie cette valeur ?
Dans un lot AFC, des messages sont envoyés automatiquement sur certains critères fonctionnels. Quelle valeur mettre ?
Le consommateur doit-il conserver cette valeur ? Si oui, combien de temps ?
Être capable de remonter la chaîne de transmission de l'information est-il suffisant
</i>

On peut inclure cet identifiant soit les métadonnées du message, soit dans le corps du message.
Dans le second cas, si le message contient des informations chiffrées et qu'il est convenu entre le
producteur et le consommateur de tracer l'identifiant de l'usager
(cf. [tracer les messages](./tracer_les_messages.md)),
alors cet identifiant doit figurer dans la partie non chiffrée du corps du message.
