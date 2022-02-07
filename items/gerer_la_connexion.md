# Gérer la connexion à RabbitMQ

## Le problème

Comment éviter des erreurs techniques (engorgements, lenteurs, consommation excessive de mémoire, etc.)
liées au mode de connexion à RabbitMQ ?

## Discussion

La production et la consommation de messages par une application se fait via une connexion et des canaux
(channels) RabbitMQ.
Sur une connexion, on peut ouvrir un ou plusieurs canaux, chaque canal pouvant être vu comme une session
ou comme un fil d'exécution.
Techniquement, les appels pour produire ou consommer des messages se font sur le canal, non sur la connexion.
Pour éviter des disfonctionnements techiques, il importe de gérer judicieusement connexions et canaux.

### Le producteur consommateur

Une application donnée peut être uniquement productrice de messages ou uniquement consommatrice. Cependant,
très fréquemment l'application tient les deux rôles ;
par exemple, elle peut être  productrice des messages d'origine et consommatrice des éventuels messages de
retour ou des messages émis par le consommateur.
Cette situation justifie la recommandation ci-dessous liée à la séparation entre production et consommation.

### Thread safe

Les canaux RabbitMQ doivent être considérés comme non thread-safe.
L'application (surtout si elle est productrice) doit donc éviter que deux threads utilisent simultanément
un même canal pour produire chacune leur message ;
ce genre de situation peut arriver si par exemple l'application est une application Web servie par une
servlet Java.
Inversement, si l'application est un batch qui cycle sur les messages à envoyer, la séquentialité garantit
un accès thread-sage au canal.

## Recommandations

Source des citations ci-dessous : 
[RabbitMQ Best Practices](https://www.cloudamqp.com/blog/part1-rabbitmq-best-practice.html)
de Lovissa Johansson.

### a) Réutiliser les connexions et les canaux

> Make sure you don’t open and close connections and channels repeatedly - doing so gives you a higher
> latency, as more TCP packages have to be sent and received.

### b) Ne pas laisser plusieurs threads utiliser un canal 

> Make sure that you don’t share channels between threads as most clients don’t make channels thread-safe
> (it would have a serious negative effect on the performance impact).

### c) Ne pas laisser un producteur et un consommateur utiliser la même connexion

> Separate the connections for publishers and consumers to achieve high throughput. RabbitMQ can apply
> back pressure on the TCP connection when the publisher is sending too many messages for the server
> to handle. If you consume on the same TCP connection, the server might not receive the message
> acknowledgments from the client, thus effecting the consume performance. With a lower consume speed,
> the server will be overwhelmed.

(voir la discussion plus haut)
