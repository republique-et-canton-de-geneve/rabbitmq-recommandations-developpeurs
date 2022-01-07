# Utiliser RabbitMQ à l'État

## Le problème

Comment envoyer des messages à RabbitMQ ou en consommer, compte tenant des contraintes de sécurité
imposées par l'État de Genève ?

## Discussion

Pour envoyer ou recevoir des messages dans vos applications, il faut exécuter des séquences d'appels
bien définies.
Nous allons vous présenter comment procéder ici, avec des exemples de codes en Java.
Vous pouvez trouver une documentation plus complète avec des exemples dans d'autres langages sur la
page des
[didacticiels RabbitMQ](https://www.rabbitmq.com/getstarted.html).

## Recommandations

### a) Comment se connecter à RabbitMQ avec utilisateur local

La fabrique de connexions (connection factory) va permettre de se connecter à un virtual host de RabbitMQ dans
le but d'envoyer ou de recevoir des messages.
Pour initialiser la fabrique de connexions, il faut indiquer les informations suivantes :

| Paramètre | Explication |
|-----------|-------------|
| hostname | le nom de la VM hébergeant RabbitMQ |
| port | le port sur lequel écoute RabbitMQ |
| utilisateur + mot de passe, ou jeton UAA | les identifiants permettant de s'authentifier auprès de RabbitMQ |
| vhost | le virtual host (= région) RabbitMQ auquel se connecter |
| contexte SSL | dans le cas d'une connexion SSL, le contexte SSL qui permet de sécuriser la connexion |

Exemple d'initialisation de la fabrique de connexions en Java avec authentification avec utilisateur et mot
de passe :

```
// Création du contexte SSL. Nécessite un fichier jks avec le certificat GE-APP, ici ge-app.jks

KeyStore tks = KeyStore.getInstance("JKS");
InputStream resourceAsStream = Sender.class.getClassLoader().getResourceAsStream("./ge-app.jks");
tks.load(resourceAsStream, null);

TrustManagerFactory tmf = TrustManagerFactory.getInstance("SunX509");
tmf.init(tks);

SSLContext c = SSLContext.getInstance("TLSv1.2");
c.init(null, tmf.getTrustManagers(), null);


// Initialisation de la fabrique de connexions

ConnectionFactory factory = new ConnectionFactory();
factory.setHost("nom de machine");
factory.setPort(5672);
factory.setUsername("foo");
factory.setPassword("bar");
factory.setVirtualHost("simple_vhost");
factory.useSslProtocol(c);
```

### b) Comment se connecter à RabbitMQ avec un utilisateur technique Gina

L'authentification via UAA se fait de la façon suivante :
1. Authentification avec utilisateur technique Gina auprès du serveur UAA et récupération d'un jeton (token).
2. Utilisation du jeton obtenu du serveur UAA pour s'authentifier auprès de RabbitMQ.

#### Récupération du jeton UAA

Pour s'authentifier auprès du serveur UAA et récupérer le token, il faudra fournir les informations suivantes :

| Paramètre | Explication |
|-----------|-------------|
| URL OAUTH UAA | l'URL du serveur UAA qui permet d'obtenir le jeton. Exemple : https://<URL RabbitMQ UAA>/oauth/token |
| Client ID	| identifie l'application auprès de laquelle l'authentification va avoir lieu |
| Client ID Secret | le secret (= mot de passe) associé au client ID |
| grant_type | mettre la valeur `password` (sic) |
| username | le compte AD de l'utilisateur |
| password | le mot de passe AD de l'utilisateur |
| response_type | le type de réponse attendue lors de l'appel au serveur UAA ; mettre la valeur `token` |

#### Authentification auprès de RabbitMQ

C'est très simple : il faut utiliser le jeton récupéré de l'UAA comme mot de passe lors de l'initialisation de la
fabrique de connexions.

Exemple :

```
public class Producer {

    private final static String TOKEN_URL = ".../oauth/token";
    private final static String RABBITMQ_URL = "...";
    private final static String EXCHANGE = "test-x";
    private final static String QUEUE_NAME = "test-q";

    public static void main(String[] args) throws IOException, KeyStoreException, CertificateException, NoSuchAlgorithmException, KeyManagementException {

        // Valeurs nécessaires à l'appel au serveur UAA
        String clientId = "...";      // voir la configuration de l'UAA, fichier uaa.yml
        String clientSecret = "...";  // idem
        String grantType = "password";
        String username = "...";      // utilisateur AD
        String password = "....";     // mot de passe AD
        String responseType = "token";

        // Paramètres pour l'accès à RabbitMQ
        String virtualHost = "...";
        int rabbitPort = 5672;
        String access_token = "";

        // Initialisation SSL
        try {
            TrustManager[] trustAllCerts = new TrustManager[]{
                    new X509TrustManager() {
                        public java.security.cert.X509Certificate[] getAcceptedIssuers() {
                            return new X509Certificate[0];
                        }

                        public void checkClientTrusted(
                                java.security.cert.X509Certificate[] certs, String authType) {
                        }

                        public void checkServerTrusted(
                                java.security.cert.X509Certificate[] certs, String authType) {
                        }
                    }
            };

            SSLContext sc = null;
            sc = SSLContext.getInstance("SSL");
            sc.init(null, trustAllCerts, new java.security.SecureRandom());
            HttpsURLConnection.setDefaultSSLSocketFactory(sc.getSocketFactory());

            // Définition des paramètres d'appel à l'UAA
            List<NameValuePair> urlParameters = new ArrayList();
            urlParameters.add(new BasicNameValuePair("client_id", clientId));
            urlParameters.add(new BasicNameValuePair("client_secret", clientSecret));
            urlParameters.add(new BasicNameValuePair("grant_type", grantType));
            urlParameters.add(new BasicNameValuePair("username", username));
            urlParameters.add(new BasicNameValuePair("password", password));
            urlParameters.add(new BasicNameValuePair("response_type", responseType));

            // Appel au serveur UAA pour obtention du jeton
            HttpPost post = new HttpPost(TOKEN_URL);
            post.setEntity(new UrlEncodedFormEntity(urlParameters));
            SSLConnectionSocketFactory factory = new SSLConnectionSocketFactory(sc, new NoopHostnameVerifier());
            CloseableHttpClient httpClient = HttpClients.custom().setSSLSocketFactory(factory).build();
            CloseableHttpResponse response = httpClient.execute(post);
            final JSONObject obj = new JSONObject(EntityUtils.toString(response.getEntity()));
            access_token = obj.getString("access_token");

            // Initialisation de la fabrique de connexions en utilisant le jeton UAA comme mot de passe
            ConnectionFactory connectionFactory = new ConnectionFactory();
            connectionFactory.setHost(RABBITMQ_URL);
            connectionFactory.setVirtualHost(virtualHost);
            connectionFactory.setPassword(access_token);
            connectionFactory.setPort(rabbitPort);
            connectionFactory.useSslProtocol(sc);

            // Utilisation de la fabrique de connexions pour publier un message dans RabbitMQ
            try (Connection connection = connectionFactory.newConnection();
                 Channel channel = connection.createChannel()) {
                String message = "Hello World!";
                channel.basicPublish(EXCHANGE, QUEUE_NAME, null, message.getBytes());
                System.out.println(" [x] Sent '" + message + "'");
            } catch (TimeoutException e) {
                e.printStackTrace();
            } catch (IOException e) {
                e.printStackTrace();
            } catch (Throwable e) {
                e.printStackTrace();
            }
        } catch (ClientProtocolException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### c) Comment obtenir une connexion et un canal (channel)

Une fois la fabrique de connexions initialisée, il faut alors obtenir une connexion et un canal avec lesquels
travailler.
Voici comment les obtenir en Java :

```
Connection connection = factory.newConnection();
Channel channel = connection.createChannel();
```

Important : il ne faut pas oublier de fermer le canal et la connexion une fois que l'on n'en a plus besoin,
pour éviter de maintenir en mémoire des ressources inutiles. Deux façon de procéder :

1. Fermeture explicite : dans un bloc de code qu'on est certain d'exécuter :

```
channel.close();
connection.close();
```

2. Comme les classes `Connection` et `Channel` implémentent l'interface `Closeable`, on peut utiliser un bloc
`try-catch` qui fermera le canal et la connexion automatiquement :

```
try (Connection connection = m_factory.newConnection();
     Channel channel = connection.createChannel()) {
    ...
}
```

### d) Comment envoyer un message dans une queue

Pour envoyer un message dans une queue, il faut trois choses :

- un message, bien sûr !
- une queue destinatrice du message
- un échange sur lequel envoyer le message.

1. Le message.
Il s'agit tout simplement d'un tableau de bytes. 
Par exemple en Java, si le message est une chaîne de caractères :

```
byte[] message = "Mon message".getBytes();
```

2. La queue de destination.
Il faut la déclarer au canal pour que l'on puisse l'utiliser comme destinatrice de messages :

```
channel.queueDeclare("MyQueue", true, false, false, null);
```

3. L'échange.
On l'utilise change pour envoyer le message en destination de la queue :

```
channel.basicPublish("MyExchange", "MyQueue", null, message.getBytes());
```

Un bloc de code Java `try-catch` complet :
```
try (Connection connection = m_factory.newConnection();
     Channel channel = connection.createChannel()) {
    String message = "Hello World!";
    channel.queueDeclare("MyQueue", true, false, false, null);
    channel.basicPublish("MyExchange", "MyQueue", null, message.getBytes());
}
```

#### e) Comment lire un message d'une queue

Pour lire un message d'une queue, il faut faire trois choses :

1. Déclarer la queue sur le canal.
Exemple en Java :

```
channel.queueDeclare("MyQueue", true, false, false, null);
```

2. Créer une fonction de rappel (callback) de livraison de messages :

```
DeliverCallback deliverCallback = (consumerTag, delivery) -> {
        String message = new String(delivery.getBody(), StandardCharsets.UTF_8);
        System.out.println("Received '" + message + "'");
};
```

3. Se mettre à l'écoute de l'arrivée d'un message par le biais de la fonction de rappel :

```
channel.basicConsume("MyQueue", true, deliverCallback, consumerTag -> {});
```

Important : la lecture se fait d'une façon asynchrone par exécution de la fonction de rappel lorsqu'un
message arrive.
Il ne faut donc pas exécuter ce code dans un bloc `try-catch` comme il est préconisé de le faire pour
l'envoi de messages.
En effet, cela fermerait le canal et la connexion immédiatement et donc aucun message ne serait reçu.
Il ne faut fermer le canal et la connexion qu'à partir du moment où l'on ne veut plus recevoir de
messages.
