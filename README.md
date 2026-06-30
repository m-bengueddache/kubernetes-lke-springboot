# Kubernetes — Spring Boot + MySQL on Linode LKE

> **FR** — Déploiement d'une application Java Spring Boot connectée à un cluster MySQL en réplication sur Kubernetes (Linode LKE) : manifests bruts, Secret/ConfigMap, gestion d'une registry privée Docker Hub, chart Helm écrit de zéro pour l'application et phpMyAdmin, et orchestration multi-release avec Helmfile.
>
> **EN** — Deployment of a Java Spring Boot application connected to a MySQL replication cluster on Kubernetes (Linode LKE): raw manifests, Secret/ConfigMap, private Docker Hub registry handling, a Helm chart written from scratch for the app and phpMyAdmin, and multi-release orchestration with Helmfile.

---

## Stack

![Spring Boot](https://img.shields.io/badge/Spring_Boot-3-6DB33F?logo=springboot)
![MySQL](https://img.shields.io/badge/MySQL-Replication-4479A1?logo=mysql)
![Kubernetes](https://img.shields.io/badge/Kubernetes-LKE-326CE5?logo=kubernetes)
![Helm](https://img.shields.io/badge/Helm-3-0F1689?logo=helm)
![Helmfile](https://img.shields.io/badge/Helmfile-declarative-5C5C5C)
![Docker](https://img.shields.io/badge/Docker-Hub-2496ED?logo=docker)
![Linode](https://img.shields.io/badge/Linode-Akamai-00A95C?logo=linode)

---

## FR — Description

### Étape 1 — Build et image Docker

L'application Spring Boot est compilée avec Gradle et packagée dans une image Docker. Le `groupId` du connecteur MySQL a changé depuis la version 8.0.31 (`mysql` → `com.mysql`) — point important lors de la migration vers Spring Boot 3.x.

L'image est poussée sur Docker Hub dans un repository privé, ce qui implique une gestion des credentials côté Kubernetes (voir étape 2).

### Étape 2 — Manifests Kubernetes bruts

`Secret`, `ConfigMap`, `Deployment`, `Service` appliqués avec `kubectl apply`.

Le `Secret` stocke les credentials MySQL en base64 (`mysql-user-password`, `database_name`). Le `ConfigMap` expose le hostname du service MySQL primaire (`mysqldb-primary`) comme variable d'environnement `DB_SERVER`. Les env vars du `Deployment` utilisent `valueFrom.secretKeyRef` et `valueFrom.configMapKeyRef` pour injecter ces valeurs sans les hardcoder dans le manifest.

Docker Desktop (Windows/Mac) stocke les credentials dans le keystore système plutôt que dans `~/.docker/config.json`. Kubernetes ne peut pas y accéder — le secret de registry doit être créé manuellement avec `kubectl create secret docker-registry` et référencé dans `imagePullSecrets`.

### Étape 3 — MySQL en réplication avec Bitnami Helm

`StatefulSet` primary + `StatefulSet` secondary, `PersistentVolumeClaim` par pod, `Service` headless.

Le chart Bitnami est surchargé via `mysql/values.yaml`. La `storageClass: linode-block-storage` provisionne les volumes cloud dynamiquement. `volumePermissions.enabled: false` est requis pour éviter un `ImagePullBackOff` sur `bitnami/os-shell` (soumis aux restrictions Docker Hub). L'image `bitnamilegacy/mysql` remplace `bitnami/mysql` (soumise à abonnement payant).

Les `PersistentVolumeClaims` survivent à un `helm uninstall` — si les mots de passe changent entre deux installations, l'ancienne data persiste sur le volume et MySQL refuse les nouvelles credentials. Toujours supprimer les PVCs avant de réinstaller.

### Étape 4 — Chart Helm écrit de zéro + Helmfile

Un seul chart partagé (`charts/app`) déployé deux fois via Helmfile — une release `java-app`, une release `phpmyadmin` — chacune avec son fichier de valeurs.

**`toYaml` pour les env vars** : les env vars utilisent `valueFrom.secretKeyRef` et `valueFrom.configMapKeyRef`, des structures imbriquées qu'une boucle `range` avec `value: {{ .value }}` ne peut pas rendre. `toYaml .Values.containerEnvVars | nindent 8` sérialise la liste telle quelle, quelle que soit la profondeur des champs.

**Secret Bitnami plutôt que Secret applicatif** : les mots de passe (`DB_PWD`, `MYSQL_ROOT_PASSWORD`) référencent directement le Secret généré par Helm lors de l'installation MySQL (`mysqldb`). Aucun mot de passe n'apparaît dans les fichiers de valeurs. Le chart supporte la création d'un Secret applicatif via `b64enc` et `secret.enabled: true` si nécessaire, mais cette approche n'est pas utilisée ici.

**`imagePullSecrets` conditionnel** : java-app pull depuis une registry privée et nécessite `imagePullSecrets`. phpMyAdmin utilise une image publique et n'en a pas besoin. Un bloc `{{- if .Values.imagePullSecrets }}` dans le template gère les deux cas sans dupliquer le chart.

**Guards `enabled`** : seuls `ConfigMap` et `Ingress` sont rendus par la release java-app (`enabled: true`). `Secret` est désactivé pour les deux releases — les credentials viennent du Secret Bitnami. La release phpmyadmin désactive également `ConfigMap` et `Ingress` (`enabled: false`).

L'Ingress route le trafic entrant vers java-app via `/` en s'appuyant sur le Nginx Ingress Controller qui provisionne automatiquement un NodeBalancer Linode.

---

## EN — Description

### Step 1 — Build and Docker image

The Spring Boot application is compiled with Gradle and packaged into a Docker image. The MySQL connector groupId changed in version 8.0.31 (`mysql` → `com.mysql`) — a breaking point when migrating to Spring Boot 3.x.

The image is pushed to a private Docker Hub repository, which requires credential management on the Kubernetes side (see step 2).

### Step 2 — Raw Kubernetes manifests

`Secret`, `ConfigMap`, `Deployment`, `Service` applied with `kubectl apply`.

The `Secret` stores MySQL credentials as base64 (`mysql-user-password`, `database_name`). The `ConfigMap` exposes the MySQL primary service hostname (`mysqldb-primary`) as the `DB_SERVER` environment variable. The `Deployment` env vars use `valueFrom.secretKeyRef` and `valueFrom.configMapKeyRef` to inject these values without hardcoding them in the manifest.

Docker Desktop (Windows/Mac) stores credentials in the system keystore rather than `~/.docker/config.json`. Kubernetes cannot access it — the registry secret must be created manually with `kubectl create secret docker-registry` and referenced in `imagePullSecrets`.

### Step 3 — MySQL replication with Bitnami Helm

Primary `StatefulSet` + secondary `StatefulSet`, one `PersistentVolumeClaim` per pod, headless `Service`.

The Bitnami chart is overridden via `mysql/values.yaml`. `storageClass: linode-block-storage` dynamically provisions cloud volumes. `volumePermissions.enabled: false` is required to avoid `ImagePullBackOff` on `bitnami/os-shell` (subject to Docker Hub rate limits). The `bitnamilegacy/mysql` image replaces `bitnami/mysql` (which requires a paid subscription).

`PersistentVolumeClaims` survive a `helm uninstall` — if passwords change between installs, the old data persists on the volume and MySQL rejects the new credentials. Always delete PVCs before reinstalling.

### Step 4 — Helm chart from scratch + Helmfile

A single shared chart (`charts/app`) deployed twice via Helmfile — one `java-app` release, one `phpmyadmin` release — each with its own values file.

**`toYaml` for env vars**: env vars use `valueFrom.secretKeyRef` and `valueFrom.configMapKeyRef`, nested structures that a `range` loop with `value: {{ .value }}` cannot render. `toYaml .Values.containerEnvVars | nindent 8` serializes the list as-is, regardless of field depth.

**Bitnami Secret over custom Secret**: passwords (`DB_PWD`, `MYSQL_ROOT_PASSWORD`) reference the Secret generated by Helm during MySQL installation (`mysqldb`). No password appears in the values files. The chart supports creating a custom Secret via `b64enc` and `secret.enabled: true` if needed, but this approach is not used here.

**Conditional `imagePullSecrets`**: java-app pulls from a private registry and needs `imagePullSecrets`. phpMyAdmin uses a public image and does not. A `{{- if .Values.imagePullSecrets }}` block in the template handles both cases without duplicating the chart.

**`enabled` guards**: only `ConfigMap` and `Ingress` are rendered by the java-app release (`enabled: true`). `Secret` is disabled for both releases — credentials come from the Bitnami-generated Secret. The phpmyadmin release also disables `ConfigMap` and `Ingress` (`enabled: false`).

The Ingress routes incoming traffic to java-app via `/`, relying on the Nginx Ingress Controller which automatically provisions a Linode NodeBalancer.
