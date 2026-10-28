# Projet DevOps 347 : Déploiement de WordPress avec Docker Swarm

> **École** : ETML / Lausanne  
> **Module** : ICT-347 – Utiliser un service avec des conteneurs  
> **Dépôt GitHub** : `347-docker-swarm-wordpress`  
> **Domaine d'accès cible** : [http://www.my-wordpress.ch](http://www.my-wordpress.ch)

---

## 1. Présentation du Projet

Ce projet a pour objectif le déploiement d'une infrastructure web **WordPress hautement disponible, évolutive et sécurisée** en utilisant le mode cluster **Docker Swarm**.

### Architecture globale
* **Nœuds du Cluster Swarm** : 3 machines virtuelles Linux
  * `LEADER` / `swarm-manager` (Manager, orchestration & base de données) : `10.228.242.214`
  * `WORKER1` (Worker applicatif) : `10.228.242.213`
  * `WORKER2` (Worker applicatif) : `10.228.242.215`
* **Services conteneurisés** :
  * **Reverse Proxy** : `nginx:1.28.0-alpine3.21` (2 répliques, écoute sur le port 80 en mode ingress)
  * **Moteur Web / PHP** : `wordpress:6.8.2-fpm-alpine` (2 répliques, traitement FastCGI sur le port 9000)
  * **Base de données** : `mariadb:12.0.1` (1 réplique contrainte sur le Manager avec persistance)
* **Supervision & Monitoring** : Interface graphique open-source **Swarmpit** accessible sur le port 888.
* **Sécurité & Cloisonnement** :
  * 2 réseaux Overlay distincts : `frontend` (Nginx ↔ WordPress) et `backend` (WordPress ↔ MariaDB). Nginx n'a aucun accès direct à MariaDB.
  * Mots de passe gérés exclusivement par **Docker Secrets** (aucun mot de passe en clair).
* **Stockage persistant & partagé** :
  * Volume persistant `db_data` pour MariaDB.
  * Volume partagé `wp_data` (`/var/www/html`) accessible à la fois par WordPress-FPM et Nginx (pour la mise en cache et le service direct des assets statiques et médias).

---

## 2. Structure des Fichiers

```text
347-docker-swarm-wordpress/
├── .gitignore                      # Exclusion des fichiers sensibles du suivi Git
├── README.md                       # Guide complet étape par étape (ce fichier)
├── docker-compose.yml              # Définition de la stack avec commentaires exhaustifs
├── nginx/
│   └── default.conf                # Configuration Nginx (Reverse Proxy + FastCGI)
└── secrets/
    ├── db_root_password.txt        # Secret : mot de passe root MariaDB
    ├── db_password.txt             # Secret : mot de passe utilisateur MariaDB
    ├── db_root_password.example.txt# Modèle de secret root
    └── db_password.example.txt     # Modèle de secret utilisateur
```

---

## 3. Guide de Déploiement Étape par Étape (CLI)

### Étape 1 : Initialisation du Cluster Docker Swarm

Sur le nœud **Manager** (`10.228.242.214`) :
```bash
# Initialiser le cluster Swarm en spécifiant l'IP d'écoute
docker swarm init --advertise-addr 10.228.242.214

# Récupérer le token pour joindre les workers si nécessaire
docker swarm join-token worker
```

Sur **WORKER1** (`10.228.242.213`) et **WORKER2** (`10.228.242.215`) :
```bash
# Exécuter la commande de jonction fournie par le manager
docker swarm join --token <TOKEN_FOURNI> 10.228.242.214:2377
```

Vérification sur le Manager :
```bash
docker node ls
```
*Le résultat doit afficher 3 nœuds en état `Ready` et `Active`.*

---

### Étape 2 : Déploiement de l'interface Swarmpit (Monitoring)

Sur le Manager :
```bash
# Télécharger la configuration officielle adaptée
curl -fsSL https://raw.githubusercontent.com/swarmpit/swarmpit/master/docker-compose.yml -o docker-compose-swarmpit.yml

# Ajuster DOCKER_API_VERSION si Docker Engine >= 26
sed -i 's/DOCKER_API_VERSION=1.35/DOCKER_API_VERSION=1.40/g' docker-compose-swarmpit.yml

# Déployer la stack Swarmpit
docker stack deploy -c docker-compose-swarmpit.yml swarmpit
```

*Accès à l'interface graphique :* **http://10.228.242.214:888**

---

### Étape 3 : Préparation du Projet et des Secrets

Sur le Manager :
```bash
# 1. Cloner ou copier le dépôt de travail
git clone <URL_DU_REPO_GITHUB> 347-docker-swarm-wordpress
cd 347-docker-swarm-wordpress

# 2. Créer le dossier des secrets et les fichiers de mots de passe
mkdir -p secrets
echo "RootStrongPassword2026!" > secrets/db_root_password.txt
echo "WordpressSecretPass2026!" > secrets/db_password.txt

# 3. Restreindre les permissions des secrets
chmod 600 secrets/*.txt
```

---

### Étape 4 : Déploiement de la Stack WordPress

Toujours depuis le dossier `347-docker-swarm-wordpress` sur le Manager :
```bash
# Déployer la stack complète
docker stack deploy -c docker-compose.yml wp-stack
```

---

### Étape 5 : Vérification du Déploiement et de la Haute Disponibilité

```bash
# 1. Vérifier l'état de la stack
docker stack ls

# 2. Vérifier les services et leurs répliques (1/1 db, 2/2 wordpress, 2/2 nginx)
docker stack services wp-stack

# 3. Inspecter la répartition des conteneurs sur les nœuds
docker stack ps wp-stack --no-trunc
```

---

### Étape 6 : Configuration du Nom de Domaine sur la Machine Client (Windows)

Pour accéder au site via l'URL demandée `http://www.my-wordpress.ch` :

1. Ouvrir le **Bloc-notes** sous Windows en tant qu'**Administrateur**.
2. Ouvrir le fichier système : `C:\Windows\System32\drivers\etc\hosts`.
3. Ajouter la ligne suivante à la fin du fichier :
   ```text
   10.228.242.214    www.my-wordpress.ch my-wordpress.ch
   ```
4. Sauvegarder le fichier.
5. Ouvrir votre navigateur et naviguer vers : **[http://www.my-wordpress.ch](http://www.my-wordpress.ch)**.

L'assistant d'installation standard de WordPress s'affiche alors, prêt pour l'installation finale.

---

##  4. Commandes de Suppression Totale (Clean Uninstall)

Conformément à la spécification (Section 7.3, point 4), voici la procédure exhaustive pour **supprimer totalement la stack Swarm ainsi que tous ses volumes et configurations associés** :

```bash
# 1. Supprimer la stack WordPress et l'ensemble de ses services et réseaux
docker stack rm wp-stack

# 2. Attendre quelques secondes que tous les conteneurs soient arrêtés
sleep 10

# 3. Supprimer définitivement les volumes de données persistantes associés
docker volume rm wp-stack_db_data wp-stack_wp_data

# 4. (Optionnel) Vérifier qu'il ne reste aucun résidu
docker volume ls | grep wp-stack
docker service ls | grep wp-stack


##  5. Commandes Utiles pour l'Exploitation et le Debug

| Action | Commande |
| :--- | :--- |
| **Logs d'un service** | `docker service logs -f wp-stack_nginx` |
| **Mettre à l'échelle (scale)** | `docker service scale wp-stack_wordpress=4` |
| **Inspecter un service** | `docker service inspect --pretty wp-stack_wordpress` |
| **Vérifier les secrets** | `docker secret ls` |
| **Vérifier les configs** | `docker config ls` |
