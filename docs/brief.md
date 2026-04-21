# Servagent — Project Brief

> **L'agent IA qui transforme un VPS nu en plateforme managée.**
> Un daemon installé sur serveur Linux, pilotable en langage naturel depuis Claude via MCP, avec des garde-fous de niveau production.

| | |
|---|---|
| **Projet** | Servagent |
| **Site web** | [servagent.org](https://servagent.org) |
| **Dépôt** | [github.com/odura/servagent](https://github.com/odura/servagent) |
| **Licence** | MIT |
| **Statut** | Pre-alpha — démarrage avril 2026 |
| **Porteur** | Solo |
| **Version du brief** | v1.0 — 20 avril 2026 |

---

## 1. Vision

### 1.1 Le problème

Les développeurs solo et les freelances qui hébergent leurs propres apps sur VPS vivent un inconfort permanent :

- Ils savent coder mais n'aiment pas faire du sysadmin.
- Les panels d'administration traditionnels sont lourds, souvent payants, orientés hébergeur multi-clients, et pas pensés IA.
- Les agents IA généralistes (Claude Desktop, Claude Code, OpenClaw) peuvent exécuter des commandes shell mais n'ont aucune connaissance du **contexte métier** d'un serveur d'hébergement web, aucun garde-fou pour la production, et aucun workflow réutilisable.
- Les serveurs MCP existants pour SSH/Shell sont soit trop permissifs (tout autorisé), soit trop restrictifs (whitelist de 10 commandes), sans couche sémantique.

### 1.2 La promesse

> **Donne-moi un VPS nu + un repo Git + un nom de domaine. Je te rends un site en ligne, monitoré, sauvegardé, sécurisé — sans que tu touches à une commande Linux.**

Servagent est un daemon qui s'installe sur un VPS Ubuntu et expose un serveur MCP sécurisé. Depuis son Claude Desktop (abonnement Pro/Max), le développeur pilote son serveur en langage naturel. L'agent exécute des plans structurés (YAML) générés par Claude, avec validation humaine sur les actions sensibles.

### 1.3 Le "wow moment"

> *"Voici mon app Laravel sur GitHub, déploie-la sur https://monapp.fr"*

En une phrase, Servagent orchestre une vingtaine d'actions : clone du repo, détection de la stack, installation des dépendances système, setup de la base MySQL, génération du `.env`, migrations, vhost Nginx, certificat Let's Encrypt, queue workers, scheduler, backups initiaux, vérification finale en HTTPS.

---

## 2. Cible utilisateur

### 2.1 Profil principal

**Le développeur solo propriétaire de 1 à 5 VPS** pour héberger ses propres apps ou celles de ses clients (freelance, indie hacker, agence web d'une personne).

**Caractéristiques** :
- Sait coder, aime coder.
- N'aime pas / ne veut pas devenir sysadmin.
- Utilise déjà Claude Desktop ou Claude Code dans son workflow quotidien.
- Possède un abonnement Claude Pro ou Max.
- Héberge typiquement : sites vitrines clients, apps SaaS en side-project, outils internes, blogs.
- Stack typique : Laravel / Symfony, Node.js, Python (FastAPI, Django), WordPress, apps Docker.

### 2.2 Hors cible (explicite)

- **Sysadmins pros** : ils ont leurs outils (Ansible, Terraform, Chef) et n'ont pas besoin d'une abstraction.
- **Équipes DevOps** : Servagent n'est pas un outil collaboratif (pas de RBAC multi-utilisateurs au MVP).
- **Hébergeurs multi-clients** : pas de logique "comptes isolés" ni d'interface revente.
- **Débutants absolus** : il faut savoir acheter un VPS, faire un `ssh`, comprendre ce qu'est un domaine.

---

## 3. Architecture

### 3.1 Vue d'ensemble

Servagent repose sur une **séparation planner / executor** :

```
┌──────────────────────────────────────────────────────┐
│  Claude Desktop (abonnement Pro/Max du dev)          │
│  ── la tête pensante ──                              │
│                                                      │
│  • Reçoit la demande en langage naturel              │
│  • Investigue via outils MCP read-only               │
│  • Génère un plan YAML structuré                     │
│  • Présente le plan, l'utilisateur valide            │
└──────────────────────────────────────────────────────┘
                    │
                    │  Plan YAML (validé JSON Schema)
                    │  via HTTPS + token bearer
                    ▼
┌──────────────────────────────────────────────────────┐
│  Agent Servagent (daemon Go sur le VPS)              │
│  ── les mains ──                                     │
│                                                      │
│  • Serveur MCP (JSON-RPC 2.0 sur HTTPS)              │
│  • Runtime d'exécution de plans déterministe         │
│  • Bibliothèque d'outils atomiques                   │
│  • État local en SQLite                              │
│  • Audit log immuable                                │
│  • PAS de LLM embarqué                               │
└──────────────────────────────────────────────────────┘
```

**Points clés** :

- Le LLM (Claude) vit **exclusivement** côté utilisateur, sur son abonnement.
- L'agent sur le VPS est **passif et déterministe** : il exécute du YAML, rien d'autre.
- **Pas de dépendance à une API externe** côté VPS : aucun risque de facture qui explose, aucune dépendance à un modèle tiers, résilience maximale aux changements de l'écosystème IA.

### 3.2 Deux modes d'interaction

#### Mode Investigation (live, read-only)

L'utilisateur discute en temps réel avec Claude Desktop. Claude appelle des outils MCP read-only exposés par Servagent pour collecter de l'information.

```
Utilisateur : "Mon site est lent depuis ce matin."

Claude Desktop (live) :
  → get_nginx_access_log(last=1000)
  → get_top_processes()
  → get_mysql_slow_queries()
  → analyse et répond à l'utilisateur
```

Tous les outils appelables en mode investigation sont **garantis sans effet de bord** (niveau 🟢).

#### Mode Mission (plan / execute, différé)

Quand une action est décidée, Claude génère un plan YAML, l'utilisateur valide (selon le niveau de risque), l'agent exécute en autonomie.

```
Utilisateur : "OK, déploie mon app."

Claude Desktop :
  → génère mission.yaml (20 étapes)
  → présente le plan
  → utilisateur valide
  → envoi à l'agent

Agent Servagent :
  → exécute étape par étape
  → audit log de chaque action
  → sur succès : notification de fin
  → sur échec : stop + remontée du contexte à Claude pour replanification
```

L'utilisateur peut **fermer Claude Desktop**, l'agent continue sa mission.

### 3.3 Transition fluide entre les modes

Le workflow typique alterne naturellement :

```
Investigation → Claude identifie l'action → génération du plan
             → Mode Mission → Exécution
             → Investigation post-action (vérification)
```

---

## 4. Sécurité

La sécurité n'est pas une option : Servagent tourne en root sur des serveurs de production. Chaque décision architecturale intègre cette contrainte.

### 4.1 Classification des actions

Chaque outil déclare son niveau de risque à la conception. Cette classification est **codée dans l'agent**, pas dans le plan — Claude ne peut pas reclasser une action.

| Niveau | Validation utilisateur ? | Exemples |
|---|---|---|
| 🟢 **Safe** | Jamais | Lire fichiers/logs, lister services, détection stack, `apt update`, dry-run, vérifications |
| 🟡 **Modifiant réversible** | Auto en mode confiance | Créer vhost, `apt install`, migrations, créer BDD, Let's Encrypt, créer cron |
| 🟠 **Modifiant peu réversible** | **Toujours** | Supprimer fichiers, `DROP` BDD, désinstaller paquet, `systemctl stop`, modifier UFW |
| 🔴 **Destructeur** | **Toujours + double confirmation** | `rm -rf` hors zones whitelist, formater, reboot, modifier SSH/sudoers, désactiver l'agent |

### 4.2 Modes de confiance

- **Mode "première fois"** (actif les 2-3 premières semaines) : validation sur tout, y compris 🟡. Permet à l'utilisateur de calibrer sa confiance.
- **Mode "confiance"** (par défaut après) : validation uniquement à partir de 🟠.
- **Mode "paranoïaque"** (optionnel) : validation sur tout, toujours.

### 4.3 Zones whitelistées

Les opérations fichiers sont contraintes à des zones prédéfinies :
- `/var/www/*` (sites web)
- `/etc/nginx/sites-available/*`, `/etc/nginx/sites-enabled/*`
- `/etc/letsencrypt/*` (lecture)
- `/var/lib/servagent/*` (état de l'agent)
- `/var/backups/servagent/*` (backups)

Tout accès hors zone = niveau 🟠 automatique.

### 4.4 Isolation et exécution

- **MVP** : l'agent tourne en `root` pour couvrir l'intégralité des cas d'usage sans complexité sudoers.
- **v1.x** : migration vers un user dédié `servagent` avec `sudoers` strictement whitelisté.

### 4.5 Exposition réseau

- **HTTPS obligatoire** (TLS auto via Let's Encrypt si domaine configuré, certificat auto-signé sinon).
- **Port dédié** : `7823` par défaut (configurable).
- **Authentification** : token bearer généré à l'installation, stocké dans `/etc/servagent/config.yaml` (permissions `600`).
- **IP whitelist** : l'utilisateur doit explicitement ajouter son IP pour que l'agent accepte des connexions.
- **Rate limiting** : 60 requêtes/minute par défaut.

### 4.6 Audit log immuable

Chaque action exécutée est loggée dans SQLite avec : timestamp, outil appelé, paramètres, utilisateur initiateur (hash du token), résultat, durée, mission ID le cas échéant. Les logs sont également écrits en append-only dans `/var/log/servagent/audit.log`.

---

## 5. Format des plans (Mission YAML)

### 5.1 Structure

Les plans sont du YAML validé par JSON Schema (`schemas/mission.schema.json` dans le dépôt).

**Exemple simplifié** :

```yaml
mission_id: deploy-monapp-2026-04-20-abc123
version: 1
created_by: claude-desktop
created_at: 2026-04-20T14:30:00Z
goal: "Déployer github.com/odura/monapp sur monapp.fr"

context:
  vps: prod-web-01
  domain: monapp.fr
  repo: https://github.com/odura/monapp

steps:
  - id: clone-repo
    tool: git_clone
    risk: 🟡
    params:
      repo: https://github.com/odura/monapp
      dest: /var/www/monapp
    on_error: abort

  - id: detect-stack
    tool: detect_stack
    risk: 🟢
    params:
      path: /var/www/monapp
    output_var: stack_info

  - id: install-php
    tool: install_package
    risk: 🟡
    depends_on: [detect-stack]
    params:
      packages: ["php8.3-fpm", "php8.3-mysql", "php8.3-mbstring"]

  # ... autres étapes

  - id: verify-https
    tool: http_check
    risk: 🟢
    params:
      url: https://monapp.fr
      expected_status: 200
```

### 5.2 Propriétés clés

- **Signé** : l'agent vérifie que le plan vient bien du client authentifié.
- **Versionné** : conservé en SQLite avec son résultat d'exécution complet.
- **Reprenable** : en cas d'échec, Claude peut générer un plan de reprise qui pointe vers le `mission_id` parent.
- **Dry-run possible** : chaque plan peut être exécuté en simulation (rapport des actions qui *seraient* prises, sans les exécuter).

---

## 6. Bibliothèque d'outils MVP

~55 outils atomiques couvrant 10 domaines. Chaque outil déclare son niveau de risque, ses paramètres (JSON Schema), et son format de sortie.

### 6.1 Observation système (🟢)
`get_system_info`, `get_resources`, `list_processes`, `get_process_tree`, `list_services`, `get_service_status`, `get_service_logs`, `get_file`, `list_directory`, `search_files`

### 6.2 Web servers (Nginx / Apache)
`list_vhosts` 🟢, `get_vhost_config` 🟢, `create_nginx_vhost` 🟡, `enable_vhost` 🟡, `disable_vhost` 🟠, `reload_webserver` 🟡, `test_webserver_config` 🟢

### 6.3 Bases de données (MySQL/MariaDB + PostgreSQL)
`list_databases` 🟢, `get_database_size` 🟢, `create_database` 🟡, `create_db_user` 🟡, `dump_database` 🟡, `restore_database` 🟠, `drop_database` 🟠, `get_slow_queries` 🟢, `run_sql_query` 🟠

### 6.4 TLS / Certificats
`list_certificates` 🟢, `get_certificate_info` 🟢, `request_certificate` 🟡, `renew_certificate` 🟡, `revoke_certificate` 🟠

### 6.5 Paquets & runtimes
`list_installed_packages` 🟢, `get_package_info` 🟢, `apt_update` 🟢, `install_package` 🟡, `remove_package` 🟠, `detect_stack` 🟢, `get_runtime_versions` 🟢

### 6.6 Docker
`list_containers` 🟢, `list_images` 🟢, `container_logs` 🟢, `compose_up` 🟡, `compose_down` 🟠, `compose_restart` 🟡, `pull_image` 🟢

### 6.7 Git & déploiement
`git_clone` 🟡, `git_pull` 🟡, `git_status` 🟢, `git_log` 🟢, `run_script` 🟡

### 6.8 Cron & schedulers
`list_crons` 🟢, `add_cron` 🟡, `remove_cron` 🟠, `create_systemd_timer` 🟡

### 6.9 Firewall (UFW)
`ufw_status` 🟢, `ufw_allow_port` 🟡, `ufw_deny_port` 🟠, `ufw_reload` 🟡

### 6.10 Backup & fichiers
`list_backups` 🟢, `create_backup` 🟡, `restore_backup` 🟠, `delete_backup` 🟠, `write_file` 🟡, `delete_file` 🟠, `chmod` 🟠, `chown` 🟠, `set_www_permissions` 🟡

### 6.11 Notifications sortantes (🟢)
`notify_webhook`, `notify_email`

### 6.12 Hors MVP (v1.x et après)

- Multi-serveurs / flotte
- Gestion users Linux
- Mail server (Postfix, Dovecot, DKIM)
- DNS management (API OVH, Cloudflare)
- Templates d'apps spécifiques (WordPress one-click)
- Supervisord, queues avancées
- Kubernetes
- Rollback automatique multi-étapes

---

## 7. Stack technique

### 7.1 Côté agent (VPS)

| Composant | Choix | Justification |
|---|---|---|
| Langage | **Go** (1.23+) | Binaire statique unique, empreinte RAM faible (~50 Mo), cross-compile natif ARM, écosystème ops mature, audit facile |
| Serveur HTTP | `net/http` stdlib | Suffit largement, pas de framework |
| Base de données | **SQLite** (`modernc.org/sqlite`, pure Go) | État local, zéro dépendance externe |
| YAML + Schema | `goccy/go-yaml` + `santhosh-tekuri/jsonschema` | Validation stricte des plans |
| systemd | `coreos/go-systemd` | Interaction avec les units |
| Logs structurés | `log/slog` stdlib | Moderne, idiomatique |
| Config | YAML + variables d'env | Simple, versionnable |

### 7.2 Distribution cible

- **MVP (v0.1)** : Ubuntu 24.04 LTS uniquement.
- **v1.0** : Ubuntu 22.04 + 24.04 + Debian 12.
- **v2.0** : Rocky Linux / AlmaLinux (à évaluer selon demande).

### 7.3 Côté client

Pas de client spécifique à développer au MVP. L'utilisateur configure directement **Claude Desktop** via `claude_desktop_config.json` :

```json
{
  "mcpServers": {
    "mon-vps": {
      "url": "https://mon-vps.fr:7823",
      "headers": {
        "Authorization": "Bearer <token-généré-à-l-install>"
      }
    }
  }
}
```

Un CLI `servagent-cli` pour piloter depuis le terminal pourra être ajouté en v1.x.

---

## 8. Distribution & installation

### 8.1 Workflow de release

Automatisé via GitHub Actions. Un `git tag v0.1.0` déclenche :

1. Build cross-platform : `linux-amd64` + `linux-arm64`
2. Génération des checksums SHA256
3. Création de la Release GitHub
4. (v1.x) Publication sur repo APT + image Docker

### 8.2 Installation utilisateur

**Une seule commande** :

```bash
curl -fsSL https://get.servagent.org | sudo bash
```

Le script :
1. Détecte l'architecture (amd64 / arm64)
2. Vérifie la distribution (Ubuntu/Debian supportées)
3. Télécharge le binaire depuis GitHub Releases
4. Vérifie le checksum
5. Génère un token d'authentification
6. Crée `/etc/servagent/config.yaml` avec une config initiale
7. Installe le service systemd
8. Affiche le token et les prochaines étapes

### 8.3 Mises à jour

- **MVP** : `sudo servagent update` (auto-update intégré, pattern Tailscale)
- **v1.x** : repo APT (`apt upgrade`)

---

## 9. Stratégie de lancement

### 9.1 Approche graduée (dévoilée progressivement)

| Phase | Visibilité | Objectif |
|---|---|---|
| **Semaines 1-2** | Privé | Poser l'archi, apprendre Go, coder sans pression |
| **Semaines 3+** | Public silencieux (pas d'annonce) | Le code est visible, le README minimal, pas de communication |
| **Semaines 5-16** | Build in public progressif | Posts LinkedIn/X/blog sur les décisions techniques, les avancées, les défis |
| **Lancement v0.1.0** | Annonce officielle | Post de lancement, démo vidéo, activation de `get.servagent.org` |

### 9.2 Canaux de communication personnels

- **Dépôt GitHub** : `github.com/odura/servagent` — historique des décisions visible.
- **Site** : `servagent.org` — landing + docs.
- **LinkedIn / X** : devlog régulier.
- **Blog technique** (optionnel) : articles de fond sur les choix d'architecture.

L'objectif explicite est de **construire la visibilité personnelle du porteur en parallèle du produit**.

### 9.3 Licence

**MIT**. Permet l'adoption maximale, l'utilisation par des agences sans friction juridique, les forks et contributions.

---

## 10. Roadmap

### Phase 0 — Fondations (semaines 1-2 · avril 2026)

- Setup du dépôt privé `github.com/odura/servagent`
- Mini-projet Go de chauffe (parser de logs nginx) pour prise en main
- Architecture détaillée : structure du code, interfaces clés, modèle de données SQLite
- Choix des libs principales
- Squelette du projet : `cmd/`, `internal/`, `schemas/`, `scripts/`, `systemd/`
- Tests unitaires basiques en place

**Livrable** : squelette compilable, CI GitHub Actions verte, aucune fonctionnalité utilisateur.

### Phase 1 — MCP server + outils d'observation (semaines 3-6 · mai 2026)

- Implémentation du serveur MCP (JSON-RPC 2.0 sur HTTPS)
- Authentification token bearer + IP whitelist
- Bibliothèque complète des outils 🟢 (observation : ~15 outils)
- Intégration Claude Desktop testée
- Dépôt basculé en **public silencieux** (README minimal)
- Premier post LinkedIn : "Je démarre Servagent"

**Livrable** : Servagent utilisable en **mode Investigation** read-only. Démontrable mais pas "utile" en autonomie.

### Phase 2 — Runtime d'exécution des missions (semaines 7-10 · juin 2026)

- Validateur YAML + JSON Schema
- Executor séquentiel avec gestion `on_error`, `depends_on`, `output_var`
- Audit log SQLite + fichier append-only
- Classification 🟢🟡🟠🔴 implémentée
- Mécanisme de validation utilisateur (callback vers Claude Desktop via MCP)
- Dry-run mode

**Livrable** : première mission exécutable de bout en bout (par exemple : "créer une BDD MySQL + un user + sauvegarder les credentials").

### Phase 3 — Outils métier (semaines 11-14 · juillet 2026)

- Outils Web servers (Nginx) : 7 outils
- Outils BDD (MySQL/MariaDB) : 9 outils
- Outils TLS (certbot) : 5 outils
- Outils Git & scripts : 5 outils
- Outils paquets & runtimes : 7 outils
- Devlog mensuel, premiers retours externes informels (cercle de confiance)

**Livrable** : le "wow moment" (déploiement Laravel complet) fonctionne en démo.

### Phase 4 — Polish & lancement v0.1.0 (semaines 15-18 · août 2026)

- Outils Docker, Cron, UFW, Backup : ~20 outils
- Script d'installation `install.sh` + domaine `get.servagent.org`
- Site `servagent.org` minimal (landing + install + premier guide)
- Documentation utilisateur : getting started, référence des outils, exemples de missions
- Tests d'installation sur 3-5 VPS fresh (Hetzner, OVH, Scaleway)
- Vidéo de démo du "wow moment"
- Post de lancement coordonné (LinkedIn, X, Hacker News, r/selfhosted)

**Livrable** : **v0.1.0 publique**, installable en une commande, démontrable.

### Phase 5 — v0.2 : consolidation (septembre-octobre 2026)

- Support Debian 12
- Auto-update intégré (`servagent update`)
- Mode "mission hooks" (webhooks entrants pour déclencher des missions depuis CI/CD)
- Amélioration de la replanification sur échec
- Premières contributions externes (si la communauté émerge)

### Phase 6 — v0.3 : sécurité renforcée (novembre-décembre 2026)

- Migration de `root` vers user dédié `servagent` + sudoers strict
- Rotation automatique des tokens
- Support Tailscale natif (détection auto, pas d'IP whitelist nécessaire)
- Signature GPG des releases

### Phase 7 — v1.0 : stable (Q1 2027)

- Repository APT officiel
- Documentation complète
- Bibliothèque de "missions types" réutilisables (déploiement Laravel, WordPress, FastAPI, Node.js, etc.)
- Couverture de tests > 70%

### Au-delà — v2.0 et plus

- Multi-serveurs (un client qui pilote N agents)
- Interface web locale (dashboard sur le port 7823 pour visualiser l'état sans Claude)
- Rollback automatique
- Templates applicatifs (WordPress one-click, etc.)
- Support Rocky/Alma
- Mode "marketplace de missions" (partage de plans YAML entre utilisateurs)

---

## 11. Critères de succès

### À 3 mois (v0.1.0 lancée)

- Servagent installable en une commande sur Ubuntu 24.04
- Le "wow moment" (déploiement Laravel) fonctionne en démo
- Au moins 3 VPS en production réelle (le porteur + 2 autres)
- Premier post de lancement publié
- 50+ stars GitHub

### À 6 mois (v0.3)

- 10-50 utilisateurs actifs identifiés
- 500+ stars GitHub
- Au moins 3 contributeurs externes
- Servagent mentionné dans au moins 2 posts tiers (blog, newsletter tech)

### À 12 mois (v1.0)

- Servagent connu dans la communauté dev solo / indie hackers
- 200-500 installations actives (métriques anonymes opt-in)
- 2000+ stars GitHub
- Le porteur est identifié comme figure dans l'écosystème MCP / agents IA sysadmin

---

## 12. Risques & mitigations

| Risque | Probabilité | Impact | Mitigation |
|---|---|---|---|
| Anthropic ajoute des "skills sysadmin" natives à Claude Code | Moyenne | Élevé | Positionnement sur les workflows métier haut niveau + agent persistant + audit, ce que Claude Code ne fait pas |
| OpenClaw ou concurrent pivote vers le sysadmin | Moyenne | Élevé | Spécialisation hébergement web (pas agent généraliste), communauté MIT ouverte |
| Faille de sécurité critique (l'agent tourne en root) | Faible mais critique | Catastrophique | Audit log, zones whitelistées, validation obligatoire 🟠🔴, migration user dédié en v0.3 |
| Apprentissage Go plus lent que prévu | Faible | Moyen | 40 ans de dev, Go très accessible, mini-projet de chauffe prévu |
| Manque de temps / projet solo qui s'essouffle | Moyenne | Élevé | Approche graduée (pas de pression publique avant le MVP), build in public comme motivation |
| Faible adoption au lancement | Moyenne | Moyen | Communauté construite progressivement en amont, positionnement personnel fort |
| MCP protocol évolue et casse la compatibilité | Faible | Moyen | Versionning strict, tests d'intégration réguliers avec Claude Desktop |

---

## 13. Décisions prises (récap)

| Décision | Choix |
|---|---|
| Positionnement | Agent IA d'administration de VPS, pour développeurs solo |
| Architecture IA | Planner (Claude Desktop) / Executor (agent VPS) — pas de LLM sur VPS |
| Deux modes | Investigation (live, read-only) + Mission (plan/execute différé) |
| Format des plans | YAML + JSON Schema |
| Langage | Go |
| Stockage état | SQLite local |
| Exposition réseau | HTTPS + token bearer + IP whitelist, port 7823 |
| Distribution MVP | Ubuntu 24.04 LTS uniquement |
| Exécution | root (MVP), user `servagent` (v0.3+) |
| Classification risque | 🟢 🟡 🟠 🔴 avec validation auto en mode confiance |
| Licence | MIT |
| Open-sourcing | Approche graduée (privé → public silencieux → build in public → launch) |
| Nom | Servagent |
| Dépôt | github.com/odura/servagent |
| Site | servagent.org |

---

## 14. Questions ouvertes (à trancher plus tard)

- Faut-il un CLI `servagent-cli` pour piloter hors Claude Desktop ? (probablement v1.x)
- Stratégie de monétisation long terme : formation ? support payant ? version "team" propriétaire ? (à évaluer après v1.0)
- Intégration avec les fournisseurs cloud (API OVH, Hetzner) pour automatiser l'achat de VPS ? (v2+)
- Servagent peut-il piloter un autre Servagent (fédération de VPS) ? (v2+)
- Faut-il un mécanisme de "templates communautaires" de missions ? (v2+)

---

*Document vivant — à mettre à jour au fur et à mesure des décisions.*
*Dernière mise à jour : 20 avril 2026.*
