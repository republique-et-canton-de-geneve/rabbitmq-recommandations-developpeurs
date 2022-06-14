# Assurer une gouvernance du système

## Le problème

Comment s'organiser pour que les différentes équipes coordonnent correctement leurs actions ?

## Discussion

Les échanges asynchrones de messages servent à séparer clairement les "clients" des "serveurs" :
les appels directs de fonctions sont supprimés ;
les contrats ne portent que sur des formats de données, échangées de surcroît dans un seul sens ;
les traitements sont asynchrones ;
la transactionnalité de bout en bout est proscrite.
Cette étanchéité recherchée présente cependant le risque de laisser des trous dans les responsabilités,
menant par exemple à des pertes de messages ou à des non-traitements de messages en erreur.

Le problème est d'autant plus prégnant que les développeurs devant intervenir dans
le développement des systèmes producteurs et consommateurs ont habituellement une bonne expérience de
la programmation "classique", comme la programmation par objets, les appels de services REST,
voire les flux réactifs, mais souvent peu de pratique de la programmation par échanges de messages.
Leur tendance est de se satisfaire de la simple réussite de leurs appels à RabbitMQ,
sans porter l'attention nécessaire à la qualité des traces,
à la possible indisponibilité de RabbitMQ ou à la perte de messages.
Il importe donc que les équipes se coordonnent entre elles, que leurs membres aient les compétences minimales
et que l'on ait clarifié ce qui relève des différentes équipes.

### Organisation

À l'État de Genève, on a été amené à définir 4 rôles :

| Rôle | Responsabilités | Commentaires |
|------|-----------------|--------------|
| Infrastructure | Mise à disposition de machines. <br> Installation de RabbitMQ. | Rôle habituel des équipes d'infrastructure dans des nombreuses organisations. |
| Support messagerie | Accueil des demandes de nouvelles plateformes RabbitMQ (nouveaux projets de messagerie). <br> Support aux équipes projets. <br> Mises à jour de RabbitMQ. <br> Respect des recommandations. | Cette série de recommandations est écrite par des personnes ayant ce rôle. |
| Propriétaire de RabbitMQ | Surveillance de la bonne marche applicative de la plateforme. | Ce rôle est normalement tenu par des personnes ayant aussi le rôle Développeur. |
| Développeur | Configuration des échanges et des queues (y compris les boîtes mortes), ainsi que leur sécurité applicative. <br/> Développement des producteurs et des consommateurs. | | 

Les équipes de projets sont chargées des deux derniers rôles ci-dessus. 

## Recommandations

### a) Responsabiliser les équipes de développeurs

Partager ces recommandations avec les équipes de développeurs.
Sensibiliser celles-ci aux particularités et aux difficultés de la programmation par messagerie.

En particulier, l'équipe Support messagerie veille à ne pas faire elle-même le développement
des producteurs et consommateurs,
afin de permettre aux équipes de projets de bien s'approprier le sujet avec toutes ses spécificités. 

### b) Définir le propriétaire de RabbitMQ

Définir quelle équipe ou quelles personnes assument le rôle de propriétaire de RabbitMQ
(voir tableau ci-dessus).
