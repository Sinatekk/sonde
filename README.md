# 🛰️ Sonde — Plateforme de supervision & détection réseau

> Stack tout-en-un de **Network Security Monitoring (NSM)** orchestrée avec Docker Compose : capture réseau (Suricata / Zeek), SIEM Elastic, Threat Intelligence (OpenCTI), analytics web et reverse proxy TLS automatique.

<p align="center">
  <img src="https://img.shields.io/badge/Docker_Compose-2496ED?style=for-the-badge&logo=docker&logoColor=white" alt="Docker Compose">
  <img src="https://img.shields.io/badge/Elastic_Stack-8.14.1-005571?style=for-the-badge&logo=elasticstack&logoColor=white" alt="Elastic Stack 8.14.1">
  <img src="https://img.shields.io/badge/Traefik-v3.1-24A1C1?style=for-the-badge&logo=traefikproxy&logoColor=white" alt="Traefik v3.1">
  <img src="https://img.shields.io/badge/OpenCTI-6.2.0-001F3F?style=for-the-badge&logo=&logoColor=white" alt="OpenCTI 6.2.0">
  <img src="https://img.shields.io/badge/Licence-trial-lightgrey?style=for-the-badge" alt="Licence trial">
</p>

---

## 📑 Sommaire

- [Présentation](#-présentation)
- [Schéma d'architecture](#️-schéma-darchitecture)
- [Stack technique & logos](#-stack-technique--logos)
- [Composants & services](#-composants--services)
- [Prérequis](#-prérequis)
- [Installation](#-installation)
- [Accès aux interfaces](#-accès-aux-interfaces)
- [Structure du projet](#-structure-du-projet)

---

## 🎯 Présentation

**Sonde** déploie une sonde de détection réseau complète et un SOC miniature sur un seul hôte.
Le trafic réseau est capturé par des sondes (**Suricata**, **Zeek**), normalisé puis ingéré
par la **stack Elastic** (Beats → Logstash → Elasticsearch → Kibana). Les indicateurs de
compromission sont gérés dans **OpenCTI**, les logs HTTP visualisés en temps réel via
**GoAccess**, et l'ensemble des interfaces est exposé derrière **Traefik** avec des
certificats Let's Encrypt obtenus automatiquement (DNS-challenge Cloudflare).

Une application volontairement vulnérable (**DVWA**) est incluse pour générer du trafic
d'attaque et tester la chaîne de détection de bout en bout.

| Domaine | Outils |
|---------|--------|
| 🛰️ Capture / Sondes | Suricata, Zeek |
| 📦 Collecte & ingestion | Filebeat, Metricbeat, Logstash, Fleet Server + APM |
| 🔎 SIEM | Elasticsearch, Kibana |
| 🧠 Threat Intelligence | OpenCTI (+ Redis, MinIO, RabbitMQ) |
| 📊 Analytics web | GoAccess + nginx |
| 🌐 Reverse proxy / TLS | Traefik + Cloudflare |
| 🎯 Lab | DVWA |

---

## 🏗️ Schéma d'architecture

```mermaid
flowchart TB
    NET([" Trafic reseau / port miroir "]):::net
    USER([" Analyste SOC "]):::user
    CF([" Cloudflare DNS + TLS "]):::ext

    subgraph PROBES["Sondes reseau (network_mode: host)"]
        SURICATA["Suricata<br/>IDS / IPS"]
        ZEEK["Zeek<br/>Analyse de flux"]
    end

    subgraph SHIP["Collecte & ingestion"]
        FILEBEAT["Filebeat"]
        METRICBEAT["Metricbeat"]
        LOGSTASH["Logstash"]
        FLEET["Fleet Server + APM<br/>(Elastic Agent)"]
    end

    subgraph ELK["Coeur Elastic (SIEM)"]
        ES[("Elasticsearch")]
        KIBANA["Kibana"]
    end

    subgraph CTI["Threat Intelligence"]
        OPENCTI["OpenCTI"]
        WORKER["Workers x3"]
        REDIS[("Redis")]
        MINIO[("MinIO / S3")]
        RABBIT[("RabbitMQ")]
    end

    subgraph WEB["Analytics web"]
        GOACCESS["GoAccess"]
        GANGINX["nginx"]
    end

    subgraph LAB["Cible vulnerable"]
        DVWA["DVWA"]
    end

    TRAEFIK{{"Traefik v3<br/>Reverse proxy + TLS"}}:::proxy

    NET --> SURICATA
    NET --> ZEEK
    SURICATA -->|"eve.json"| FILEBEAT
    ZEEK -->|"logs"| FILEBEAT
    TRAEFIK -->|"access.log"| FILEBEAT
    TRAEFIK -->|"access.log"| GOACCESS

    FILEBEAT --> ES
    METRICBEAT --> ES
    LOGSTASH --> ES
    FLEET --> ES
    ES --> KIBANA

    OPENCTI --> ES
    OPENCTI --- REDIS
    OPENCTI --- MINIO
    OPENCTI --- RABBIT
    WORKER --- OPENCTI
    GOACCESS --> GANGINX

    USER --> CF --> TRAEFIK
    TRAEFIK --> KIBANA
    TRAEFIK --> ES
    TRAEFIK --> OPENCTI
    TRAEFIK --> GANGINX
    TRAEFIK --> DVWA

    classDef net fill:#1f2937,stroke:#60a5fa,color:#fff
    classDef user fill:#0f766e,stroke:#5eead4,color:#fff
    classDef ext fill:#7c2d12,stroke:#fb923c,color:#fff
    classDef proxy fill:#4338ca,stroke:#a5b4fc,color:#fff
```

> 💡 Le diagramme est rendu nativement sur GitHub. Version éditable : [ouvrir sur Mermaid Live](https://l.mermaid.ai/aHUNBA).

**Flux de données :**
1. Les **sondes** Suricata & Zeek écoutent l'interface réseau (`network_mode: host`) et écrivent leurs journaux sur disque.
2. **Filebeat** collecte ces journaux (+ logs Traefik & conteneurs Docker), **Metricbeat** les métriques, **Logstash** les ingestions custom.
3. Tout converge vers **Elasticsearch** (chiffré TLS) et se visualise dans **Kibana**.
4. **OpenCTI** corrèle la threat intelligence et indexe dans Elasticsearch.
5. **Traefik** publie chaque interface en HTTPS sur `*.sonde.dylanlasjunies.fr`.

---

## 🧰 Stack technique & logos

<table>
  <tr>
    <td align="center" width="150">
      <img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/elasticsearch/elasticsearch-original.svg" width="48" height="48" alt="Elasticsearch"/><br/>
      <sub><b>Elasticsearch</b></sub>
    </td>
    <td align="center" width="150">
      <img src="https://www.vectorlogo.zone/logos/elasticco_kibana/elasticco_kibana-icon.svg" width="48" height="48" alt="Kibana"/><br/>
      <sub><b>Kibana</b></sub>
    </td>
    <td align="center" width="150">
      <img src="https://www.vectorlogo.zone/logos/elasticco_logstash/elasticco_logstash-icon.svg" width="48" height="48" alt="Logstash"/><br/>
      <sub><b>Logstash</b></sub>
    </td>
    <td align="center" width="150">
      <img src="https://www.vectorlogo.zone/logos/elastic/elastic-icon.svg" width="48" height="48" alt="Beats / Elastic Agent"/><br/>
      <sub><b>Beats / Fleet</b></sub>
    </td>
  </tr>
  <tr>
    <td align="center">
      <img src="https://suricata.io/wp-content/uploads/2023/09/Logo-Suricata-vert-R.jpg" width="48" height="48" alt="Suricata"/><br/>
      <sub><b>Suricata</b></sub>
    </td>
    <td align="center">
      <img src="https://avatars.githubusercontent.com/u/1194067?s=200&v=4" width="48" height="48" alt="Zeek"/><br/>
      <sub><b>Zeek</b></sub>
    </td>
    <td align="center">
      <img src="https://www.vectorlogo.zone/logos/traefikio/traefikio-icon.svg" width="48" height="48" alt="Traefik"/><br/>
      <sub><b>Traefik</b></sub>
    </td>
    <td align="center">
      <img src="https://www.vectorlogo.zone/logos/cloudflare/cloudflare-icon.svg" width="48" height="48" alt="Cloudflare"/><br/>
      <sub><b>Cloudflare</b></sub>
    </td>
  </tr>
  <tr>
    <td align="center">
      <img src="https://avatars.githubusercontent.com/u/51881218?s=200&v=4" width="48" height="48" alt="OpenCTI"/><br/>
      <sub><b>OpenCTI</b></sub>
    </td>
    <td align="center">
      <img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/redis/redis-original.svg" width="48" height="48" alt="Redis"/><br/>
      <sub><b>Redis</b></sub>
    </td>
    <td align="center">
      <img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/rabbitmq/rabbitmq-original.svg" width="48" height="48" alt="RabbitMQ"/><br/>
      <sub><b>RabbitMQ</b></sub>
    </td>
    <td align="center">
      <img src="https://www.vectorlogo.zone/logos/minioio/minioio-icon.svg" width="48" height="48" alt="MinIO"/><br/>
      <sub><b>MinIO</b></sub>
    </td>
  </tr>
  <tr>
    <td align="center">
      <img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/nginx/nginx-original.svg" width="48" height="48" alt="nginx"/><br/>
      <sub><b>nginx</b></sub>
    </td>
    <td align="center">
      <img src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/goaccess.svg" width="48" height="48" alt="GoAccess"/><br/>
      <sub><b>GoAccess</b></sub>
    </td>
    <td align="center">
      <img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/docker/docker-original.svg" width="48" height="48" alt="Docker"/><br/>
      <sub><b>Docker</b></sub>
    </td>
    <td align="center">
      <img src="https://raw.githubusercontent.com/digininja/DVWA/master/dvwa/images/logo.png" width="48" height="48" alt="DVWA"/><br/>
      <sub><b>DVWA</b></sub>
    </td>
  </tr>
</table>

---

## 🧩 Composants & services

Le `compose.yml` agrège plusieurs fichiers via `include:`. Chaque domaine fonctionnel est isolé dans son propre fichier Compose.

| Fichier | Service(s) | Image | Rôle |
|---------|-----------|-------|------|
| `elasticsearch.compose.yml` | `setup` | `elasticsearch:8.14.1` | Génère la CA & les certificats TLS, initialise les mots de passe |
| | `elasticsearch` | `elasticsearch:8.14.1` | Moteur de stockage & recherche (single-node, TLS) |
| | `kibana` | `kibana:8.14.1` | Visualisation, dashboards, SIEM |
| | `fleet-server` | `elastic-agent:8.14.1` | Gestion des agents + serveur APM |
| `beats.compose.yml` | `filebeat` | `filebeat:8.14.1` | Collecte des logs (Suricata, Zeek, Traefik, Docker) |
| | `metricbeat` | `metricbeat:8.14.1` | Collecte des métriques (ES, Kibana, Logstash, Docker) |
| `logstash.compose.yml` | `logstash` | `logstash:8.14.1` | Pipeline d'ingestion personnalisé |
| `probe.compose.yml` | `suricata` | `jasonish/suricata:master` | IDS/IPS — détection d'intrusion réseau |
| | `zeek` | `blacktop/zeek:elastic` | Analyse passive de flux réseau |
| `opencti.compose.yml` | `opencti` | `opencti/platform:6.2.0` | Plateforme de Threat Intelligence |
| | `opencti-worker` ×3 | `opencti/worker:6.2.0` | Workers d'enrichissement |
| | `opencti-redis` | `redis:7.2.5` | Cache / file de messages |
| | `opencti-minio` | `minio:RELEASE.2024-05-28` | Stockage objet S3 |
| | `opencti-rabbitmq` | `rabbitmq:3.13-management` | Bus de messages AMQP |
| `goaccess.compose.yml` | `goaccess` | `allinurl/goaccess:latest` | Analytics temps réel des logs HTTP |
| | `goaccess-nginx` | `nginx:latest` | Sert le rapport GoAccess |
| `traefik.compose.yml` | `traefik` | `traefik:v3.1` | Reverse proxy + TLS automatique (Cloudflare) |
| `compose.yml` | `dvwa` | `kaakaww/dvwa-docker` | Application web vulnérable (lab) |

### Réseaux & volumes

- **Réseaux :** `elastic` (interne), `traefik` (externe, à créer), `vulnerable` (isolation DVWA).
- **Volumes persistants :** `certs`, `elastic-data`, `kibana-data`, `filebeat-data`, `metricbeat-data`, `logstash-data`, `fleetserver-data`, `goaccess-data`, `redisdata`, `s3data`, `amqpdata`.

---

## ✅ Prérequis

- **Docker** & **Docker Compose v2** (`include:` requiert Compose ≥ 2.20).
- Un hôte Linux avec :
  - `vm.max_map_count=262144` (requis par Elasticsearch) ;
  - une interface réseau en écoute (par défaut `enp0s6`, voir `INTERFACE`) ;
  - ≥ 24 Go de RAM recommandés (ES 12G + Kibana 8G + OpenCTI 8G).
- Un **domaine** géré par **Cloudflare** + un **token DNS API** (pour le challenge ACME).
- Le réseau externe Traefik créé au préalable :
  ```bash
  docker network create traefik
  ```

---

## 🚀 Installation

1. **Cloner le dépôt**
   ```bash
   git clone <repo-url> sonde && cd sonde
   ```

2. **Configurer l'environnement** — copier le gabarit puis renseigner les secrets :
   ```bash
   cp .env.dist .env
   ```
   Variables clés à définir dans `.env` :

   | Variable | Description |
   |----------|-------------|
   | `ELASTIC_PASSWORD` / `KIBANA_PASSWORD` | Mots de passe Elastic (≥ 6 caractères) |
   | `ENCRYPTION_KEY` | Clé de chiffrement Kibana (saved objects, reporting) |
   | `STACK_VERSION` | Version de la stack Elastic (défaut `8.14.1`) |
   | `INTERFACE` | Interface réseau écoutée par Suricata/Zeek (défaut `enp0s6`) |
   | `ES_MEM_LIMIT` / `KB_MEM_LIMIT` / `LS_MEM_LIMIT` | Limites mémoire |
   | `CLOUDFLARE_EMAIL` / `CF_DNS_API_TOKEN` | Identifiants Cloudflare pour le TLS |
   | `OPENCTI_*`, `MINIO_*`, `RABBITMQ_*` | Identifiants OpenCTI & dépendances |

3. **Régler le paramètre kernel** (Elasticsearch) :
   ```bash
   sudo sysctl -w vm.max_map_count=262144
   ```

4. **Démarrer la stack**
   ```bash
   docker compose up -d
   ```
   Le service `setup` génère d'abord la CA et les certificats, puis les autres services
   démarrent en cascade via leurs `healthcheck`.

5. **Suivre le démarrage**
   ```bash
   docker compose ps
   docker compose logs -f setup
   ```

---

## 🌐 Accès aux interfaces

Une fois la stack démarrée, les interfaces sont exposées en HTTPS via Traefik
(remplacez le domaine par le vôtre dans les fichiers Compose et `.env`) :

| Service | URL | Port direct |
|---------|-----|-------------|
| 🔎 Kibana | `https://kibana.sonde.dylanlasjunies.fr` | `5601` |
| 🗄️ Elasticsearch | `https://elastic.sonde.dylanlasjunies.fr` | `9200` |
| 🧠 OpenCTI | `https://opencti.sonde.dylanlasjunies.fr` | `5050` |
| 📊 GoAccess | `https://goaccess.sonde.dylanlasjunies.fr` | `7890` |
| 🎯 DVWA (lab) | `https://vulnerable.sonde.dylanlasjunies.fr` | `8080` |
| 🚦 Traefik Dashboard | `https://traefik.sonde.dylanlasjunies.fr` | — |
| 🛡️ Fleet Server / APM | — | `8220` / `8200` |

> ⚠️ Identifiants par défaut Elastic : utilisateur `elastic`, mot de passe = `ELASTIC_PASSWORD`.

---

## 📁 Structure du projet

```
sonde/
├── compose.yml                  # Point d'entrée — include des sous-fichiers + DVWA
├── elasticsearch.compose.yml    # setup, elasticsearch, kibana, fleet-server
├── beats.compose.yml            # filebeat, metricbeat
├── logstash.compose.yml         # logstash
├── probe.compose.yml            # suricata, zeek
├── opencti.compose.yml          # opencti + redis/minio/rabbitmq + workers
├── goaccess.compose.yml         # goaccess, goaccess-nginx
├── traefik.compose.yml          # traefik (reverse proxy + TLS)
├── networks_volumes.compose.yml # définition des réseaux & volumes
├── .env.dist                    # gabarit de configuration
└── data/
    ├── filebeat.yml             # config Filebeat
    ├── metricbeat.yml           # config Metricbeat
    ├── logstash.conf            # pipeline Logstash
    ├── kibana.yml               # config Kibana
    ├── nginx.conf               # config nginx (GoAccess)
    ├── modules.d/               # modules Filebeat (suricata, zeek, traefik activés)
    ├── suricata/                # suricata.yaml + règles + logs
    ├── zeek/                    # local.zeek + pcap
    └── traefik/                 # letsencrypt, rules, logs
```

---

<p align="center"><sub>Stack de supervision réseau — Suricata · Zeek · Elastic · OpenCTI · Traefik</sub></p>
