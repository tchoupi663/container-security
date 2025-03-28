Absolument \! Voici une version corrigée et reformulée de votre texte, avec des améliorations pour plus de clarté :

# Session 4

## Sécurité dans la CI/CD

### 1\. Signature d'images avec COSIGN

Nous allons commencer par créer une paire de clés GPG :

```sh
gpg --full-generate-key
gpg --list-secret-keys
```

Après avoir choisi toutes les options qui nous conviennent, nous devrions pouvoir voir notre clé dans la liste.

```sh
sec   rsa3072 2025-03-28 [SC] [expiration : 2025-04-04]
      28157687BAD134D64E11FA0CF539DFE4225745CC
uid           [dernier] Ruoran ZHOU (container_sec) <ruoran.zhou.pro@gmail.com>
ssb   rsa3072 2025-03-28 [E] [expiration : 2025-04-04]
```

Nous pouvons donc l'exporter afin de pouvoir l'utiliser. Le fichier exporté ressemble à ceci :

```sh
cat private-gpg.key
```

```sh
-----BEGIN PGP PRIVATE KEY BLOCK-----

lQWGBGfmhGsBDACs7je3cUZLvuvcfwX2m9cjAMNzbnV...
```

Avec cette clé, nous pouvons signer nos commits et les pousser vers notre dépôt GitLab.

![img](images/1.png)

![img](images/2.png)

![img](images/3.png)


Pour vérifier notre signature, nous pouvons utiliser la commande suivante :

```sh
./cosign verify --key gpg://public-gpg.key registry.gitlab.com/votre-projet/image:v1
```

Cela nous donne le résultat suivant :

```sh
Verified OK
```

Mais lorsque nous n'avons pas signé, le résultat est le suivant :

```sh
./cosign verify --key gpg://public-gpg.key registry.gitlab.com/votre-projet/image:v2

Error: verifying signature: invalid signature
```

### 2\. Créer un pipeline CI/CD avec GitLab CI

Pour cette partie, nous allons utiliser le [Dockerfile](/session4/files/Dockerfile) suivant. Nous allons ensuite simplement faire un `git push`, ce qui nous permet d'observer 4 étapes dans le pipeline :

  * Phase de lint
  * Phase de build (avec une signature Cosign également)
  * Phase de vérification de la signature Cosign
  * Analyse des vulnérabilités avec l'outil Trivy

<!-- end list -->

```sh
/container_security/session4/files/Dockerfile - DL3018: Pin versions in apk add. Instead of `apk add <package>` specify `apk add <package>=<version>` (warning)
Authenticating with existing credentials...
Login Succeeded
Sending build context to Docker daemon  3.072kB
Step 1/2 : FROM alpine:3.21.3
 ---> <image_id_alpine_3.21.3>
Step 2/2 : RUN apk add --no-cache curl=8.12.1-r1
 ---> Running in <container_id>
fetch http://dl-cdn.alpinelinux.org/alpine/v3.21/main/x86_64/APKINDEX.tar.gz
fetch http://dl-cdn.alpinelinux.org/alpine/v3.21/community/x86_64/APKINDEX.tar.gz
(1/1) Installing curl=8.12.1-r1
OK: 4 MiB in 5 packages
Removing intermediate container <container_id>
 ---> <image_id_my_image>
Successfully built <image_id_my_image>
...
```

Après cette opération, nous pouvons constater que le push a réussi.

Afin de simuler le cas où nous poussons une image avec des vulnérabilités sévères, nous allons modifier notre image Docker pour utiliser une version plus ancienne de curl (`7.79.1-r1`).

Lorsque nous faisons un push après cette modification, l'opération échoue. En effet, cette version présente une vulnérabilité qui a été identifiée par l'étape d'analyse en utilisant Trivy.

```sh
2024-10-27T10:05:00.000Z        INFO        Need to update DB
2024-10-27T10:05:05.000Z        INFO        DB updated: trivy.db.version: 1, trivy.db.updated_at: 2024-10-27T10:05:05Z
<gitlab_registry_image>:<commit_ref_slug> (alpine 3.12)
HIGH: 2, CRITICAL: 1

<VULNERABILITY_DETAILS_1>
<VULNERABILITY_DETAILS_2>
<VULNERABILITY_DETAILS_3>

Exit code: 1
```
