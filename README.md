
# Configuration des 4 VMs pour le Déploiement de BookStack

## 7. **Créer et Configurer les 4 VMs**

### Configuration des Rôles
| Rôle                    | Nom recommandé | Configuration spécifique                     |
|--------------------------|----------------|----------------------------------------------|
| Application (réplica 1)  | `app-replica-1`| Volume partagé pour les fichiers uploadés.   |
| Application (réplica 2)  | `app-replica-2`| Idem que `app-replica-1`.                    |
| Reverse Proxy            | `reverse-proxy`| Expose les ports 80/443 pour l’accès externe.|
| Base de données + Redis  | `db-redis`     | Volume supplémentaire pour les données.      |

### Étapes de Configuration des VMs

#### 1. Application (Réplicas 1 et 2)
1. Installez Docker et Docker Compose :
   ```bash
   sudo apt update && sudo apt install -y docker.io docker-compose
   ```
2. Configurez un volume partagé pour les fichiers uploadés par les utilisateurs :
   - Installez le client NFS :
     ```bash
     sudo apt install -y nfs-common
     ```
   - Montez le stockage partagé :
     ```bash
     sudo mount -t nfs <IP_NFS_SERVER>:/srv/nfs_share /mnt/shared-storage
     ```
   - Ajoutez au fichier `/etc/fstab` pour un montage automatique au démarrage :
     ```
     <IP_NFS_SERVER>:/shared /mnt/shared-storage nfs defaults 0 0
     ```
3. Placez le fichier `docker-compose.yml` pour déployer l’application BookStack.
4. Démarrez le conteneur :
   ```bash
   docker-compose up -d
   ```

#### 2. Reverse Proxy
1. Installez Docker et Docker Compose :
   ```bash
   sudo apt update && sudo apt install -y docker.io docker-compose
   ```
2. Placez le fichier `docker-compose.yml` pour Traefik ou Nginx.
3. Démarrez le reverse proxy :
   ```bash
   docker-compose up -d
   ```
4. Configurez les règles de pare-feu pour n’autoriser que les ports 80 et 443 en accès externe.

#### 3. Base de Données + Redis
1. Installez Docker et Docker Compose :
   ```bash
   sudo apt update && sudo apt install -y docker.io docker-compose
   ```
2. Créez un volume local pour les données de la base de données :
   ```bash
   mkdir -p /mnt/db_data
   ```
3. Placez le fichier `docker-compose.yml` pour MariaDB et Redis.
4. Démarrez les services :
   ```bash
   docker-compose up -d
   ```
5. Configurez des sauvegardes automatiques :
   - Ajoutez un cron job pour effectuer des sauvegardes régulières :
     ```bash
     crontab -e
     ```
     Exemple de commande pour une sauvegarde quotidienne :
     ```bash
     0 2 * * * docker exec mariadb sh -c 'exec mysqldump -u root -p"rootpassword" bookstack' > /backups/backup.sql
     ```

---

### Vérification
1. Vérifiez que chaque VM peut communiquer avec les autres via leurs IP privées.
2. Testez le bon fonctionnement des services Docker sur chaque VM :
   ```bash
   docker ps
   ```
3. Testez la montée en charge en déconnectant un des réplicas pour vérifier la résilience.



# ca c'est suite 


# Documentation: Déploiement de BookStack en Haute Disponibilité

## Prérequis

Avant de commencer, assurez-vous d’avoir les éléments suivants :
- Serveurs physiques ou machines virtuelles disponibles (minimum 4) :
  - **Serveur 1 et 2** : Hébergement des réplicas de l’application BookStack.
  - **Serveur 3** : Reverse proxy (Traefik ou Nginx).
  - **Serveur 4** : Base de données MySQL/MariaDB et Redis.
- Accès SSH à tous les serveurs.
- Docker et Docker Compose installés sur chaque serveur.
- Une solution de stockage partagé entre les réplicas (NFS, GlusterFS, ou autre).
- Domaine configuré avec un certificat SSL.

---

## Étapes du Déploiement

### 1. Configuration du Reverse Proxy

#### Installation de Nginx

1. Connectez-vous au serveur du reverse proxy.
2. Créez un fichier `docker-compose.yml` pour Nginx :

```yaml
version: '3.8'

services:
  nginx:
    image: nginx:latest
    container_name: nginx_reverse_proxy
    ports:
      - "80:80" # HTTP
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./logs:/var/log/nginx
    networks:
      - backend

networks:
  backend:
    driver: bridge
```

3. Démarrez Nginx :

```bash
docker-compose up -d
```

### 2. Déploiement de BookStack

#### Création d'un volume partagé

1. Configurez un stockage partagé (NFS ou GlusterFS) pour les fichiers téléchargés par les utilisateurs.
2. Montez le volume partagé sur les serveurs hébergeant les réplicas BookStack.

#### Configuration des réplicas

1. Créez un fichier `docker-compose.yml` pour BookStack sur chaque serveur d’application :

```yaml
version: '3.8'

services:
  bookstack:
    image: linuxserver/bookstack
    container_name: bookstack_replica
    environment:
      - DB_HOST=192.168.64.6 # IP de la VM base de données
      - DB_USER=bookstack
      - DB_PASS=password
      - DB_DATABASE=bookstack
      - APP_URL=http://192.168.64.4 # URL du reverse proxy
      - APP_KEY=base64:BE4aoEklcVGIb+cLYOauITaK4RJEOAFq5y64t/eodOo= #docker run -it --rm --entrypoint /bin/bash lscr.io/linuxserver/bookstack:latest appkey
    volumes:
      - /mnt/shared-storage/uploads:/config/uploads # NFS pour les fichiers utilisateur
    networks:
      - backend
    ports:
      - "8080:80" # Port pour le débogage local (facultatif)

networks:
  backend:
    driver: bridge
```

2. Démarrez BookStack :

```bash
docker-compose up -d
```

3. Répétez cette opération sur le second serveur d’application.

### 3. Base de données MySQL/MariaDB en Haute Disponibilité

#### Option 1 : Configuration Standard

1. Créez un fichier `docker-compose.yml` :

```yaml
version: '3.8'
services:
  db:
    image: mariadb:10.11
    container_name: mariadb
    environment:
      - MYSQL_ROOT_PASSWORD=rootpassword
      - MYSQL_DATABASE=bookstack
      - MYSQL_USER=bookstack
      - MYSQL_PASSWORD=securepassword
    volumes:
      - /mnt/db_data:/var/lib/mysql
    networks:
      - db_network

networks:
  db_network:
    driver: bridge
```

2. Lancez le conteneur :

```bash
docker-compose up -d
```

#### Option 2 : Cluster MySQL/MariaDB (Docker Swarm)

1. Initiez un cluster Docker Swarm :

```bash
docker swarm init
```

2. Configurez un cluster MySQL/MariaDB avec Galera ou autre outil de réplication en haute disponibilité (documentation dédiée).

### 4. Sauvegardes

#### Base de données

1. Installez un outil de sauvegarde comme `mysqldump` :

```bash
mysqldump -u root -p bookstack > /backups/bookstack_backup.sql
```

2. Planifiez des sauvegardes régulières avec un cron job :

```bash
crontab -e
```

Ajoutez une ligne :

```bash
0 2 * * * docker exec mariadb sh -c 'exec mysqldump -u root -p"rootpassword" bookstack' > /backups/backup.sql
```

#### Fichiers

1. Utilisez `rsync` pour copier les fichiers du volume partagé vers un emplacement sécurisé :

```bash
rsync -avz /mnt/shared-storage /backups/uploads
```

2. Planifiez avec cron :

```bash
crontab -e
```

Ajoutez une ligne :

```bash
0 3 * * * rsync -avz /mnt/shared-storage /backups/uploads
```

### 5. Monitoring et Health Checks

1. Configurez des health checks Docker pour chaque service :

Dans `docker-compose.yml` :

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost"]
  interval: 30s
  timeout: 10s
  retries: 3
```

2. Installez un outil comme Prometheus + Grafana pour surveiller les performances.

---

## Fichiers fournis

- `docker-compose.yml` pour Nginx.
- `docker-compose.yml` pour BookStack.
- Scripts de sauvegarde.

---

## Points Optionnels

- Configurez un cluster MySQL/MariaDB haute disponibilité si possible.
- Ajoutez un système d’alerte avec Prometheus ou Grafana.

---

## Tests de Validation

1. Accédez à BookStack via le domaine configuré.
2. Téléchargez des fichiers et vérifiez leur présence sur les deux réplicas.
3. Déconnectez un réplica pour tester la résilience.
4. Restaurer une sauvegarde de la base de données et des fichiers.

---

En suivant cette documentation, vous disposerez d’une application BookStack hautement disponible et sécurisée.
