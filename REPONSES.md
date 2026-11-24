# Réponses — TP Migrer StayBook vers Kubernetes avec Argo CD

**Nom(s) :**  
**Dépôt GitHub :** https://github.com/Sephoyo/staybook-gitops  


## Étape 1 — Docker Compose : le point de départ

### Question 1

_Dans ce `docker-compose.yml`, repérez tout ce que Compose fait **pour vous** et que vous n'aurez plus « gratuitement » dans Kubernetes : l'ordre de démarrage, le réseau, les volumes, les ports, les variables d'environnement, la construction des images. Pour chacun, notez le mot-clé Compose correspondant._

Réponse :

| Ce que Compose fait pour nous | Mot-clé Compose |
|---|---|
| Ordre de démarrage | `depends_on` (forme courte = « démarré ») |
| Attendre qu'un service soit **prêt** | `depends_on: … condition: service_healthy` + `healthcheck` |
| Réseau privé + DNS par nom de service | `networks: [tp]` (réseau `bridge`) — `DB_HOST: postgres-hotels` |
| Persistance des données | `volumes:` (volumes nommés `hotels-data`, `ollama-data`…) |
| Script SQL d'initialisation | `volumes: ./db/hotels-init.sql:/docker-entrypoint-initdb.d/…` (bind mount) |
| Exposer le site sur la machine | `ports: "8080:80"` |
| Variables d'environnement | `environment:` (+ `${EMBED_MODEL}` lu dans `.env`) |
| Mots de passe | `environment: POSTGRES_PASSWORD` (en clair) |
| Construction des images | `build: ./hotels-service` |
| Image toute faite | `image: postgres:16-alpine` |

Dans Kubernetes, rien de tout ça n'est implicite : il faut un Service pour le DNS, un PVC pour les données, un NodePort pour exposer, des images déjà construites et disponibles sur le nœud, etc.


### Question 2

_Pourquoi faut-il impérativement faire `docker compose down` avant de continuer ? (Deux raisons : l'une concerne un port, l'autre la mémoire.)_

Réponse :

1. **Le port** : le conteneur `front` de Compose publie `8080:80`. Or `kind-config.yaml` redirige aussi le port **8080** de la machine (vers le NodePort 30080 d'Argo CD). Deux processus ne peuvent pas écouter sur le même port : la création du cluster échoue avec `Bind for 0.0.0.0:8080 failed: port is already allocated`. C'est exactement l'erreur obtenue pendant le TP tant que `staybook-src-front-1` tournait.
2. **La mémoire** : les 9 conteneurs (surtout Ollama et son modèle, plus Qdrant et deux PostgreSQL) consomment plusieurs Go de RAM dans la VM Docker. Le cluster kind, Argo CD, puis StayBook *une deuxième fois* dans Kubernetes ont besoin de cette mémoire (Ollama demande à lui seul 2 Gi). Laisser Compose tourner, c'est faire tourner l'application en double et risquer des pods `Pending` ou `OOMKilled`.


## Étape 2 — Kubernetes + Argo CD : la migration

### Question 3

_En mode push, qui détient les droits d'administration sur le cluster ? Citez deux risques concrets que le modèle pull réduit._

Réponse :

En mode push, c'est **l'outil de CI** (GitLab CI, GitHub Actions, Jenkins…), via un kubeconfig ou un token de ServiceAccount stocké dans ses secrets, qui a les droits d'admin sur le cluster, et indirectement toute personne capable de modifier un pipeline.

Risques réduits par le modèle pull :
- **Fuite des identifiants** : un token admin stocké dans la CI (souvent un service externe, partagé entre beaucoup de projets et de personnes) peut fuiter via un log, un fork, une dépendance compromise. En pull, les droits d'écriture sur le cluster restent **dans** le cluster (le ServiceAccount d'Argo CD) ; l'extérieur n'a besoin que d'un accès en **lecture** au dépôt Git.
- **Exposition de l'API Kubernetes** : en push, l'API server doit être joignable depuis la CI (ouverture réseau vers l'extérieur). En pull, c'est le cluster qui sort vers Git ; aucune connexion entrante n'est nécessaire.
- (bonus) **Dérive non détectée** : en push, on déploie puis plus personne ne vérifie. En pull, l'agent compare en continu et signale/corrige toute modification manuelle.


### Question 4

_Un collègue modifie « à la main » le nombre de réplicas d'un déploiement avec `kubectl scale` sur un cluster géré par Argo CD. D'après le principe 4, que va-t-il se passer ? Est-ce souhaitable ?_

Réponse :

Argo CD compare en permanence l'état réel à l'état décrit dans Git : il détecte la différence et passe l'application en **OutOfSync**. Si `selfHeal: true` est activé, il **remet automatiquement** le nombre de réplicas de Git (quelques secondes après). Sinon, il signale seulement la dérive jusqu'au prochain sync.

C'est souhaitable : Git est la seule source de vérité, chaque changement doit être tracé (qui, quand, pourquoi) et relu. Une modification manuelle serait invisible et perdue au prochain déploiement. Si on veut vraiment faire varier les réplicas dynamiquement, on le fait proprement : changement dans Git, ou un HorizontalPodAutoscaler avec `ignoreDifferences` sur `spec.replicas`.


### Question 5

_Quel composant est le seul à parler à GitHub ? Lequel est le seul à créer des ressources dans le cluster ?_

Réponse :

- Seul **`argocd-repo-server`** parle à GitHub : il clone le dépôt, le met en cache et génère les manifests (YAML brut, Kustomize, Helm).
- Seul **`argocd-application-controller`** crée/modifie/supprime des ressources dans le cluster : il compare les manifests générés à l'état réel et applique les différences (sync, self-heal, prune).

Tableau de la partie 1 :

| Composant | Rôle | Deployment ou StatefulSet ? |
|---|---|---|
| argocd-server | API (gRPC/REST) + interface web, utilisée par la CLI et l'UI ; gère l'authentification | Deployment |
| argocd-repo-server | Clone les dépôts Git et génère les manifests | Deployment |
| argocd-application-controller | Compare Git ↔ cluster en continu et applique les changements | **StatefulSet** |
| argocd-redis | Cache (manifests générés, état des applications) | Deployment |
| argocd-dex-server | Fournisseur d'identité / SSO (OIDC, GitHub, LDAP…) | Deployment |
| argocd-applicationset-controller | Génère des Applications en série à partir d'un modèle (ApplicationSet) | Deployment |
| argocd-notifications-controller | Envoie des notifications (Slack, mail…) sur les événements de sync | Deployment |


### Question 6

_Le cluster est « vierge ». Quels namespaces existent déjà, et à quoi servent-ils ? Combien de pods tournent, et pourquoi n'y en a-t-il aucun dans `default` ?_

Réponse :

| Namespace | Rôle |
|---|---|
| `default` | Namespace utilisé quand on n'en précise pas ; vide au départ |
| `kube-system` | Composants du control plane et du système (API server, etcd, scheduler, controller-manager, kube-proxy, CoreDNS, kindnet) |
| `kube-public` | Lisible par tous, même non authentifié (ConfigMap `cluster-info`) |
| `kube-node-lease` | Objets Lease des nœuds (battement de cœur, détection de nœud mort) |
| `local-path-storage` | Propre à kind : provisioner qui crée les volumes (PVC) sur le disque du nœud |

**9 pods** tournent : 8 dans `kube-system` (2 × coredns, etcd, kindnet, kube-apiserver, kube-controller-manager, kube-proxy, kube-scheduler) + 1 `local-path-provisioner`.

Aucun dans `default` parce que ce sont tous des composants d'infrastructure, rangés dans leurs propres namespaces. `default` est réservé à nos charges applicatives, et on n'a encore rien déployé.


### Question 7

_Pourquoi le script tague les images `v1` plutôt que `latest` ? Cherchez la règle de Kubernetes sur `imagePullPolicy` quand le tag est `latest` : que se passerait-il au démarrage d'un pod ?_

Réponse :

Règle Kubernetes : si `imagePullPolicy` n'est pas précisé et que le tag est **`latest`** (ou absent), la politique par défaut est **`Always`**. Pour tout autre tag, c'est `IfNotPresent`.

Avec `latest`, le kubelet tenterait donc de **télécharger** `staybook-front:latest` depuis Docker Hub à chaque démarrage de pod, en ignorant l'image chargée par `kind load`. L'image n'existe sur aucun registre : le pod resterait en `ErrImagePull` / `ImagePullBackOff`.

Avec `v1`, la politique est `IfNotPresent` : l'image déjà présente sur le nœud est utilisée. En plus, un tag explicite rend le déploiement **reproductible et traçable** : `latest` ne dit pas quelle version tourne, et changer d'image ne modifierait pas le manifest, donc Git ne verrait aucun changement.


### Question 8

_Où sont physiquement les images maintenant ? Vérifiez avec `docker exec argocd-tp-control-plane crictl images | grep staybook`. Qu'est-ce que `crictl`, et pourquoi n'est-ce pas `docker images` ?_

Réponse :

Les images sont **dans le stockage de containerd, à l'intérieur du conteneur `argocd-tp-control-plane`** (le nœud kind), pas dans le Docker de la machine :

```
docker.io/library/staybook-front                  v1   6200105c1190c   21.8MB
docker.io/library/staybook-gateway                v1   af12751a74615   54.7MB
docker.io/library/staybook-hotels-service         v1   f32df538366a4   52MB
docker.io/library/staybook-rag-service            v1   731023ac8af28   52MB
docker.io/library/staybook-reservations-service   v1   93b79c3b5448b   52.5MB
```

`crictl` est l'outil en ligne de commande de la **CRI** (Container Runtime Interface), l'interface standard entre le kubelet et le runtime de conteneurs. Un nœud Kubernetes moderne n'utilise pas Docker mais **containerd** (dockershim a été retiré en 1.24). Dans le nœud kind, il n'y a pas de démon Docker : `docker images` ne verrait rien, `crictl` interroge containerd directement. (Depuis la machine, `docker images` montre l'autre copie, celle qui a servi à construire.)


### Question 9

_Combien d'objets vont être créés ? Parmi les types listés, trois ne sont pas des objets Kubernetes « standard ». Lesquels, et à quoi servent-ils ? (Indice : `CustomResourceDefinition`.)_

Réponse :

**60 objets** (Argo CD v3.5.3) :

```
3 ClusterRole            7 ConfigMap        8 Service
3 ClusterRoleBinding     6 Deployment       7 ServiceAccount
3 CustomResourceDefinition  7 NetworkPolicy  1 StatefulSet
1 Namespace              6 Role / 6 RoleBinding   2 Secret
```

Les 3 CustomResourceDefinition ajoutent à l'API Kubernetes trois nouveaux types propres à Argo CD :
- **`Application`** (`applications.argoproj.io`) : relie une source Git (dépôt, chemin, révision) à une destination (cluster, namespace), avec sa politique de sync.
- **`AppProject`** (`appprojects.argoproj.io`) : regroupe des Applications et fixe des limites (dépôts autorisés, clusters/namespaces cibles, types de ressources, RBAC).
- **`ApplicationSet`** (`applicationsets.argoproj.io`) : génère automatiquement plusieurs Applications à partir d'un modèle et d'un générateur (liste, clusters, dossiers Git…).


### Question 10

_Pourquoi est-il préférable de passer par Kustomize plutôt que de faire `kubectl apply -f install.yaml` puis `kubectl patch` sur le Service ? Pensez au jour où vous mettrez Argo CD à jour._

Réponse :

- Un `kubectl patch` est une **action manuelle, impérative et non tracée** : elle n'existe que dans le cluster. Au moment de la mise à jour, un nouveau `kubectl apply -f install.yaml` remet le Service en `ClusterIP` et `server.insecure` à sa valeur par défaut : il faut se souvenir de refaire les patchs, sinon l'interface devient inaccessible.
- Avec Kustomize, **le fichier officiel reste intact** et nos modifications sont des patchs séparés, **versionnés dans Git**. Mettre à jour Argo CD revient à changer une ligne (`v3.5.3` → nouvelle version dans l'URL) : les patchs sont réappliqués automatiquement, à chaque fois, de manière reproductible.
- C'est déclaratif (on peut prévisualiser avec `kubectl kustomize`, relire en revue de code) : la même logique GitOps que pour l'application.


### Question 11

_Vérifiez votre tableau de la partie 1 : quel composant est un StatefulSet ? Avez-vous les sept composants ?_

Réponse :

Oui, les sept composants sont là : **6 Deployments** (`argocd-server`, `argocd-repo-server`, `argocd-redis`, `argocd-dex-server`, `argocd-applicationset-controller`, `argocd-notifications-controller`) + **1 StatefulSet** : **`argocd-application-controller`**.

C'est un StatefulSet car le contrôleur peut être réparti (*sharding*) entre plusieurs réplicas : chaque réplica a une identité stable (`argocd-application-controller-0`, `-1`…) qui détermine quels clusters il gère. Un Deployment donnerait des noms aléatoires.

Les 3 CRD sont aussi présentes : `applications`, `applicationsets`, `appprojects` (`.argoproj.io`).


### Question 12

_Essayez `kubectl apply -k argocd-install` **sans** `--server-side` (c'est sans danger, tout existe déjà). Quelle erreur obtenez-vous, sur quel objet ? Cherchez la cause (`last-applied-configuration`)._

Réponse :

```
The CustomResourceDefinition "applicationsets.argoproj.io" is invalid:
metadata.annotations: Too long: may not be more than 262144 bytes
```

L'erreur concerne la CRD **`applicationsets.argoproj.io`**.

Cause : un `kubectl apply` classique (*client-side apply*) enregistre **une copie complète de l'objet** dans l'annotation `kubectl.kubernetes.io/last-applied-configuration`, pour calculer le diff au prochain apply. Or la CRD ApplicationSet (avec son schéma OpenAPI) est énorme, et la somme des annotations d'un objet est limitée à **256 Kio (262 144 octets)**.

Avec `--server-side`, c'est l'API server qui calcule la fusion et qui suit qui possède quel champ dans `metadata.managedFields`, sans cette annotation géante : plus de limite. (On voit aussi des warnings « missing the last-applied-configuration annotation » : les objets créés en server-side n'ont justement pas cette annotation.)


### Question 13

_Pourquoi la commande se termine-t-elle par `| base64 -d` ? Que verriez-vous sans ? Un Secret Kubernetes est-il chiffré ?_

Réponse :

Les valeurs du champ `data` d'un Secret sont **encodées en base64** (pour pouvoir stocker n'importe quelles données binaires dans du JSON/YAML). Sans `| base64 -d`, on verrait une chaîne du type `aXZ3M0V0...` au lieu du mot de passe lisible.

**Un Secret n'est pas chiffré** : base64 est un encodage, réversible par n'importe qui. Par défaut, les Secrets sont stockés en clair dans etcd ; toute personne autorisée à faire `get secret` dans le namespace peut les lire. La protection repose sur le **RBAC**, et éventuellement sur le chiffrement au repos d'etcd (`EncryptionConfiguration`, KMS), qu'il faut activer explicitement.


### Question 14

_Que signifient `--plaintext` et `--grpc-web` ? Que se passe-t-il sans `--plaintext` (essayez) ? Ces options seraient-elles acceptables en production ?_

Réponse :

- **`--plaintext`** : la CLI se connecte en HTTP simple, **sans TLS**. Nécessaire ici car on a configuré `server.insecure: "true"` : le serveur ne parle pas HTTPS.
- **`--grpc-web`** : la CLI encapsule ses appels gRPC dans des requêtes HTTP/1.1 (gRPC-Web) au lieu de HTTP/2 natif. Utile quand un proxy, un ingress ou ici la redirection de port de kind ne transmet pas correctement le HTTP/2.

Sans `--plaintext`, la CLI tente une poignée de main TLS sur un serveur qui répond en HTTP : la connexion échoue avec une erreur du type `tls: first record does not look like a TLS handshake`.

En production : **`--plaintext` non**. Mot de passe admin et token de session passeraient en clair sur le réseau. Il faut TLS, avec un vrai certificat (cert-manager, ou terminaison TLS sur l'ingress). `--grpc-web` est acceptable : c'est un mode de transport, pas un affaiblissement de la sécurité. On l'utilise souvent derrière un ingress ou un load balancer.


### Question 15

_Regardez `apps/staybook/00-secrets.yaml` : les mots de passe PostgreSQL sont en clair dans Git. Est-ce acceptable ici ? Et en entreprise ? Citez une solution._

Réponse :

- **Ici, oui** : c'est un TP, avec des mots de passe triviaux (`hotels`/`hotels`), une base jetable dans un cluster local, et un dépôt privé.
- **En entreprise, non** : tout ce qui est poussé dans Git y reste pour toujours (l'historique), est copié sur chaque poste, dans la CI, dans les forks… Révoquer demande de réécrire l'historique et de changer le mot de passe partout.

Solutions :
- **Sealed Secrets** (Bitnami) : on commite un `SealedSecret` chiffré avec la clé publique du cluster ; seul le contrôleur dans le cluster peut le déchiffrer.
- **External Secrets Operator** : Git ne contient qu'une référence ; la valeur vit dans un coffre (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault…) et l'opérateur crée le Secret.
- Autres : SOPS + age/KMS (fichiers chiffrés dans Git), plugin Argo CD Vault.


### Question 16

_Notez l'erreur. Pourquoi GitHub répond-il « Repository not found » et non « accès refusé » ?_

Réponse :

```
rpc error: code = Unknown desc = error testing repository connectivity: unable to ls-remote HEAD
on repository: failed to list refs: authentication required: Repository not found.
```

Le dépôt est **privé** et la requête est anonyme. GitHub répond volontairement **404 (not found)** et non 403 (forbidden) pour **ne pas révéler l'existence** du dépôt. Sinon, un attaquant pourrait deviner les noms de dépôts privés d'une organisation (projets internes, clients…) simplement en testant des URL. Pour un visiteur non autorisé, un dépôt privé et un dépôt inexistant sont indiscernables.


### Question 17

_Sous quelle forme Argo CD a-t-il stocké ces identifiants dans le cluster ? Trouvez l'objet avec `kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=repository` et listez ses clés (`-o jsonpath='{.data}'` puis décodez `url` et `username`, pas `password`)._

Réponse :

Sous la forme d'un **Secret Kubernetes ordinaire** dans le namespace `argocd`, reconnu par Argo CD grâce au label `argocd.argoproj.io/secret-type=repository` :

```
NAME              TYPE     DATA   AGE
repo-2842142950   Opaque   4      19h
```

Clés : `password`, `type`, `url`, `username`. Une fois décodées :

```
url      = https://github.com/Sephoyo/staybook-gitops.git
username = Sephoyo
type     = git
```

Le token est donc dans `password`, simplement **encodé en base64, pas chiffré** (cf. Q13). Conséquence : le déclarer via `argocd repo add` revient à créer ce Secret. On pourrait aussi le déclarer en YAML, mais il ne faut surtout pas le commiter en clair.


### Question 18

_Comparez trois façons de s'authentifier (token personnel, deploy key SSH, GitHub App) : portée, rotation, ce qui se passe quand la personne quitte l'entreprise. Laquelle recommanderiez-vous pour 200 dépôts ?_

Réponse :

| | Token personnel (PAT) | Deploy key SSH | GitHub App |
|---|---|---|---|
| **Portée** | Les dépôts auxquels **la personne** a accès (fine-grained : limitable à certains dépôts + permissions) | **Un seul dépôt** (une clé par dépôt), lecture seule ou lecture/écriture | Les dépôts où l'App est installée, avec des permissions fines (ex. Contents: read) |
| **Rotation** | Manuelle, expiration à gérer, à recopier dans Argo CD | Manuelle, clé par clé (200 clés = 200 rotations) | Automatique : Argo CD génère des tokens d'installation qui expirent au bout d'une heure ; seule la clé privée de l'App est à protéger |
| **Départ de la personne** | Le compte est désactivé → le token cesse de marcher → **tous les déploiements cassent** (ou pire, l'accès reste si le compte est oublié) | Non liée à une personne, survit au départ | Appartient à **l'organisation**, pas à une personne : aucun impact |
| **Traçabilité** | Les accès apparaissent au nom de la personne | Peu lisible | Identité dédiée « bot », journalisée |

Pour **200 dépôts** : la **GitHub App**. Une seule identité d'organisation, installée sur tous les dépôts (ou une sélection), en lecture seule, sans secret qui expire à faire tourner à la main. Argo CD la supporte nativement (`githubAppID`, `githubAppInstallationID`, `githubAppPrivateKey`) et un *credential template* (`argocd repocreds`) permet de couvrir tous les dépôts `https://github.com/mon-org/*` d'un coup. Les deploy keys ne passent pas à l'échelle (200 clés). Le PAT lie la production à un humain.


### Question 19

_Combien de ressources Argo CD compte-t-il déployer ? Retrouvez chacune dans les fichiers de `apps/staybook/`._

Réponse :

**16 ressources** :

| Fichier | Ressources |
|---|---|
| `00-secrets.yaml` | Secret `postgres-hotels`, Secret `postgres-reservations` (2) |
| `10-postgres-hotels.yaml` | ConfigMap `postgres-hotels-init`, Service `postgres-hotels`, StatefulSet `postgres-hotels` (3) |
| `11-postgres-reservations.yaml` | ConfigMap `postgres-reservations-init`, Service `postgres-reservations`, StatefulSet `postgres-reservations` (3) |
| `20-hotels-service.yaml` | Service + Deployment `hotels-service` (2) |
| `21-reservations-service.yaml` | Service + Deployment `reservations-service` (2) |
| `30-gateway.yaml` | Service + Deployment `gateway` (2) |
| `40-front.yaml` | Service (NodePort 30081) + Deployment `front` (2) |

Soit 2 Secrets, 2 ConfigMaps, 6 Services, 2 StatefulSets, 4 Deployments. Dans l'arbre de l'UI apparaissent aussi les objets *dérivés* (ReplicaSets, Pods, PVC créés par les `volumeClaimTemplates`, EndpointSlices), qu'Argo CD ne déploie pas lui-même : ils sont créés par les contrôleurs de Kubernetes.


### Question 20

_Les ressources ne sont pas déployées toutes en même temps. Ouvrez les manifests et trouvez l'annotation qui pilote cet ordre. Quel ordre a été choisi, et pourquoi ? À quoi cela correspondait-il dans `docker-compose.yml` ?_

Réponse :

L'annotation est **`argocd.argoproj.io/sync-wave`**. Argo CD applique les vagues dans l'ordre croissant et **attend que toutes les ressources d'une vague soient `Healthy`** avant de passer à la suivante :

| Vague | Ressources | Pourquoi |
|---|---|---|
| `0` | Secrets, ConfigMaps d'init, PostgreSQL hotels + reservations | Les données et identifiants d'abord : tout le reste en dépend |
| `1` | `hotels-service`, `reservations-service` | Les API métier ont besoin de leurs bases |
| `2` | `gateway` | Elle route vers hotels et reservations |
| `3` | `front` | Le site public, en dernier, quand l'API est prête |

Dans `docker-compose.yml`, c'était **`depends_on`**, et surtout `depends_on: … condition: service_healthy` avec un `healthcheck`. L'équivalent du `healthcheck` côté Kubernetes est la **`readinessProbe`** (`pg_isready`, `/health`), qui détermine quand une ressource est `Healthy` et donc quand la vague suivante démarre.


### Question 21

_« OutOfSync » et « Healthy » sont deux statuts indépendants. Donnez un exemple d'application `Synced` mais `Degraded`, et un exemple `OutOfSync` mais `Healthy`._

Réponse :

- **Sync** = le cluster correspond-il à Git ? **Health** = ce qui tourne fonctionne-t-il ?
- **`Synced` mais `Degraded`** : Git demande `image: staybook-front:v3`, qui n'existe pas. Argo CD a bien appliqué exactement ce que dit Git (Synced), mais le pod est en `ImagePullBackOff` et le Deployment n'atteint pas ses réplicas (Degraded). Autre exemple : une application qui plante au démarrage (`CrashLoopBackOff`) à cause d'une mauvaise variable d'environnement commitée.
- **`OutOfSync` mais `Healthy`** : quelqu'un fait `kubectl scale deploy front --replicas=3` (sans self-heal). Les 3 pods tournent parfaitement (Healthy), mais l'état ne correspond plus à Git (OutOfSync). Autre exemple : on vient de pousser `replicas: 2` et Argo CD ne l'a pas encore synchronisé ; l'ancienne version tourne très bien.


### Question 22

_Vous venez de faire un `kubectl apply`… n'est-ce pas contraire au GitOps ? Pourquoi ce dernier `apply` est-il inévitable, et comment réduire cette « amorce » à un seul objet pour tout un cluster ? (Cherchez « App of Apps ».)_

Réponse :

C'est une entorse, mais une entorse **d'amorçage** (*bootstrap*) : c'est l'objet `Application` qui dit à Argo CD quel dépôt surveiller. Tant qu'il n'existe pas, Argo CD ne sait pas que ce dépôt existe : il ne peut donc pas lire dans Git l'objet qui lui dirait de lire Git. C'est le problème de l'œuf et de la poule ; il faut au moins **un** objet créé à la main.

On réduit cette amorce à un seul objet avec le motif **App of Apps** : une Application « racine » (ex. `root`) pointe vers le dossier `argocd/` du dépôt, qui contient toutes les autres Applications (`staybook`, `concierge`, et même Argo CD lui-même). On applique **une seule fois** `kubectl apply -f root.yaml`. Ensuite, ajouter, modifier ou supprimer une application se fait par commit dans `argocd/`, et Argo CD gère aussi ses propres Applications. (Variante : un `ApplicationSet` avec un générateur de dossiers Git, qui crée une Application par dossier de `apps/`.)

Ça résout aussi le problème de la partie 8.2 : la modification de `syncPolicy` poussée dans Git serait prise en compte automatiquement, sans nouveau `kubectl apply`.


### Question 23

_Reprenez votre réponse à la question 1 et complétez la colonne de droite en vous appuyant sur les manifests fournis._

Réponse :

| Ce que faisait Compose | Mot-clé Compose | Dans les manifests Kubernetes |
|---|---|---|
| Ordre de démarrage | `depends_on` | Annotation `argocd.argoproj.io/sync-wave` (0 → 3) |
| Attendre qu'un service soit prêt | `healthcheck` + `condition: service_healthy` | `readinessProbe` (`pg_isready`, `httpGet /health`) + `initContainers` d'attente (`wait-for-ollama`, `wait-for-qdrant`) |
| Réseau et DNS entre services | `networks: [tp]` | Un **Service** (ClusterIP) par composant + DNS CoreDNS dans le namespace `staybook` (`http://hotels-service:3001`) |
| Persistance des données | `volumes:` (volumes nommés) | **PersistentVolumeClaim** : `volumeClaimTemplates` des StatefulSets (postgres, qdrant), PVC `ollama-models` |
| Script d'init SQL | bind mount `./db/*.sql` | **ConfigMap** `postgres-*-init` monté dans `/docker-entrypoint-initdb.d` |
| Exposer le site | `ports: "8080:80"` | Service **NodePort** `front` (30081) + `extraPortMappings` kind (30081 → localhost:8081) |
| Variables d'environnement | `environment:` / `.env` | `env:` dans les Deployments, `configMapKeyRef` vers le ConfigMap `concierge-config` |
| Mots de passe | `environment: POSTGRES_PASSWORD` | **Secret** `postgres-hotels` / `postgres-reservations` chargé via `envFrom.secretRef` |
| Image à utiliser | `build:` / `image:` | `image: staybook-front:v1` : image construite **à l'avance** (`build-images.sh`) et chargée dans le nœud (`kind load`) |


### Question 24

_Pourquoi les deux Applications pointent-elles vers le **même namespace** `staybook` ? Que se passerait-il si le concierge était dans un namespace `concierge` ? (Indice : regardez `RAG_URL` dans le manifest de la gateway.)_

Réponse :

La gateway appelle `RAG_URL: http://rag-service:3003`, et rag-service appelle `http://hotels-service:3001` et `http://reservations-service:3002`. Un **nom court** de Service n'est résolu par le DNS que **dans le même namespace** (le pod cherche `rag-service.<son-namespace>.svc.cluster.local`).

Si le concierge était dans un namespace `concierge`, la gateway (dans `staybook`) ne trouverait pas `rag-service` : erreur DNS, le chat du site serait cassé. Dans l'autre sens, rag-service ne trouverait pas `hotels-service` et ne pourrait pas indexer les hôtels. Il faudrait utiliser les noms complets (`http://rag-service.concierge.svc.cluster.local:3003`, `http://hotels-service.staybook.svc.cluster.local:3001`) et vérifier qu'aucune NetworkPolicy ne bloque le trafic entre namespaces.

Séparer en deux **Applications** permet de déployer ou retirer l'IA indépendamment, sans pour autant isoler le réseau.


### Question 25

_Ouvrez `apps/concierge/20-rag-service.yaml`. Que font les deux `initContainers` ? En quoi est-ce différent des sync waves de la question 20 ? Pourquoi avoir les deux ?_

Réponse :

Deux conteneurs `busybox` qui s'exécutent **l'un après l'autre, avant** le conteneur principal, et bouclent jusqu'à ce que leur dépendance réponde :
- `wait-for-ollama` : `wget http://ollama:11434/api/tags` toutes les 3 s ;
- `wait-for-qdrant` : `wget http://qdrant:6333/readyz` toutes les 3 s.

Tant qu'ils ne sont pas terminés, le pod reste en `Init:0/2`, puis `Init:1/2`. C'est l'équivalent du `depends_on` de Compose, puisque Kubernetes n'en a pas.

Différences avec les sync waves :

| Sync waves | initContainers |
|---|---|
| Gérées par **Argo CD**, au moment d'un **sync** | Gérées par **Kubernetes** (kubelet), à **chaque démarrage du pod** |
| Ordonnent des ressources **à l'intérieur d'une seule Application** | Vérifient une dépendance **réelle**, quelle que soit son origine (autre Application, autre outil) |
| N'agissent plus ensuite | Rejouées si le pod redémarre, est replanifié ou si le nœud reboote |

On a besoin des deux : les waves rendent le déploiement initial propre et lisible. Mais elles ne servent à rien quand un pod redémarre plus tard, par exemple après le redémarrage du PC, quand tous les pods repartent en même temps. Elles ne couvrent pas non plus les dépendances entre Applications : rag-service dépend de hotels/reservations, qui sont dans l'Application `staybook`. Les initContainers garantissent l'ordre **à l'exécution**, à tout moment.


### Question 26

_Ouvrez `apps/concierge/11-ollama.yaml`. Où sont stockés les modèles téléchargés ? Supprimez le pod Ollama (`kubectl -n staybook delete pod -l app=ollama`) et vérifiez avec `kubectl -n staybook exec deploy/ollama -- ollama list` que les modèles sont toujours là. Qu'est-ce qui les a conservés ? Que signifie `strategy: Recreate` ?_

Réponse :

Les modèles sont stockés dans `/root/.ollama`, monté depuis le **PersistentVolumeClaim `ollama-models`** (5 Gi). Dans kind, le provisioner `local-path` crée le volume correspondant (PersistentVolume) sur le disque du nœud (un dossier dans le conteneur `argocd-tp-control-plane`).

Après suppression du pod, le Deployment en recrée un nouveau, qui monte **le même PVC** : `ollama list` montre toujours `nomic-embed-text` et `llama3.2:1b`. C'est le **PersistentVolume** qui les a conservés : son cycle de vie est indépendant de celui du pod. Sans lui (stockage éphémère du conteneur), il faudrait retélécharger environ 1,6 Go à chaque redémarrage.

**`strategy: Recreate`** : lors d'une mise à jour, Kubernetes **supprime d'abord l'ancien pod, puis crée le nouveau** (au lieu du `RollingUpdate` par défaut, qui démarre le nouveau avant de couper l'ancien). Nécessaire ici car le volume est en **`ReadWriteOnce`** : avec un RollingUpdate, le nouveau pod ne pourrait pas monter le volume encore utilisé par l'ancien. Contrepartie : une courte coupure du service pendant la mise à jour.


### Question 27

_Toujours dans ce fichier, lisez le bloc `resources`. Que se passe-t-il si Ollama dépasse la limite mémoire ? Et si le nœud n'a pas les 2 Gi demandés ?_

Réponse :

```yaml
resources:
  requests: { cpu: "1", memory: 2Gi }
  limits:   { memory: 6Gi }
```

- **Dépassement de la limite (6 Gi)** : le noyau tue le processus. Le conteneur est **`OOMKilled`** (code de sortie 137) et le kubelet le redémarre. Si ça se répète : `CrashLoopBackOff`, avec un délai de plus en plus long entre les tentatives. La mémoire n'est pas « compressible » : contrairement au CPU (simplement ralenti), on ne peut pas juste freiner un processus qui en consomme trop.
- **Nœud sans les 2 Gi demandés** : la `request` sert au **scheduler**. Si aucun nœud n'a 2 Gi de mémoire allouable (et 1 CPU) encore disponibles, le pod n'est placé nulle part. Il reste **`Pending`**, avec un événement `FailedScheduling … Insufficient memory`. Il ne démarre pas tant que des ressources ne se libèrent pas. Ici, la mémoire du nœud kind est celle de la VM Docker Desktop, d'où l'importance du `docker compose down` (Q2).


## Étape 3 — GitOps : Git pilote le cluster

### Question 28

_Sans `--refresh`, au bout de combien de temps Argo CD aurait-il vu le commit ? D'où vient cette valeur (ConfigMap `argocd-cm`, clé `timeout.reconciliation`) ? Quel mécanisme permettrait de réagir en quelques secondes sans interroger GitHub en boucle, et pourquoi ne peut-on pas l'utiliser ici ?_

Réponse :

Au bout de **3 minutes au maximum** : Argo CD interroge Git toutes les **120 s** (`timeout.reconciliation`), plus un décalage aléatoire de 0 à 60 s (`timeout.reconciliation.jitter`) pour ne pas interroger tous les dépôts en même temps.

Cette valeur se règle dans la ConfigMap **`argocd-cm`**, clé `timeout.reconciliation`. Dans notre installation, la clé n'est pas définie (`kubectl get cm argocd-cm -n argocd -o jsonpath='{.data}'` ne la montre pas) : c'est donc la valeur par défaut de 120 s qui s'applique.

Le mécanisme pour réagir en quelques secondes est le **webhook** : GitHub envoie une requête HTTP POST à Argo CD (`https://<argocd>/api/webhook`) à chaque `push`, et Argo CD rafraîchit immédiatement l'application concernée.

Impossible ici : notre Argo CD tourne sur `localhost:8080`, **inaccessible depuis Internet**. Les serveurs de GitHub ne peuvent pas le joindre. Il faudrait l'exposer publiquement, via une IP publique, un ingress avec TLS, ou un tunnel type ngrok / smee.io.


### Question 29

_Que s'est-il passé, et en combien de temps ? Quel composant a agi ? Retrouvez sa trace : `kubectl -n argocd logs sts/argocd-application-controller | grep staybook | tail`. Essayez plus fort : `kubectl -n staybook delete deploy gateway`. Le site est-il resté accessible ?_

Réponse :

Le Deployment `front` passe à 3 réplicas, puis **revient presque aussitôt à 1** (en quelques secondes). Dans le terminal 1 : `READY 1/1 → 1/3 → 3/3`, puis retour à `1/1`, et les 2 pods en trop sont supprimés.

C'est l'**`argocd-application-controller`** qui a agi. Il surveille les ressources du cluster en temps réel (watch), pas seulement toutes les 3 minutes. Il voit la modification, compare à Git (`replicas: 1`), constate la dérive et, comme `selfHeal: true` est activé, relance un sync. Trace dans les logs du controller (extrait) :

```
"msg":"Initiated automated sync to 'c5d0e3c2e747aa5083fd63ffb339315807cf2815'","reason":"OperationStarted","time":"2026-11-24T14:01:19Z"
"msg":"Partial sync operation to c5d0e3c2e747aa5083fd63ffb339315807cf2815 succeeded","reason":"OperationCompleted","time":"2026-11-24T14:01:19Z"
```

Le sync automatique est lancé et terminé dans la même seconde. Argo CD ne refait qu'un **sync partiel** : il ne réapplique que la ressource qui a dérivé, vers le commit courant.

Avec `kubectl -n staybook delete deploy gateway`, le controller **recrée le Deployment** en quelques secondes (`gateway 1/1`, `AGE 18s` juste après). **Le site est resté accessible** : la page est servie par le pod `front`, qui n'a pas été touché. Seuls les appels à l'API peuvent échouer pendant les quelques secondes où aucun pod gateway n'est `Ready`, et le selfHeal referme cette fenêtre presque aussitôt. Sans Argo CD, le gateway serait resté supprimé jusqu'à ce que quelqu'un s'en aperçoive.


### Question 30

_Décrivez précisément la chaîne d'événements entre votre `git push` et le nouveau titre dans le navigateur. Qui a fait quoi ? Où est passé l'ancien pod ? (`kubectl -n staybook get rs`)_

Réponse :

1. **Moi (CI à la main)** : `docker build -t staybook-front:v2` puis `kind load docker-image` : l'image v2 est présente dans le containerd du nœud.
2. **Moi** : commit `image: staybook-front:v2` dans `apps/staybook/40-front.yaml`, puis `git push` vers GitHub.
3. **argocd-repo-server** : au prochain refresh (≤ 3 min, ou immédiat avec `--refresh`), il fait un `git fetch` du dépôt, voit le nouveau commit et régénère les manifests.
4. **argocd-application-controller** : compare ces manifests au cluster et voit que l'image du Deployment `front` diffère : `OutOfSync`. Auto-sync activé : il applique le nouveau Deployment.
5. **Contrôleur Deployment de Kubernetes** (kube-controller-manager) : le template du pod a changé, il crée un **nouveau ReplicaSet** (avec un nouveau hash) et lance un *rolling update*.
6. **Scheduler + kubelet** : le nouveau pod est placé sur le nœud. Le kubelet démarre le conteneur avec l'image v2 déjà présente (`IfNotPresent`).
7. Quand la **readinessProbe** (`GET /`) réussit, le pod est ajouté aux endpoints du Service `front` ; l'ancien ReplicaSet est réduit à 0 et **l'ancien pod est supprimé**.
8. **Navigateur** : `localhost:8081` → kind → NodePort 30081 → Service `front` → nouveau pod : titre « StayBook v2 ».

`kubectl -n staybook get rs` montre **deux ReplicaSets pour front** : le nouveau (`DESIRED 1`) et l'ancien (`DESIRED 0`). L'ancien pod n'existe plus, mais son ReplicaSet est **conservé** (historique, `revisionHistoryLimit: 10` par défaut), ce qui permettrait un `kubectl rollout undo`.


### Question 31

_Dans une entreprise, qui construirait et chargerait l'image à votre place, et où ? Quelle partie de ce que vous venez de faire relève de la **CI**, et quelle partie du **CD** ?_

Réponse :

Un **pipeline de CI** (GitHub Actions, GitLab CI, Jenkins), déclenché par un push ou une merge request sur le **dépôt du code source** (`staybook-src`), pas sur le dépôt GitOps. Il lance les tests, construit l'image, la scanne, puis la pousse dans un **registre** (GHCR, Docker Hub, Harbor, ECR…) avec un tag unique (version ou SHA du commit). Les nœuds du cluster tirent ensuite l'image depuis ce registre (au lieu de `kind load`).

| CI (intégration continue) | CD (déploiement continu) |
|---|---|
| Modifier `index.html` dans le code source | Modifier `image: staybook-front:v2` dans le dépôt GitOps, commit, push (souvent automatisé par la CI via une merge request, ou par Argo CD Image Updater) |
| `docker build -t staybook-front:v2` | Argo CD détecte le commit, synchronise et applique le Deployment |
| `kind load docker-image` (≈ `docker push` vers un registre) | Kubernetes fait le rolling update |

La CI produit un **artefact** (l'image) ; le CD décide **quelle version tourne où**. Avec GitOps, la frontière est nette : la CI n'a jamais accès au cluster, elle écrit au mieux dans le dépôt GitOps.


### Question 32

_Qu'est-il arrivé au ConfigMap ? Que se serait-il passé avec `prune: false` ? Pourquoi `prune` est-il désactivé par défaut dans Argo CD ?_

Réponse :

Avec `prune: true`, après le `git rm` et le push, Argo CD a constaté qu'une ressource qu'il gérait n'était plus dans Git : il a **supprimé le ConfigMap du cluster**. Le cluster reflète exactement Git, y compris les suppressions.

Avec `prune: false`, le ConfigMap serait **resté dans le cluster**. L'application serait passée `OutOfSync`, la ressource marquée « requires pruning » (icône de suppression dans l'UI). Il aurait fallu un sync manuel avec l'option *Prune* (`argocd app sync staybook --prune`) pour la supprimer.

Prune est désactivé par défaut **par sécurité** : la suppression est l'action la plus destructrice. Un fichier supprimé par erreur, un mauvais `path`, une branche erronée, une erreur de rendu Kustomize/Helm qui produit une liste vide… et Argo CD effacerait des ressources en production, **y compris des PVC et donc des données** (ex. les bases PostgreSQL). Argo CD préfère qu'on active explicitement ce comportement, application par application (et on peut protéger certaines ressources avec l'annotation `argocd.argoproj.io/sync-options: Prune=false`).


### Question 33

_Après le rollback (a), l'auto-sync est-il toujours actif ? Pourquoi Argo CD fait-il cela ? Laquelle des deux approches est « GitOps-compatible » ? Le site est-il resté disponible pendant le mauvais déploiement, et grâce à quoi ?_

Réponse :

**Non, l'auto-sync n'est plus actif.** Argo CD refuse le rollback tant que la synchronisation automatique est activée :

```
$ argocd app rollback staybook 5
rpc error: code = FailedPrecondition desc = rollback cannot be initiated when auto-sync is enabled
```

 Il faut d'abord désactiver l'auto-sync (`argocd app set staybook --sync-policy none`). Après le rollback, l'application reste en sync **manuel**, et `OutOfSync` puisque Git demande toujours v3.

Raison : un rollback remet le cluster dans un état **différent de Git**. Si l'auto-sync restait actif, Argo CD resynchroniserait immédiatement sur le HEAD de Git, c'est-à-dire sur la v3 cassée, et annulerait le rollback. Argo CD « débranche » donc l'automatisme pour que le retour arrière tienne, le temps de corriger Git.

L'approche **GitOps-compatible est (b), `git revert`** : on corrige la source de vérité. Le retour arrière est un commit, tracé, relu, et le cluster reconverge tout seul. (a) est un **bouton d'urgence** qui crée volontairement une divergence entre Git et le cluster : utile en incident pour aller vite, mais il faut ensuite faire (b) et réactiver l'auto-sync (ici en réappliquant `argocd/staybook-application.yaml`, cf. Q22).

**Oui, le site est resté disponible**, grâce au **rolling update** du Deployment. Le nouveau pod `staybook-front:v3` n'a jamais démarré (`ErrImagePull` / `ImagePullBackOff` : l'image n'existe ni sur le nœud ni sur Docker Hub), donc il n'est jamais devenu **Ready**. Avec 1 réplica, `maxUnavailable: 25%` arrondi à 0 : Kubernetes ne supprime pas l'ancien pod tant que le nouveau n'est pas prêt. L'ancien pod v2 a continué de servir le trafic. Côté Argo CD, l'application était `Synced` mais `Progressing`, puis `Degraded` (dépassement de `progressDeadlineSeconds`).


### Question 34

_Reliez chaque entrée de `argocd app history staybook` à un commit de `git log --oneline`. Face à un auditeur qui demande « qui a déployé quoi, quand », que répondez-vous, en combien de commandes ?_

Réponse :

Chaque ligne de `argocd app history staybook` (ID, date de sync, révision) contient le **SHA du commit Git** déployé. Ce SHA correspond à une ligne de `git log --oneline` : les 7 premiers caractères sont identiques. Correspondance observée :

| ID Argo CD | Commit | Message |
|---|---|---|
| 0 | `7231e31` | feat: StayBook manifests + Argo CD install (v1) |
| 1 | `cac301b` | docs: réponses aux questions 1 à 34 (aucun changement dans `apps/`) |
| 2 | `41be646` | feat: hotels-service passe à 2 réplicas |
| 3 | `d4ece1c` | feat: front v2 |
| 4 | `3d162be` | test: ConfigMap pour le prune |
| 5 | `e691d18` | test: suppression du ConfigMap de test |
| 6 | `d296892` | feat: front v3 (image inexistante) |
| 7 | `e691d18` | **rollback (a)** `argocd app rollback staybook 5` : redéploie une **ancienne** révision, sans nouveau commit |
| 8 | `d296892` | auto-sync réactivé (`kubectl apply` de l'Application) **avant** que le revert soit lu : Argo CD resynchronise HEAD, donc la v3 cassée |
| 9 | `ad3db13` | **revert (b)** `Revert "feat: front v3 (image inexistante)"` : retour à v2 par Git |

Les entrées 7 et 8 illustrent la question 33 : le rollback n'existe que dans Argo CD. Dès que l'auto-sync est réactivé, le cluster revient à ce que dit Git. Seul le `git revert` (entrée 9) corrige durablement.

Le commit `c5d0e3c` (auto-sync) n'a pas d'entrée : il ne modifie que `argocd/`, dossier que l'Application `staybook` (path `apps/staybook`) ne surveille pas. Le rollback (a) apparaît comme une entrée supplémentaire qui redéploie une **ancienne** révision. Le `git revert` apparaît comme un nouveau commit « Revert … ».

Réponse à l'auditeur : **tout est dans Git**, en **une commande** (deux pour croiser avec les dates de déploiement effectives) :

```bash
git log --format='%h %an %ad %s' --date=iso -- apps/
argocd app history staybook
```

- *Qui* : l'auteur du commit (et, avec des branches protégées + merge requests, la personne qui a relu et approuvé) ;
- *Quoi* : le diff du commit (`git show <sha>`) ;
- *Quand* : la date du commit, et la date de sync dans `argocd app history`.

Personne ne déploie autrement qu'en commitant (les droits d'écriture sur le cluster sont réservés à Argo CD), donc l'historique Git **est** le journal d'audit des déploiements. Exception notable : un `argocd app rollback` manuel, qui n'apparaît que dans l'historique Argo CD : c'est une raison de plus de lui préférer `git revert`.


## Preuves

- Sortie de `argocd app history staybook` :

```
SOURCE  https://github.com/Sephoyo/staybook-gitops.git
ID      DATE                           REVISION
0       2026-11-24 14:39:00 +0100 CET  HEAD (7231e31)
1       2026-11-24 14:52:24 +0100 CET  HEAD (cac301b)
2       2026-11-24 14:56:38 +0100 CET  HEAD (41be646)
3       2026-11-24 15:03:23 +0100 CET  HEAD (d4ece1c)
4       2026-11-24 15:05:43 +0100 CET  HEAD (3d162be)
5       2026-11-24 15:06:13 +0100 CET  HEAD (e691d18)
6       2026-11-24 15:07:06 +0100 CET  HEAD (d296892)
7       2026-11-24 15:08:23 +0100 CET  HEAD (e691d18)
8       2026-11-24 15:08:53 +0100 CET  HEAD (d296892)
9       2026-11-24 15:09:00 +0100 CET  HEAD (ad3db13)
```

- Sortie de `argocd app list` :

```
NAME              CLUSTER                         NAMESPACE  PROJECT  STATUS  HEALTH   SYNCPOLICY  CONDITIONS  REPO                                            PATH            TARGET
argocd/concierge  https://kubernetes.default.svc  staybook   default  Synced  Healthy  Auto-Prune  <none>      https://github.com/Sephoyo/staybook-gitops.git  apps/concierge  HEAD
argocd/staybook   https://kubernetes.default.svc  staybook   default  Synced  Healthy  Auto-Prune  <none>      https://github.com/Sephoyo/staybook-gitops.git  apps/staybook   HEAD
```

- Sortie de `git log --oneline` (pour relier l'historique Argo CD aux commits, cf. Q34) :

```
ba401da docs: réponses 29, 33 et 34 avec les sorties réelles
ad3db13 Revert "feat: front v3 (image inexistante)"
d296892 feat: front v3 (image inexistante)
e691d18 test: suppression du ConfigMap de test
3d162be test: ConfigMap pour le prune
d4ece1c feat: front v2
c5d0e3c feat: auto-sync avec prune et selfHeal pour staybook et concierge
41be646 feat: hotels-service passe à 2 réplicas
cac301b docs: réponses aux questions 1 à 34
1019162 feat: Applications staybook et concierge pointent vers mon dépôt
7231e31 feat: StayBook manifests + Argo CD install (v1)
```
