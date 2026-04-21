# ADR-002 — SQLite local comme backend d'état

- **Date** : 21 avril 2026
- **Statut** : Accepté

## Contexte

L'agent persiste plusieurs catégories de données : missions, steps, step_executions (retries inclus), validations en attente, tokens, audit log append-only. Les contraintes qui pèsent sur le choix du backend :

- **Volumes modestes.** Un dev solo avec 1 à 5 VPS produit quelques milliers de lignes par an, audit log inclus (ordre du Mo/an).
- **Concurrence simple.** L'executor est sérialisé (§8.5 du design) : un seul writer effectif, plusieurs lecteurs (handlers MCP qui répondent à `mission_status`, `mission_logs`).
- **Déploiement.** Cohérent avec l'ADR-001 et la règle 1 du §17.2 : le backend doit être embarqué, pas un service tiers à installer et à superviser.
- **Durabilité.** L'audit log doit survivre à un crash ; les missions `running` doivent être détectables au redémarrage (cf. D2).
- **Backup.** La base doit rentrer dans le backup journalier par simple copie de fichier.
- **Inspection manuelle.** Le porteur doit pouvoir ouvrir la base avec un outil standard et faire des requêtes ad hoc en cas d'incident.

## Décision

L'état de l'agent est persisté dans **SQLite** via le driver pur Go `modernc.org/sqlite`, dans un fichier unique `/var/lib/servagent/state.db` en mode `journal_mode=WAL` et `synchronous=NORMAL`.

## Conséquences

**Positives.** Aucune dépendance externe à installer, configurer, sauvegarder séparément. Driver pur Go : pas de CGO, compilation statique préservée. WAL permet des lectures concurrentes pendant écriture, suffisant pour le modèle single-writer / multi-readers. Backup trivial (copie à chaud ou `.backup`). Inspection en une commande : `sqlite3 /var/lib/servagent/state.db`. Performances largement au-dessus de la charge visée.

**Négatives.** Pas de scalabilité horizontale : une future fédération multi-agents (cf. questions ouvertes du brief §14) imposerait une migration de backend — acceptable, hors objectifs v1. Écritures sérialisées au niveau fichier, ce qui interdit toute future parallélisation de l'executor sans redesign. Requêtes analytiques plus pauvres qu'un Postgres (pas de fenêtrage avancé, pas d'index GIN), non-bloquant pour des cas d'usage transactionnels.

**Neutres.** Dicte un modèle relationnel classique avec FK entre `missions`, `steps`, `step_executions` (§9 du design). Les données semi-structurées sont stockées en TEXT contenant du JSON, lisibles via les fonctions `json_*`. Migrations numérotées append-only, sans rollback (§9.3).

## Alternatives considérées

- **PostgreSQL local** — écarté. Impose un service séparé (package, user, initdb, sauvegarde dédiée), contredit la règle 1 du §17.2. Overkill pour le volume visé.
- **MySQL / MariaDB local** — écarté pour les mêmes raisons que Postgres, avec un argument aggravant : MySQL/MariaDB est justement une *cible* des missions applicatives (l'agent pilote des bases pour les apps de l'utilisateur). Mélanger l'état de l'agent et celui des apps dans le même service créerait un couplage et des problèmes d'isolation.
- **Fichiers plats (JSON/YAML)** — écarté. Pas de transactions, risque de corruption sur crash, requêtes impossibles (filtrer par état, paginer l'audit log), concurrence à réimplémenter à la main.
- **BoltDB / BadgerDB** — écarté. Perte de SQL : plus d'inspection ad hoc via `sqlite3` CLI, plus de JOIN entre missions et step_executions, modèle relationnel qui colle mieux aux tables prévues.
