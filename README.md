# Servagent

> **L'agent IA qui transforme un VPS nu en plateforme managée.**
> Un daemon installé sur serveur Linux, pilotable en langage naturel depuis Claude via MCP, avec des garde-fous de niveau production.

---

## État du projet

**Pre-alpha — démarrage avril 2026.**

Ce dépôt contient les fondations d'un projet en construction active. **Le code n'est pas encore utilisable.** La première release installable (`v0.1.0`) est prévue pour août 2026.

Si vous cherchez un outil prêt à l'emploi : repassez plus tard. Si vous êtes curieux de suivre la construction d'un agent IA sysadmin open-source, le dépôt restera public pendant toute la phase de développement.

## De quoi s'agit-il

Servagent est un daemon Go qu'on installe sur un VPS Linux (Ubuntu 24.04 comme cible initiale). Il expose un serveur MCP sécurisé. Depuis son Claude Desktop (abonnement Pro/Max), un développeur solo pilote son serveur en langage naturel : déploiement d'apps, gestion des bases de données, certificats TLS, backups, monitoring.

Deux modes d'interaction complémentaires :

- **Investigation** — Claude appelle des outils read-only pour diagnostiquer l'état du serveur en temps réel.
- **Mission** — Claude génère un plan structuré (YAML validé par JSON Schema), l'utilisateur valide les étapes sensibles, l'agent exécute de manière déterministe avec audit log complet.

Principes structurants : pas de LLM embarqué sur le VPS, pas de dépendance à une API externe côté serveur, pas de stack pré-installée (tout se pose à la demande via des missions). Un binaire Go statique, SQLite pour l'état local, systemd pour le lifecycle.

## Documentation

La doc vit dans le dépôt. Pour comprendre le projet en profondeur :

- [`docs/brief.md`](docs/brief.md) — vision, cible utilisateur, architecture d'ensemble, roadmap, risques.
- [`docs/design.md`](docs/design.md) — architecture technique détaillée (structure du code, protocole MCP, modèle de données, runtime d'exécution).
- [`schemas/mission.schema.json`](schemas/mission.schema.json) — format des plans exécutés par l'agent.
- [`test/fixtures/missions/`](test/fixtures/missions/) — exemples de missions YAML.

Les ADRs (`docs/adr/`) documentent les décisions d'architecture structurantes au fil de l'eau.

## Feuille de route (résumé)

| Phase | Période | Livrable |
|---|---|---|
| Phase 0 | avril 2026 | Fondations : dépôt, docs, squelette Go |
| Phase 1 | mai 2026 | Serveur MCP + outils d'observation |
| Phase 2 | juin 2026 | Runtime d'exécution des missions |
| Phase 3 | juillet 2026 | Bibliothèque d'outils métier |
| Phase 4 | août 2026 | **Lancement `v0.1.0`** |

La roadmap complète (y compris au-delà de v0.1) est dans le brief.

## Licence

MIT. Voir [`LICENSE`](LICENSE).

## Auteur

Projet porté en solo par [@odura](https://github.com/odura).
Site : [servagent.org](https://servagent.org) *(en construction, redirection à venir)*.
