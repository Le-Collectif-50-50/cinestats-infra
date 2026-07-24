# CineStats Infra — Collectif 50/50

Infrastructure Docker Compose pour le projet CineStats, hébergée sur OVHcloud — **scénario Devis 1B – Éco++** (5 VPS isolés, ~33,54 € HT/mois).

## Périmètre de ce repo

Ce repo gère **uniquement l'infra des 3 VPS-1** : services partagés, monitoring, Metabase et l'environnement de preview.

> ⚠️ **Hors périmètre :** l'application CineStats de production et sa base PostgreSQL de production (les 2 VPS-2) sont gérées ailleurs. Ce repo ne contient ni leur code, ni leur déploiement.

## Architecture cible (Devis 1B)

| Machine | Specs | Rôle | Géré ici ? |
|---|---|---|---|
| **VPS-1 — Services** | 4 vCPU / 8 Go / 75 Go | Traefik (reverse proxy partagé), Monitoring (Prometheus, Grafana, Bugsink), SSO Authentik, Vaultwarden | ✅ |
| **VPS-1 — Metabase** | 4 vCPU / 8 Go / 75 Go | Metabase + PostgreSQL interne + Traefik dédié | ✅ |
| **VPS-1 — Preview** | 4 vCPU / 8 Go / 75 Go | App CineStats en preview + BDD préprod | ❌ (autre repo) |
| **VPS-2 — Prod app** | 6 vCPU / 12 Go / 100 Go NVMe | App CineStats prod + DBT | ❌ (autre repo) |
| **VPS-2 — BDD prod** | 6 vCPU / 12 Go / 100 Go NVMe | PostgreSQL self-managed, pg_dump planifiés | ❌ (autre repo) |

### Répartition des stacks de ce repo

| Stack | VPS cible | Services |
|---|---|---|
| `traefik/` | VPS-1 Services | Reverse proxy partagé, TLS Let's Encrypt |
| `monitoring/` | VPS-1 Services | Prometheus, Grafana, Bugsink, node-exporter, postgres-exporter |
| `authentik/` | VPS-1 Services | SSO/OIDC (server, worker, PostgreSQL, Redis) |
| `vaultwarden/` | VPS-1 Services | Gestionnaire de secrets Bitwarden-compatible |
| `metabase/` | VPS-1 Metabase | Metabase + PostgreSQL interne + Traefik dédié |

## Prérequis

- Docker Engine 24+ et Docker Compose v2 sur chaque VPS
- Noms de domaine pointant vers les IPs OVH (enregistrements A)
- Firewall (UFW) configuré sur chaque VPS — voir [Réseau & firewall](#réseau--firewall)

## Déploiement — VPS-1 Services

Héberge le reverse proxy partagé et tous les services exposés de cette machine.

```bash
docker network create web

cd traefik/
cp .env.example .env       # remplir ACME_EMAIL
touch acme.json && chmod 600 acme.json
docker compose up -d

cd ../authentik/
cp .env.example .env       # AUTHENTIK_DOMAIN + secrets (clé + BDD)
docker compose up -d

cd ../vaultwarden/
cp .env.example .env       # VAULTWARDEN_DOMAIN + ADMIN_TOKEN (+ SMTP optionnel)
docker compose up -d

cd ../monitoring/
cp .env.example .env       # domaines Grafana/Bugsink + credentials + DSN postgres-exporter
docker compose up -d
```

## Déploiement — VPS-1 Metabase

Stack autonome, avec son propre Traefik.

```bash
cd metabase/
cp .env.example .env       # METABASE_DOMAIN + secret d'embedding + BDD interne
touch traefik/acme.json && chmod 600 traefik/acme.json
docker compose up -d
```

## Dev local

Les fichiers `docker-compose.override.yml` (traefik, metabase) désactivent TLS et activent le dashboard Traefik sur `:8080`.

```bash
cd metabase/
cp .env.example .env
docker compose up -d
# Metabase accessible sur http://localhost
```

## Réseaux & firewall

**Réseaux Docker — VPS-1 Services :**
- `web` (externe) : Traefik + services exposés publiquement (Grafana, Bugsink, Authentik, Vaultwarden)
- `monitoring-internal` (interne) : Prometheus + exporters, jamais exposé
- `authentik-internal` (interne) : PostgreSQL + Redis d'Authentik, jamais exposé

**Réseaux Docker — VPS-1 Metabase :**
- `web` (bridge local) : Traefik + Metabase
- `metabase-internal` (interne) : PostgreSQL de Metabase

**Communication inter-VPS — IP publique + firewall :**

Il n'y a pas de réseau privé (vRack) : les échanges entre machines passent par les IP publiques, **restreintes par firewall**. Sur chaque VPS, UFW n'autorise que :
- `80` / `443` ouverts à tous (trafic web derrière Traefik) ;
- `22` (SSH) restreint aux IP d'administration ;
- les ports d'observabilité (PostgreSQL `5432`, node-exporter `9100`, postgres-exporter `9187`) ouverts **uniquement à l'IP du VPS-1 Services (monitoring)**.

Exemple — sur le VPS-2 BDD prod, autoriser le monitoring à scraper Postgres :

```bash
ufw allow from <IP_VPS1_SERVICES> to any port 5432 proto tcp
```

> Le `POSTGRES_EXPORTER_DSN` (stack `monitoring/`) pointe vers la BDD prod (VPS-2) via son IP publique + `sslmode=require`. Les cibles supplémentaires (node-exporter des autres VPS, backend applicatif) s'ajoutent dans `monitoring/prometheus/prometheus.yml`.

## Secrets

Tous les secrets vivent dans des fichiers `.env` (non commités). Les `.env.example` documentent les variables attendues — copier puis remplir avant tout `docker compose up`. Générer les clés/longs tokens avec `openssl rand -hex 32`.
