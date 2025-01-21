# Documentation: Déploiement de BookStack en Haute Disponibilité

## Prérequis

Avant de commencer, assurez-vous d’avoir les éléments suivants :
- Serveurs physiques ou machines virtuelles disponibles (minimum 4) :
  - **Serveur 1 et 2** : Hébergement des réplicas de l’application BookStack.
  - **Serveur 3** : Reverse proxy (Traefik ou Nginx).
  - **Serveur 4** : Base de données MySQL/MariaDB.
- Accès SSH à tous les serveurs.
- Docker et Docker Compose installés sur chaque serveur.
- Une solution de stockage partagé entre les réplicas (NFS, GlusterFS, ou autre).
- Domaine configuré avec un certificat SSL.

# Configuration des 4 VMs pour le Déploiement de BookStack

## 7. **Créer et Configurer les 4 VMs**

### Configuration des Rôles
| Rôle                     | Nom recommandé | Configuration spécifique                     |
|--------------------------|----------------|----------------------------------------------|
| Application (réplica 1)  | `app-replica-1`| Volume partagé pour les fichiers uploadés.   |
| Application (réplica 2)  | `app-replica-2`| Idem que `app-replica-1`.                    |
| Reverse Proxy            | `reverse-proxy`| Expose les ports 80/443 pour l’accès externe.|
| Base de données          | `db-mariadb`   | Volume supplémentaire pour les données.      |

### Étapes de Configuration des VMs

#### 1. Application (Réplicas 1 et 2)
1. Installer Docker et Docker Compose :
   ```bash
    sudo apt update
    sudo apt install -y docker.io docker-compose
    sudo systemctl enable --now docker
    sudo usermod -aG docker $USER
   ```
2. Configurez un volume partagé pour les fichiers uploadés par les utilisateurs :
   - Installer le client NFS :
     ```bash
     sudo apt install -y nfs-common
     ```
   - Créer un répertoire partagé : (sur nfs serveur)
    ```bash
      sudo mkdir -p /srv/nfs_share
      sudo chmod 777 /srv/nfs_share
      sudo mkdir -p /srv/nfs_share/uploads /srv/nfs_share/db_data
      sudo chown nobody:nogroup /srv/nfs_share/uploads /srv/nfs_share/db_data
      sudo chmod 777 /srv/nfs_share/uploads /srv/nfs_share/db_data
      echo "/srv/nfs_share *(rw,sync,no_root_squash,no_subtree_check)" | sudo tee /etc/exports
      sudo exportfs -a
      sudo systemctl restart nfs-server
    ```
    - Ouvrir le fichier /etc/exports puis ajouter la ligne suivante pour partager le répertoire avec vos autres VMs et sauvegarder :
    ```bash
      sudo nano /etc/exports
      /srv/nfs_share 192.168.64.0/24(rw,sync,no_subtree_check)
      sudo exportfs -a
      sudo systemctl restart nfs-kernel-server
    ```
    - Vérfier les exportations : 
    ```bash
      sudo exportfs -v
    ```
    - Créer un point de montage :
    ```bash
      sudo mkdir -p /mnt/nfs-shared
    ```
   - Monter le stockage partagé et vérifier le montage :
     ```bash
     sudo mount -t nfs <IP_NFS_SERVER>:/srv/nfs_share /mnt/nfs-shared
     ls /mnt/nfs-shared
     ```
    - Montage des Répertoires Partagés sur les VMs
      Sur les autres VMs :
      ```bash
      sudo mkdir -p /mnt/shared-storage/uploads /mnt/db_data
      sudo mount -t nfs 192.168.64.5:/srv/nfs_share/uploads /mnt/shared-storage/uploads
      sudo mount -t nfs 192.168.64.5:/srv/nfs_share/db_data /mnt/db_data
      ```

   - Ajoutez au fichier pour un montage automatique au démarrage :
     ```
     <IP_NFS_SERVER>:/srv/nfs_share /mnt/nfs-shared nfs defaults 0 0
     ```
    - Tester et vérifier : 
    ```bash 
      echo "Test depuis VM cliente" > /mnt/nfs-shared/test.txt
    ```
    - Assurez-vous que le fichier est visible depuis d'autres VMs :
    ```bash
      cat /mnt/nfs-shared/test.txt
    ```
    - Exportation dans /etc/exports : (sur serveur)
    ```bash
      /srv/nfs_share 192.168.64.0/24(rw,sync,no_subtree_check)
    ```
3. Placez le fichier `docker-compose.yml` pour déployer l’application BookStack.
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
      - /mnt/nfs-shared/uploads:/config/uploads # NFS pour les fichiers utilisateur
    networks:
      - backend
    ports:
      - "8080:80" # Port pour le débogage local (facultatif)

networks:
  backend:
    driver: bridge
```
4. Pour tester que le conteneur démarre :
   ```bash
   docker-compose up -d
   ```

#### 2. Reverse Proxy
1. Installer Docker et Docker Compose :
   ```bash
    sudo apt update
    sudo apt install -y docker.io docker-compose
    sudo systemctl enable --now docker
    sudo usermod -aG docker $USER
   ```
2. Placer le fichier `docker-compose.yml` pour Nginx.
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
Créer un fichier de conf `nginx.conf`
```
server {
    listen 80;
    server_name 192.168.64.4;

    location / {
        proxy_pass http://bookstack:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

3. Démarrez le reverse proxy :
   ```bash
   docker-compose up -d
   ```
4. Configurez les règles de pare-feu pour n’autoriser que les ports 80 et 443 en accès externe.


#### 3. Base de Données
1. Installer Docker et Docker Compose :
   ```bash
    sudo apt update
    sudo apt install -y docker.io docker-compose
    sudo systemctl enable --now docker
    sudo usermod -aG docker $USER
   ```
2. Créez un volume local pour les données de la base de données :
   ```bash
   mkdir -p /mnt/db_data
   ```
3. Placez le fichier `docker-compose.yml` pour MariaDB.
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
4. Démarrez les services :
   ```bash
   docker-compose up -d
   ```
Cette requête à aboutie sur cette erreur : 
ERROR: toomanyrequests: You have reached your pull rate limit. You may increase the limit by authenticating and upgrading: 
https://www.docker.com/increase-rate-limit

On a donc du faire : 
```bash
docker login
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

#### Docker Swarm

1. Initier un cluster Docker Swarm sur le reverse-proxy
```bash
docker swarm init --advertise-addr 192.168.64.4
```

2. Rejoindre le swarm avec les autres VMs : 
```bash
docker swarm join <TOKEN> 192.168.64.4
```

Remplacer le `docker-compose.yml` du reverse proxy par : 
```yml
version: '3.8'

services:
  bookstack:
    image: linuxserver/bookstack
    environment:
      - DB_HOST=192.168.64.6
      - DB_USER=bookstack
      - DB_PASS=securepassword
      - DB_DATABASE=bookstack
      - APP_URL=http://192.168.64.4
    volumes:
      - /mnt/shared-storage/uploads:/config/uploads
    deploy:
      replicas: 2
      restart_policy:
        condition: on-failure
    networks:
      - backend

  reverse_proxy:
    image: nginx:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]
    networks:
      - backend

  db:
    image: mariadb:10.11
    environment:
      - MYSQL_ROOT_PASSWORD=rootpassword
      - MYSQL_DATABASE=bookstack
      - MYSQL_USER=bookstack
      - MYSQL_PASSWORD=securepassword
    volumes:
      - /mnt/db_data:/var/lib/mysql
    deploy:
      mode: global
      placement:
        constraints: [node.role == manager]
    networks:
      - backend

networks:
  backend:
    driver: overlay
```

Demarrer les services : 
```bash
docker stack deploy -c docker-compose.yml bookstack_stack
```

Une fois tout cela fait, nous avons une erreur au démarrage des services : 
```
Waiting for DB to be available
Illuminate\Database\QueryException 

SQLSTATE[HY000] [1045] Access denied for user 'database_username'@'10.0.1.173' (using password: YES) (Connection: mysql, SQL: select table_name as `name`, (data_length + index_length) as `size`, table_comment as `comment`, engine as `engine`, table_collation as `collation` from information_schema.tables where table_schema = 'bookstack' and table_type in ('BASE TABLE', 'SYSTEM VERSIONED') order by table_name)
```

on essaye des commandes tel que : 
```
GRANT ALL PRIVILEGES ON bookstack.* TO 'bookstack'@'%' IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
```

## Points Optionnels (non mis en place actuellement)

- Configurer un cluster MariaDB haute disponibilité si possible.
- Ajouter un système d’alerte avec Prometheus ou Grafana.


## **Vérifications et Tests**
### Liste des Services
```bash
docker service ls
```
### Vérification des Réplicas
```bash
docker service ps bookstack_stack_bookstack
```
### Surveiller les conteneurs : 
```bash
docker ps
```
### Consulter log service spécifiques : 
```bash
docker service logs bookstack_stack_bookstack
```

### Accès à la Base de Données depuis un Replica
1. Connectez-vous au conteneur :
   ```bash
   docker exec -it $(docker ps --filter "name=bookstack_stack_bookstack.1" -q) bash
   ```
2. Testez la connexion :
   ```bash
   mysql -h db -u bookstack -p --ssl-mode=DISABLED
   ```

### Consultation des Logs des Services
- **BookStack** :
  ```bash
  docker service logs -f bookstack_stack_bookstack
  ```
- **MariaDB** :
  ```bash
  docker service logs -f bookstack_stack_db
  ```
- **Reverse Proxy** :
  ```bash
  docker service logs -f bookstack_stack_reverse_proxy
  ```

---

## **Résolution des Problèmes Identifiés**
- Ajustement des permissions NFS :
  ```bash
  sudo chown -R 911:911 /srv/nfs_share/uploads
  sudo chmod -R 775 /srv/nfs_share/uploads
  ```
- Vérification des privilèges MariaDB :
  ```sql
  GRANT ALL PRIVILEGES ON bookstack.* TO 'bookstack'@'%' IDENTIFIED BY 'password';
  FLUSH PRIVILEGES;
  ```
- Désactivation de SSL si nécessaire :
  ```bash
  mysql -h db -u bookstack -p --ssl-mode=DISABLED
  ```

---

## **Conclusion**
Ce rapport résume l’intégralité des étapes nécessaires pour configurer, déployer et tester l’application BookStack dans un environnement Docker Swarm. Avec ces commandes et ajustements, l’infrastructure est prête à être utilisée en production, avec une base solide pour une haute disponibilité et une évolutivité future.
