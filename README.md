# Harbor Registry - Guide de Déploiement

Ce projet fournit une configuration complète de déploiement du registre Harbor. Harbor est un référentiel cloud-native pour les images Kubernetes et conteneurs.

## Prérequis

### Configuration Système
- Docker 20.10.10+ ([Guide d'installation](https://docs.docker.com/engine/install/))
- Docker Compose 1.18.0+ ([Guide d'installation](https://docs.docker.com/compose/install/))
- Hôte Linux avec accès root ou sudo
- Minimum 4 GB de RAM
- Au moins 10 GB d'espace disque libre

### Permissions Requises
Assurez-vous d'avoir les permissions pour :
- Accéder au répertoire `/data/` (volume de données Harbor par défaut)
- Accéder à `/var/log/harbor/` (répertoire des journaux)
- Gérer les services systemd

## Configuration

### 1. Mise à Jour de la Configuration Harbor

Modifiez `harbor.yml` avec vos paramètres d'environnement :

```yaml
hostname: harbor.local
http:
  port: 8080
harbor_admin_password: harborlocaltest
database:
  password: root123
data_volume: /data
```

**Paramètres clés :**
- `hostname` : Nom d'hôte ou adresse IP de votre instance Harbor
- `http.port` : Port HTTP (par défaut : 8080)
- `harbor_admin_password` : Mot de passe administrateur initial
- `data_volume` : Chemin de stockage des données persistantes

### 2. Configuration du Daemon Docker pour un Registre Non Sécurisé

Modifiez ou créez le fichier `/etc/docker/daemon.json` :

```bash
sudo nano /etc/docker/daemon.json
```

Ajoutez la configuration suivante :

```json
{
  "insecure-registries": ["harbor.local:8080"]
}
```

Puis redémarrez Docker :

```bash
sudo systemctl restart docker
```

**Vérifiez la configuration :**
```bash
docker info | grep Insecure
```

### 3. Ajout de la Résolution DNS

Modifiez le fichier `/etc/hosts` pour ajouter Harbor :

```bash
sudo nano /etc/hosts
```

Ajoutez la ligne suivante :

```bash
127.0.0.1 harbor.local
```

**Ou utilisez cette commande :**
```bash
sudo bash -c 'echo "127.0.0.1 harbor.local" >> /etc/hosts'
```

**Vérifiez la résolution DNS :**
```bash
ping harbor.local
nslookup harbor.local
```

### 4. Configuration du Service Systemd

Créez un service systemd pour le démarrage automatique de Harbor au démarrage de la machine hôte.

#### Créez le fichier de service

Créez le fichier `/etc/systemd/system/harbor.service` :

```bash
sudo nano /etc/systemd/system/harbor.service
```

Ajoutez le contenu suivant :

```ini
[Unit]
Description=Harbor Registry Service
Documentation=https://goharbor.io/
Requires=docker.service
After=docker.service

[Service]
Type=simple
Restart=on-failure
RestartSec=5
TimeoutStartSec=0
WorkingDirectory=<votre_repertoire_harbor>
ExecStart=/usr/bin/docker compose -f <votre_repertoire_harbor>/docker-compose.yml up
ExecStop=/usr/bin/docker compose -f <votre_repertoire_harbor>/docker-compose.yml down
StandardOutput=journal
StandardError=journal
SyslogIdentifier=harbor

[Install]
WantedBy=multi-user.target
```

**Remarque :** Remplacez `<votre_repertoire_harbor>` par le chemin réel de votre installation Harbor (exemple : `/home/origins/Bureau/HARBOR`).

#### Activez et démarrez le service

```bash
# Rechargez le daemon systemd
sudo systemctl daemon-reload

# Activez Harbor pour démarrer au boot
sudo systemctl enable harbor.service

# Démarrez Harbor
sudo systemctl start harbor.service

# Vérifiez le statut du service
sudo systemctl status harbor.service

# Visualisez les journaux du service
sudo journalctl -u harbor.service -f
```

## Installation

### Étape 1 : Préparez l'Environnement

```bash
cd <votre_repertoire_harbor>
./prepare
```

Ce script va :
- Valider votre configuration
- Générer les certificats nécessaires
- Préparer l'environnement Harbor

### Étape 2 : Démarrez les Services Harbor

```bash
docker-compose up -d
```

**Suivez le démarrage :**
```bash
docker-compose logs -f
```

### Étape 3 : Vérifiez l'Installation

```bash
# Vérifiez que tous les conteneurs sont en cours d'exécution
docker-compose ps

# Testez la connectivité du registre
curl http://localhost:8080/api/v2/
```

## Accès à Harbor

### Interface Web
- **URL :** `http://harbor.local:8080`
- **Nom d'utilisateur par défaut :** `admin`
- **Mot de passe par défaut :** `harborlocaltest` (tel que configuré dans `harbor.yml`)

### API du Registre
```bash
curl http://harbor.local:8080/api/v2/
```

## Opérations Courantes

### Arrêtez Harbor
```bash
sudo systemctl stop harbor.service
# ou
docker-compose down
```

### Redémarrez Harbor
```bash
sudo systemctl restart harbor.service
```

### Visualisez les Journaux
```bash
# Journaux systemd
sudo journalctl -u harbor.service -f

# Journaux Docker Compose
docker-compose logs -f [nom_du_service]
```

### Poussez une Image vers Harbor
```bash
# Connectez-vous
docker login harbor.local:8080

# Taguez l'image
docker tag monimage:latest harbor.local:8080/library/monimage:latest

# Poussez l'image
docker push harbor.local:8080/library/monimage:latest
```

## Dépannage

### Docker Ne Peut Pas Se Connecter au Registre
```bash
# Vérifiez la configuration insecure-registries
cat /etc/docker/daemon.json

# Vérifiez la résolution DNS
ping harbor.local
nslookup harbor.local

# Testez la connectivité
curl -v http://harbor.local:8080/api/v2/
```

### Le Service Ne Veut Pas Démarrer
```bash
# Vérifiez le statut du service
sudo systemctl status harbor.service

# Visualisez les journaux détaillés
sudo journalctl -u harbor.service -n 50

# Vérifiez Docker Compose
cd <votre_repertoire_harbor>
docker-compose ps
```

### Les Conteneurs Continuent à Redémarrer
```bash
# Vérifiez les journaux
docker-compose logs [nom_du_service]

# Vérifiez l'espace disque
df -h /data

# Vérifiez les permissions
ls -la /data
ls -la /var/log/harbor
```

## Fichiers à Modifier

| Fichier | Description |
|---------|-------------|
| `/etc/docker/daemon.json` | Configuration Docker pour autoriser le registre non sécurisé |
| `/etc/hosts` | Résolution DNS locale pour harbor.local |
| `/etc/systemd/system/harbor.service` | Service systemd pour le démarrage automatique |
| `harbor.yml` | Configuration Harbor locale |
| `docker-compose.yml` | Composition des services Docker |

## Structure du Projet

- `prepare` : Script de préparation de la configuration
- `install.sh` : Script d'installation
- `docker-compose.yml` : Configuration Docker Compose
- `harbor.yml` : Fichier de configuration Harbor
- `common.sh` : Fonctions shell communes
- `common/config` : Modèles de configuration pour les services

## Support

Pour plus d'informations, consultez la [Documentation Harbor](https://goharbor.io/docs/)

---

**Version :** 2.14.0  
**Licence :** Apache License 2.0  
**Date de création :** 4 décembre 2025