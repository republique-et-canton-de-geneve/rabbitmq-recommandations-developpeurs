# Prévoir les évolutions du contenu des messages

## Le problème

Comment faire, alors que le système de messagerie est déjà en production, pour absorber les changements
inévitables apportés au cours du temps au contenu des messages ?

## Discussion

### Conduite du changement

Une fois que les systèmes s'échangent des messages en production, il est assez facile d'ajouter, voire
de supprimer, certains messages, échanges ou queues.
Il est en revanche plus délicat de modifier certains messages.
Par exemple : le consommateur impose au producteur l'ajout d'un nouveau champ obligatoire, ou bien un
message doit être scindé en plusieurs messages.

La séquence recommandée pour une telle évolution est la suivante :
1. Les systèmes producteurs et consommateurs utilisent la version V1 des messages
2. Les systèmes consommateurs mettent des consommateurs V2 à disposition, en plus de leurs consommateurs V1
3. Les systèmes producteurs passent, chacun à leur rythme, à la version V2
4. Les systèmes producteurs utilisent tous la version V2 des messages
5. Les systèmes consommateurs retirent du service leurs consommateurs V1

On a donc un chevauchement temporaire V1 + V2 qui assure que la production n'est jamais interrompue.
La coordination entre équipes est simplifiée.
A contrario, le cas où tous les consommateurs et tous les producteurs peuvent s'entendre pour mettre
simultanément en recette, puis en production, leurs systèmes modifiés, conservant donc à tout moment une
seule version des contrats, doit être considéré comme peu représentatif et certainement comme risqué.

Les changements fonctionnels se traduisent par des changements dans les messages uniquement.
Il n'est habituellement pas utile de passer par des changements dans la définition des échanges, clefs
de routage et queues.

### Typologie des changements

Exemples de changements avec rupture de contrat :
- ajout d'une donnée obligatoire
- donnée existante facultative devenant obligatoire

Exemples de changements sans rupture de contrat :
- ajout d'une donnée facultative
- suppression d'une donnée
- donnée existante obligatoire devenant facultative

## Recommandations

### a) Négocier les changements avec les consommateurs

L'équipe (ou les équipes) consommatrices, à l'origine des modifications dans les données échangées, discute
de ces modifications avec les équipes productrices, afin que la conduite du changement s'effectue sans heurts
ni blocages.

### b) Versionner les types de messages

Exemple (cas d'un message JSON) :
```
application/new-document-v1.0+json  // version initiale
application/new-document-v2.0+json  // nouvelle version
```

Cette valeur est habituellement fournie dans la propriété AMQP `content-type` du message, cf.
[Échanger des métadonnées sur chaque message](./echanger_des_metadonnees.md).

De préférence, mais pas obligatoirement, on adoptera la nomenclature suivante :
- changement mineur de version (par exemple de 1.0 à 1.1) : sans rupture de contrat (voir typologie ci-dessus)
- changement majeur de version (par exemple de 1.0 à 2.0) : avec rupture de contrat (idem)

### c) Savoir consommer simultanément plusieurs versions d'un message

Le système consommateur doit être modifié pour être prévu pour traiter tant un message au format V1 qu'un
message au format V2.
Une fois que tous les producteurs sont passés au format V2, le consommateur peut supprimer le traitement
des messages au format V1.

À l´État, il faut prévoir 2 à 3 ans de recouvrement lors d'un changement majeur ; pour un changement mineur,
prévoir quelques mois.
Aussi le producteur devra s'attendre à devoir consommer simultanément plusieurs - plus de deux - versions
du message.
