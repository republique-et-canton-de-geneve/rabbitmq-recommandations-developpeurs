# RabbitMQ : recommandations pour les développeurs

Ce projet définit les recommandations élaborées à l'État de Genève
pour ses développeurs d'applications productices et consommatrices de messages
[RabbitMQ](https://www.rabbitmq.com).

(Être au clair sur le contenu des messages)[./items/etre_au_clair_sur_le_contenu_des_messages.md].

| Recommandation | Catégorie | Producteurs | Consommateurs |
|----------------|-----------|-------------|---------------|
| [Être au clair sur le contenu des messages](items/a1_etre_au_clair_sur_le_contenu_des_messages.md) | Communication | X | X |
| Prendre en main RabbitMQ | Messages | X | X |
| Utiliser RabbitMQ à l'État | Messages | X | X |
| Échanger de petits messages lisibles | Messages | X | X |
| Échanger des métadonnées sur chaque message | Messages | X | X |
| Gérer les erreurs de traitement | Messages | X | X |
| Ne pas perdre de messages (acquittements) | Messages | X | X |
| Gérer l'indisponibilité de RabbitMQ | Gestion opérationnelle | X | X |
| Savoir réémettre un message | Gestion opérationnelle | X | |
| Pouvoir consommer plusieurs fois le même message (idempotence) | Gestion opérationnelle | | X |
| Tracer les messages | Gestion opérationnelle | X | X |
| Respecter la bande passante allouée | Gestion de la capacité de RabbitMQ | X | X |
| Utiliser un pool de connexions à RabbitMQ | Gestion de la capacité de RabbitMQ | X | X |
| Gérer la connexion à RabbitMQ | Gestion de la capacité de RabbitMQ | X | X |
| Contrôler les envois de message en fonction de leur importance | Suivi et surveillance | X | X |
