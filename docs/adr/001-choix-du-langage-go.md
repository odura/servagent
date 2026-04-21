# ADR-001 — Choix de Go comme langage d'implémentation de l'agent

- **Date** : 21 avril 2026
- **Statut** : Accepté

## Contexte

L'agent tourne en root sur des VPS Ubuntu de production. Plusieurs contraintes discriminent le choix du langage :

- **Distribution sans runtime.** Le §17 du design exige que Servagent ne pose rien d'autre que lui-même. Un binaire unique, statique, cross-compilé `linux-amd64` et `linux-arm64`, est la seule forme acceptable.
- **Empreinte faible.** Cible ~50 Mo RAM pour un agent au repos.
- **Surface technique standard.** HTTPS/JSON-RPC, SQLite, YAML, systemd D-Bus, subprocess, exécution asynchrone des missions.
- **Auditabilité.** Le code doit être relisible sans cérémonie — exclut les langages à forte gymnastique syntaxique ou à macros intrusives.
- **Contrainte humaine.** Porteur solo qui doit apprendre le langage sans compromettre la roadmap — exclut les stacks à très forte courbe.

## Décision

L'agent Servagent et le CLI `servagentctl` sont écrits en **Go 1.23 ou supérieur**.

## Conséquences

**Positives.** Binaire statique unique, sans CGO grâce à `modernc.org/sqlite`. Cross-compilation amd64/arm64 native. Stdlib suffisante (`net/http`, `log/slog`, `crypto/tls`), peu de tiers donc peu de surface d'audit. Goroutines et `context` natifs pour l'executor et les timeouts par step. Écosystème ops mature (systemd, JSON Schema, YAML). Code lisible pour un contributeur externe avec quelques mois de Go.

**Négatives.** Apprentissage requis pour le porteur, mitigé par le mini-projet de chauffe prévu en Phase 0. Gestion d'erreurs verbeuse (`if err != nil`), acceptée comme le prix de la simplicité. Génériques minimalistes — non-bloquant pour le domaine. SDK MCP serveur moins matures qu'en TypeScript au moment de la décision : JSON-RPC 2.0 et le transport "Streamable HTTP" sont implémentés à la main, ce qui reste trivial.

**Neutres.** Verrouille le toolchain (`go`, `gofmt`, `go test`, `golangci-lint`). Le style idiomatique Go guide la structure du code.

## Alternatives considérées

- **Rust** — écarté. Courbe d'apprentissage incompatible avec un démarrage solo, et complexité mémoire non justifiée pour un agent majoritairement I/O-bound.
- **Python** — écarté. Pas de binaire unique natif fiable ; PyInstaller/pex reste fragile pour une install `curl | sudo bash`, et l'empreinte runtime contredit la cible mémoire.
- **Node.js / TypeScript** — écarté. Impose un runtime Node sur le VPS, ce qui viole la règle 1 du §17.2. Les solutions de compilation statique (`pkg`, `deno compile`) sont trop jeunes ou fragiles pour du code qui tourne en root.
