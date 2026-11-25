# TP — Docker avancé

Ce TP permet d'approfondir le fonctionnement interne de Docker, les mécanismes de sécurité et les outils de diagnostic.

---
## Exercice 1 — Comprendre le cycle de vie des processus dans un conteneur

**Objectif :** comprendre comment Docker gère les processus et pourquoi certains conteneurs s'arrêtent immédiatement.

### Étape 1 : Lancer un conteneur avec un processus actif

```bash
docker run -d --name processus-test alpine:3.22 sleep 300
```

### Étape 2 : Identifier les processus

```bash
docker top processus-test
docker inspect --format='PID hôte : {{.State.Pid}}' processus-test
```

Observez le processus exécuté dans le conteneur et son processus correspondant sur l'hôte.

### Étape 3 : Arrêter le conteneur

```bash
docker stop processus-test
```

Par défaut, Docker envoie un signal `SIGTERM` au processus principal, attend le délai prévu, puis peut envoyer `SIGKILL` si le processus ne s'arrête pas.

Vérifiez le résultat :

```bash
docker ps -a
docker inspect --format='État : {{.State.Status}}' processus-test
```

### Étape 4 : Comparer avec un processus de longue durée

Lancez un nouveau conteneur :

```bash
docker run -d --name processus-test-2 alpine:3.22 sleep 300
```

Puis envoyez un signal d'arrêt au processus principal :

```bash
docker kill --signal=TERM processus-test-2
```

Observez l'état du conteneur :

```bash
docker inspect --format='État : {{.State.Status}}' processus-test-2
```

### Étape 5 : Nettoyer

```bash
docker rm processus-test processus-test-2
```

### Questions

1. Quel est le rôle du processus principal dans un conteneur ? Le processus principal (PID 1) est celui qui définit la vie du conteneur. Le conteneur démarre avec lui, et il s'arrête quand ce processus se termine.
2. Quelle différence existe-t-il entre `docker stop` et `docker kill` ? `docker stop` envoie un `SIGTERM` pour laisser l'application s'arrêter proprement, puis un `SIGKILL` si elle ne s'arrête pas après le délai prévu (10 secondes par défaut). `docker kill` envoie directement un `SIGKILL`, sans délai (ou le signal choisi avec `--signal`).
3. Pourquoi un conteneur s'arrête-t-il lorsque son processus principal se termine ? Parce qu'un conteneur n'existe que pour exécuter ce processus. Une fois qu'il est terminé, il n'y a plus rien à exécuter, donc Docker arrête le conteneur.
4. Quelle différence existe-t-il entre `SIGTERM` et `SIGKILL` ? `SIGTERM` est une demande d'arrêt que le processus peut intercepter pour fermer ses fichiers et connexions proprement. `SIGKILL` ne peut pas être intercepté ni ignoré : le processus est tué immédiatement.
5. Pourquoi est-il important qu'une application gère correctement les signaux d'arrêt ? Pour qu'elle puisse s'arrêter proprement (terminer les requêtes en cours, sauvegarder les données, fermer les connexions) et éviter d'être tuée brutalement par `SIGKILL` avec un risque de perte de données.

## Exercice 2 — Comprendre les couches d'une image et le système de fichiers

**Objectif :** comprendre comment Docker stocke les images et les modifications effectuées dans un conteneur.

### Étape 1 : Examiner les couches d'une image

Téléchargez une image Alpine :

```bash
docker pull alpine:3.22
```

Affichez l'historique de construction :

```bash
docker history alpine:3.22
```

Examinez ensuite les métadonnées de l'image :

```bash
docker image inspect alpine:3.22
```

Observez notamment les informations relatives aux couches de l'image.

### Étape 2 : Créer un conteneur de test

```bash
docker run -d \
  --name couches-test \
  alpine:3.22 \
  sh -c "sleep 600"
```

Le conteneur reste actif grâce à la commande `sleep`.

### Étape 3 : Modifier le système de fichiers

Créez un fichier et un répertoire dans le conteneur :

```bash
docker exec couches-test sh -c \
  "echo 'Modification Docker' > /tmp/test.txt"

docker exec couches-test mkdir /opt/test
```

Modifiez également un fichier existant :

```bash
docker exec couches-test sh -c \
  "echo '# modification locale' >> /etc/alpine-release"
```

### Étape 4 : Identifier les modifications

Exécutez :

```bash
docker diff couches-test
```

Les lignes affichées utilisent généralement les indicateurs suivants :

- `A` : élément ajouté.
- `C` : élément modifié.
- `D` : élément supprimé.

Vérifiez le contenu des fichiers :

```bash
docker exec couches-test cat /tmp/test.txt
docker exec couches-test cat /etc/alpine-release
```

### Étape 5 : Supprimer le conteneur

```bash
docker rm -f couches-test
```

Relancez ensuite un nouveau conteneur à partir de la même image :

```bash
docker run --rm alpine:3.22 cat /etc/alpine-release
```

La modification précédente n'est pas présente dans ce nouveau conteneur.

### Questions

1. Quelle différence existe-t-il entre les couches d'une image et la couche inscriptible d'un conteneur ? Les couches d'une image sont en lecture seule et partagées entre tous les conteneurs qui utilisent cette image. La couche inscriptible est propre à chaque conteneur et ajoutée par-dessus, c'est là que vont toutes les modifications.
2. À quoi sert le mécanisme *copy-on-write* ? Quand un conteneur modifie un fichier qui vient d'une couche de l'image, le fichier est d'abord copié dans la couche inscriptible, puis modifié là. Cela évite de dupliquer toute l'image pour chaque conteneur.
3. Pourquoi les modifications effectuées dans un conteneur ne modifient-elles pas directement l'image d'origine ? Parce que les couches de l'image sont en lecture seule. Les changements sont écrits uniquement dans la couche du conteneur, ce qui permet de réutiliser la même image pour plusieurs conteneurs sans qu'ils s'influencent.
4. À quoi sert la commande `docker diff` ? Elle liste les changements faits dans le système de fichiers d'un conteneur par rapport à son image : fichiers ajoutés (`A`), modifiés (`C`) ou supprimés (`D`).
5. Pourquoi les modifications apportées à la couche inscriptible sont-elles perdues lorsqu'on supprime le conteneur ? Parce que cette couche appartient au conteneur. Quand on le supprime, elle est supprimée avec lui. Pour conserver des données, il faut utiliser des volumes.

---

## Exercice 3 — Construire une image avec plusieurs étapes (Multi-stage build)

**Objectif :** séparer l'environnement de compilation de l'environnement d'exécution pour produire une image finale plus légère.

Dans cet exercice, vous allez créer une petite application HTTP en Go et utiliser un build multi-stage.

### Étape 1 : Créer le projet

Créez le dossier suivant :

```text
tp-multistage/
├── main.go
└── Dockerfile
```

### Étape 2 : Écrire l'application

Dans `main.go`, ajoutez :

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "Application Go executee dans Docker !")
	})

	log.Println("Serveur demarre sur le port 8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### Étape 3 : Créer le Dockerfile multi-stage

Dans `Dockerfile`, écrivez :

```dockerfile
FROM golang:1.24-alpine AS builder

WORKDIR /src

COPY main.go .

RUN CGO_ENABLED=0 GOOS=linux go build -o /app main.go

FROM scratch

COPY --from=builder /app /app

EXPOSE 8080

ENTRYPOINT ["/app"]
```

**Explication :**

- `FROM golang:1.24-alpine AS builder` définit l'environnement de compilation.
- `RUN` compile le programme en un exécutable autonome.
- `FROM scratch` démarre une image finale vide.
- `COPY --from=builder` récupère uniquement l'exécutable produit pendant la première étape.
- `ENTRYPOINT` définit le programme exécuté au démarrage.

L'image `scratch` ne contient ni shell ni outils système classiques. L'application doit donc être compilée de manière compatible avec cet environnement.

### Étape 4 : Construire l'image

Depuis le dossier `tp-multistage`, exécutez :

```bash
docker build -t application-go:v1 .
```

Affichez la taille de l'image :

```bash
docker image ls application-go
```

### Étape 5 : Exécuter l'application

```bash
docker run -d \
  --name application-go-test \
  -p 8086:8080 \
  application-go:v1
```

Testez le serveur :

```bash
curl http://localhost:8086
```

**Résultat attendu :**

```text
Application Go executee dans Docker !
```

### Étape 6 : Examiner les processus et nettoyer

```bash
docker top application-go-test
docker logs application-go-test
docker rm -f application-go-test
```

### Questions

1. Pourquoi utiliser plusieurs instructions `FROM` ? Chaque `FROM` démarre une nouvelle étape. On peut ainsi compiler l'application dans une première étape, puis ne garder dans l'image finale que ce qui est nécessaire pour l'exécuter.
2. Quelle différence existe-t-il entre l'étape `builder` et l'étape finale ? L'étape `builder` contient le compilateur Go et le code source, elle sert à produire l'exécutable. L'étape finale est une image vide (`scratch`) qui ne contient que l'exécutable copié depuis `builder`.
3. Pourquoi l'image finale ne contient-elle pas le compilateur Go ? Parce que `COPY --from=builder` ne récupère que l'exécutable `/app`. Le reste de l'étape `builder` n'est pas conservé dans l'image finale.
4. Quels sont les avantages d'une image finale minimale ? Elle est plus légère (téléchargement et démarrage plus rapides), prend moins de place, et présente une surface d'attaque réduite puisqu'il y a moins d'outils et de bibliothèques pouvant contenir des failles.
5. Quelles précautions faut-il prendre avant d'utiliser une image `scratch` en production ? Il faut compiler un exécutable statique (`CGO_ENABLED=0`), car il n'y a aucune bibliothèque système. Il n'y a pas de shell, donc pas de `docker exec` pour déboguer. Il faut aussi penser à ajouter les certificats CA si l'application fait des appels HTTPS, un utilisateur non root, et les fuseaux horaires si besoin.

---

## Exercice 4 — Renforcer la sécurité d'un conteneur

**Objectif :** limiter les privilèges et les possibilités de modification du système de fichiers d'un conteneur.

### Étape 1 : Exécuter un conteneur avec un utilisateur non privilégié

```bash
docker run --rm \
  --user 10001:10001 \
  alpine:3.22 \
  sh -c 'id'
```

Observez l'identifiant de l'utilisateur et son groupe.

### Étape 2 : Tester les permissions

Exécutez :

```bash
docker run --rm \
  --user 10001:10001 \
  alpine:3.22 \
  sh -c 'touch /root/test.txt'
```

**Résultat attendu :** l'opération échoue normalement, car l'utilisateur n'a pas les permissions nécessaires.

### Étape 3 : Exécuter un conteneur en lecture seule

```bash
docker run --rm \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=16m \
  --cap-drop=ALL \
  --security-opt=no-new-privileges \
  --user 10001:10001 \
  alpine:3.22 \
  sh -c 'id; touch /tmp/test.txt; echo "Fichier temporaire cree"'
```

Les options utilisées permettent de :

- `--read-only` : rendre le système de fichiers racine du conteneur non modifiable.
- `--tmpfs` : fournir un espace temporaire en mémoire pour les fichiers qui doivent être écrits.
- `--cap-drop=ALL` : supprimer les capacités Linux supplémentaires du processus.
- `--security-opt=no-new-privileges` : empêcher l'acquisition de privilèges supplémentaires via les mécanismes concernés.
- `--user` : exécuter le processus avec un identifiant non nul.

### Étape 4 : Vérifier les limites d'écriture

Essayez de créer un fichier dans le répertoire racine :

```bash
docker run --rm \
  --read-only \
  --user 10001:10001 \
  alpine:3.22 \
  sh -c 'touch /test.txt'
```

**Résultat attendu :** l'écriture échoue.

### Étape 5 : Comparer les configurations

Lancez un conteneur avec les options de sécurité précédentes, puis examinez sa configuration avec :

```bash
docker inspect --format='Utilisateur : {{.Config.User}}' NOM_DU_CONTENEUR
docker inspect --format='Lecture seule : {{.HostConfig.ReadonlyRootfs}}' NOM_DU_CONTENEUR
```

Remplacez `NOM_DU_CONTENEUR` par le nom d'un conteneur actif créé pour cette vérification.

### Questions

1. Pourquoi est-il préférable d'exécuter une application avec un utilisateur non privilégié ? Si l'application est compromise, l'attaquant n'a que les droits limités de cet utilisateur et non ceux de root, ce qui réduit les dégâts possibles et les risques d'évasion du conteneur.
2. Que sont les capacités Linux ? Ce sont des morceaux des privilèges de root, découpés en droits plus précis (par exemple changer le propriétaire d'un fichier ou ouvrir un port inférieur à 1024). On peut donner à un processus uniquement celles dont il a besoin.
3. Quelle différence existe-t-il entre `--read-only` et `--tmpfs` ? `--read-only` rend le système de fichiers racine du conteneur non modifiable. `--tmpfs` monte un espace temporaire en mémoire, inscriptible, à un endroit précis (ici `/tmp`).
4. Pourquoi un conteneur peut-il avoir besoin de zones inscriptibles malgré un système de fichiers racine en lecture seule ? Beaucoup d'applications ont besoin d'écrire des fichiers temporaires, des sockets, des PID ou du cache. On autorise donc l'écriture uniquement à ces endroits, le reste du système reste protégé.
5. Ces options suffisent-elles à garantir une isolation parfaite ? Justifiez votre réponse. Non. Les conteneurs partagent le noyau de l'hôte, donc une faille du noyau ou une mauvaise configuration peut permettre de s'échapper. Il faut aussi d'autres protections (seccomp, AppArmor, réseau, mises à jour, images de confiance).

---

## Exercice 5 — Diagnostiquer un conteneur qui se termine anormalement

**Objectif :** analyser le code de sortie d'un processus et exploiter les journaux pour identifier un problème.

### Étape 1 : Simuler une application en erreur

Lancez un conteneur dont le processus retourne volontairement un code d'erreur :

```bash
docker run \
  --name diagnostic-test \
  alpine:3.22 \
  sh -c 'echo "Demarrage de l application"; echo "Erreur simulee" >&2; exit 42'
```

Le conteneur doit se terminer avec le code de sortie `42`.

### Étape 2 : Examiner l'état du conteneur

```bash
docker ps -a
```

Affichez son code de sortie :

```bash
docker inspect \
  --format='Code de sortie : {{.State.ExitCode}}' \
  diagnostic-test
```

### Étape 3 : Consulter les journaux

```bash
docker logs diagnostic-test
```

Les journaux doivent permettre de retrouver les messages émis par le processus, notamment celui envoyé sur la sortie d'erreur standard.

### Étape 4 : Inspecter les métadonnées

```bash
docker inspect diagnostic-test
```

Recherchez notamment les champs :

- `State.Status`
- `State.ExitCode`
- `State.Error`
- `Config.Cmd`
- `Config.Image`

### Étape 5 : Nettoyer

```bash
docker rm diagnostic-test
```

### Questions

1. Quelle différence existe-t-il entre un conteneur arrêté et un conteneur supprimé ? Un conteneur arrêté existe toujours : il apparaît avec `docker ps -a`, et on peut consulter ses logs et son code de sortie. Un conteneur supprimé n'existe plus du tout, avec sa couche inscriptible.
2. Que signifie un code de sortie différent de zéro ? Que le processus s'est terminé avec une erreur. `0` signifie que tout s'est bien passé. Ici, `42` est un code choisi volontairement pour simuler une erreur.
3. Quelle différence existe-t-il entre les sorties standard et d'erreur ? La sortie standard (`stdout`) sert aux messages normaux du programme, et la sortie d'erreur (`stderr`) aux messages d'erreur. On peut ainsi les traiter séparément (par exemple rediriger uniquement les erreurs).
4. Pourquoi les journaux sont-ils utiles lorsqu'un conteneur se termine trop rapidement ? Parce qu'on n'a pas le temps de s'y connecter avec `docker exec`. `docker logs` permet de lire après coup ce que l'application a affiché et de comprendre pourquoi elle s'est arrêtée.
5. Pourquoi faut-il examiner le code de sortie au lieu de se fier uniquement au message d'erreur ? Le message peut être absent, incomplet ou trompeur, alors que le code de sortie indique clairement si le processus a réussi ou échoué (et parfois pourquoi, comme `137` pour un `SIGKILL` ou un manque de mémoire). Il est aussi utilisable par des scripts et outils d'orchestration.

---

## Exercice 6 — Configurer la rotation des journaux

**Objectif :** comprendre comment éviter une consommation excessive de l'espace disque par les journaux Docker.

### Étape 1 : Créer un conteneur générant des journaux

```bash
docker run -d \
  --name logs-test \
  --log-driver local \
  --log-opt max-size=1m \
  --log-opt max-file=2 \
  alpine:3.22 \
  sh -c 'i=0; while [ "$i" -lt 10000 ]; do echo "Journal numero $i"; i=$((i+1)); done; sleep 600'
```

Le conteneur génère des messages puis reste actif.

### Étape 2 : Consulter les journaux

```bash
docker logs --tail 20 logs-test
```

Affichez la configuration du pilote de logs :

```bash
docker inspect \
  --format='Pilote : {{.HostConfig.LogConfig.Type}}' \
  logs-test
```

Puis affichez les options configurées :

```bash
docker inspect \
  --format='{{json .HostConfig.LogConfig.Config}}' \
  logs-test
```

### Étape 3 : Comprendre la rotation

Dans cette configuration :

- `--log-driver local` utilise le pilote de journaux local de Docker.
- `max-size=1m` fixe une taille cible de rotation par fichier.
- `max-file=2` limite le nombre de fichiers conservés par la rotation.

Ces paramètres évitent une croissance illimitée des fichiers de logs gérés par ce pilote. Ils ne constituent pas une limite globale de stockage pour tout le conteneur.

### Étape 4 : Comparer avec une configuration sans rotation explicite

Inspectez les paramètres de journalisation d'un autre conteneur :

```bash
docker inspect \
  --format='Pilote : {{.HostConfig.LogConfig.Type}} ; Options : {{json .HostConfig.LogConfig.Config}}' \
  NOM_DU_CONTENEUR
```

Remplacez le nom par celui d'un conteneur existant.

Comparez les résultats avec la configuration de `logs-test`.

### Étape 5 : Nettoyer

```bash
docker rm -f logs-test
```

### Questions

1. Pourquoi des journaux trop volumineux peuvent-ils poser problème ? Ils peuvent remplir le disque de l'hôte, ce qui ralentit ou fait planter Docker et les autres applications. Ils sont aussi plus difficiles à consulter.
2. À quoi sert un pilote de journaux Docker ? Il définit comment et où Docker enregistre les logs d'un conteneur (fichiers locaux, syslog, système de logs externe, etc.).
3. Quelle différence existe-t-il entre la consultation des journaux avec `docker logs` et leur stockage sur le long terme ? `docker logs` permet de lire les logs gardés par Docker sur l'hôte, ils disparaissent avec le conteneur et sont limités par la rotation. Le stockage long terme consiste à envoyer les logs vers un système externe pour les conserver, les rechercher et les analyser.
4. Pourquoi la rotation des journaux est-elle importante sur un serveur ? Elle limite la taille des fichiers de logs en supprimant les plus anciens, donc elle évite de saturer le disque d'un serveur qui tourne en continu.
5. Pourquoi faut-il choisir une stratégie de journalisation adaptée à l'environnement de production ? Parce que les besoins sont différents : conservation des logs, volume généré, recherche, sécurité, obligations légales. Une mauvaise stratégie peut faire perdre des informations utiles ou consommer trop de ressources.

---

## Exercice 7 — Explorer les processus et les modifications d'un conteneur en fonctionnement

**Objectif :** utiliser les outils de diagnostic Docker pour comprendre le comportement d'un conteneur actif.

### Étape 1 : Lancer un conteneur

```bash
docker run -d \
  --name inspection-test \
  alpine:3.22 \
  sh -c 'while true; do sleep 30; done'
```

### Étape 2 : Examiner les processus

```bash
docker top inspection-test
```

Observez les processus associés au conteneur.

### Étape 3 : Exécuter des commandes dans le conteneur

```bash
docker exec inspection-test ps
docker exec inspection-test sh -c 'echo "Test de diagnostic" > /tmp/diagnostic.txt'
docker exec inspection-test cat /tmp/diagnostic.txt
```

`docker exec` lance un nouveau processus dans un conteneur déjà actif. Il ne remplace pas le processus principal.

### Étape 4 : Comparer le système de fichiers

```bash
docker diff inspection-test
```

Repérez le fichier `/tmp/diagnostic.txt` dans la liste des modifications.

### Étape 5 : Inspecter les ressources

```bash
docker stats --no-stream inspection-test
```

Observez les informations sur le CPU, la mémoire et les entrées/sorties.

### Étape 6 : Nettoyer

```bash
docker rm -f inspection-test
```

### Questions

1. Quelle différence existe-t-il entre `docker exec` et `docker run` ? `docker run` crée et démarre un nouveau conteneur à partir d'une image. `docker exec` lance une commande supplémentaire dans un conteneur déjà en cours d'exécution, sans remplacer son processus principal.
2. À quoi sert `docker top` ? À afficher les processus qui tournent dans un conteneur, depuis l'hôte, sans avoir à entrer dedans.
3. Pourquoi `docker diff` peut-il être utile pendant un diagnostic ? Il montre quels fichiers ont été ajoutés, modifiés ou supprimés dans le conteneur depuis son démarrage. Cela aide à repérer des écritures inattendues ou à comprendre ce que fait l'application.
4. Quelles informations fournit `docker stats` ? L'utilisation du CPU, de la mémoire (et sa limite), du réseau et des entrées/sorties disque, ainsi que le nombre de processus du conteneur.
5. Pourquoi faut-il éviter de modifier manuellement un conteneur en production sans documenter les changements ? Parce que les modifications faites à la main sont perdues si le conteneur est recréé, et personne ne sait ce qui a été changé. Cela rend le système difficile à reproduire et à maintenir. Il vaut mieux modifier l'image ou la configuration et la versionner.

---

### Livrables attendus

1. Les réponses argumentées aux questions.
2. Une courte conclusion sur les améliorations apportées à l'image et à la sécurité du conteneur.

Dans ce TP, on a vu comment rendre un conteneur plus propre et plus sûr. Côté image, le build multi-stage permet d'obtenir une image finale beaucoup plus légère, avec uniquement l'exécutable et sans compilateur, ce qui réduit la taille et la surface d'attaque. Côté sécurité, on limite les privilèges en utilisant un utilisateur non root, un système de fichiers en lecture seule avec un `tmpfs` pour les fichiers temporaires, la suppression des capacités Linux inutiles et `no-new-privileges`. On a aussi vu l'importance de la rotation des logs, et des outils de diagnostic (`logs`, `inspect`, `top`, `diff`, `stats`) pour comprendre ce qui se passe. Ces mesures restent insuffisantes seules, car les conteneurs partagent le noyau de l'hôte : il faut aussi mettre à jour les images et surveiller les applications.

---

**Fin du TP Docker avancé**
