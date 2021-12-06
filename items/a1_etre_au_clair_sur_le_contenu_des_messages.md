# Être au clair sur le contenu des messages

2 LIENS

## Le problème

Comment faire pour que l'équipe du côté de la production des messages sache exactement comment remplir
ses messages ?

## Discussion

Les équipes de développement du côté producteur et consommateur peuvent perdre beaucoup de temps et de
bonne humeur à se coordonner sur le contenu exact des messages.
Il arrive même que l'équipe qui définit le contenu des messages ne soit elle-même pas tout à fait au
clair sur certaines informations qu'elle fournit ou qu'elle attend.
Il importe donc que les contrats soient clairs.
Ces contrats sont l'équivalent des descriptions de méthodes en Java, JavaScript ou PHP, aux schémas XML
des services SOAP ou aux interfaces Open API des services REST.

Actuellement on a tendance à faire des messages, donc des contrats, au format JSON, mais d'autres
formats, comme XML, sont acceptables.
En début de projet, c'est une décision à prendre entre équipes.

Il faut également prévoir que les contrats ont des chances d'évoluer dans le temps, alors que les
systèmes sont déjà en production.
Le versionnement des contrats doit donc être abordé dès le début de leur conception.

## Recommandations

### a) Documenter

C'est probablement la clef de tout. Une documentation écrite des contrats doit être fournie et tenue à jour. Informations nécessaires :

    expliquer le contexte du message (à quoi il sert)
    pour chaque champ :
        nom
        type
        contraintes (champ obligatoire, taille maximale, champ obligatoire si un autre champ fourni, etc.)
        exemple de valeur
    liste des en-têtes

Le mieux est probablement d'insérer cette documentation dans le code source, comme le fichier README.md.
À la stocker dans un lieu externe, comme un wiki, on risque qu'elle soit moins consultée par les développeurs-
cible et moins soigneusement tenue à jour.

Référence interne : documentation du projet
[enu-mediation](<URL GITLAB>/ACCES_RESTREINT/3417_espace_numerique_usager/enu-mediation/-/blob/master/docs/messages.md).

Si les contrats sont en JSON, fournir une interface Open API (Swagger) comme pour des services REST est
appréciable.


### b) Privilégier les pratiques du Web

Utiliser un type de média (media type, anciennement MIME type) pour désigner le type de l'objet contenu dans le message. Voir la page Échanger des métadonnées sur chaque message.


### c) Versionner les contrats

Cette question importante est discutée dans la page Prévoir les évolutions du contenu des messages.
