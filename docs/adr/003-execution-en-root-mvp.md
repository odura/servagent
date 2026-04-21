# ADR-003 — Exécution de l'agent en root au MVP

- **Date** : 21 avril 2026
- **Statut** : Accepté

## Contexte

L'agent doit piloter l'intégralité d'un VPS Ubuntu de production : `apt` / `dpkg`, écriture dans `/etc/nginx/sites-*` et `/etc/letsencrypt/*`, reload de services systemd, opérations MySQL globales (création de base, de user, dump/restore), règles UFW, lecture de `/var/log/*`, écriture dans `/var/backups/servagent/*`. Toutes ces actions exigent soit root, soit une matrice sudoers très fine et exhaustive.

Construire cette matrice sudoers correctement — sur Ubuntu 24.04 d'abord, puis Debian 12 à partir de la v0.2 — est un chantier à part entière : une règle par commande autorisée, par chemin de fichier, par nom de service systemd, avec des tests de non-régression à chaque ajout de tool. Tenable à long terme, incompatible avec la vélocité d'un porteur solo au MVP.

L'inconfort doit être nommé explicitement plutôt que dissous dans le raisonnement. **Un daemon exposé au réseau qui tourne en root est une cible de grande valeur.** Toute vulnérabilité dans le parsing MCP, la validation de schéma, l'interpolation de variables ou l'un des ~55 tools se traduit en compromission complète de la machine. Cet ADR ne prétend pas rendre ce risque nul — il le compense par un ensemble de couches de mitigation et l'engage sur un chemin de sortie daté.

Le contexte atténuant est réel mais limité : l'utilisateur cible administre son propre VPS, il y est déjà root via SSH, l'agent ne traverse pas une frontière de privilège qui n'existerait pas autrement. L'agent n'est pas un outil multi-tenant qui séparerait des utilisateurs entre eux.

## Décision

Au MVP (v0.1), `servagent` tourne en **root** via son unit systemd (`User=root`).

Le choix est explicitement adossé à plusieurs couches de mitigation déjà décidées ailleurs dans le design, qui deviennent non-négociables du fait de cet ADR :

- **Zones fichiers whitelistées en dur** (§11.5 du design) : tout tool qui manipule des fichiers vérifie que le path cible est dans `/var/www/*`, `/etc/nginx/sites-*`, `/var/lib/servagent/*`, `/var/backups/servagent/*`, `/etc/letsencrypt/*` en lecture. Hors zone → le risque est escaladé à 🟠 automatiquement.
- **Classification de risque obligatoire et non-éludable par le client** (cf. ADR-007) : Claude ne peut pas déclasser une action destructrice. Le registre Go est la seule autorité.
- **Validation utilisateur obligatoire sur 🟠**, double sur 🔴, y compris sur 🟡 en mode `first_time` pendant la période d'observation.
- **Authentification à plusieurs couches** avant tout traitement : IP whitelist (avant CPU bcrypt), bearer token hashé, TLS obligatoire même en local, rate limit global et par IP.
- **Audit log en double écriture** (SQLite WAL + fichier append-only JSONL), défense en profondeur contre une DB compromise ou corrompue.

**Chemin de sortie engagé, avec date.** La **v0.3 (T4 2026, cf. roadmap Phase 6)** migre l'agent vers un user dédié `servagent` avec sudoers strictement whitelisté. Cette migration n'est pas conditionnelle à la demande communautaire : elle est dans la roadmap au même titre que les autres phases. `/etc/servagent/config.yaml` reste `600 root:root` même à ce stade — seul le processus descend de privilèges, pas la configuration.

## Conséquences

**Positives.** Couvre tous les cas d'usage prévus sans explosion de complexité dans les premières phases. Pas de piège sudoers à chaque nouveau tool métier : la surface couverte par le registre peut grandir en Phase 3 et 4 sans refactor de privilèges en parallèle. Delivery du MVP tenable en solo. Modèle simple à documenter pour les premiers utilisateurs : ils savent que l'agent a les mêmes droits qu'eux en SSH.

**Négatives.** Surface d'attaque maximale côté agent assumée. Une RCE dans n'importe quelle couche (parsing JSON-RPC, validation JSON Schema, interpolation `${var}`, exécution d'un tool, bug dans une dépendance Go) se traduit en root sur la machine. Le coût d'une vulnérabilité est binaire : soit rien, soit tout. Cette propriété doit être communiquée honnêtement dans le README et le site public au lancement — un lecteur sécu doit pouvoir comprendre le trade-off sans avoir à décoder les ADRs.

**Neutres.** Rend plusieurs autres décisions du design structurellement obligatoires plutôt que "nice-to-have" : TLS partout (pas de mode HTTP clair, même en local), bearer + IP whitelist, audit log en double, zones fichiers whitelistées, ADR-007 sur la classification non-falsifiable. Cet enchevêtrement est un trait de cohérence, pas un coût additionnel : les mitigations s'appuient les unes sur les autres.

## Alternatives considérées

- **User dédié `servagent` + sudoers whitelist dès le MVP.** L'option de rigueur, rejetée pour le MVP uniquement. Exige d'écrire et de tester une règle sudoers par commande système invoquée par chacun des ~55 tools, avec validation sur au moins deux distributions cibles à partir de la v0.2. Au-delà du volume initial, chaque PR qui ajoute un tool métier touche simultanément au code Go et à la policy sudoers — couplage qui ralentit l'itération sans bénéfice tant que la base d'utilisateurs est petite. Cette option n'est pas rejetée dans l'absolu : c'est littéralement la décision qui remplace celle-ci en v0.3.

- **Containers / namespaces Linux (`bubblewrap`, `unshare`, systemd `DynamicUser`).** Rejeté frontalement. Servagent doit *piloter* le système hôte — installer des paquets, configurer Nginx, recharger systemd. L'isoler du système revient à lui couper les mains. Les capabilities Linux (`CAP_NET_ADMIN` pour UFW, `CAP_DAC_OVERRIDE` pour les fichiers, etc.) couvrent en théorie une partie du besoin, mais en pratique l'ensemble exigé pour la surface d'outils du MVP est tellement large que le gain de confinement devient symbolique.

- **`sudo` passwordless global pour un user non-root.** Rejeté : strictement équivalent à root avec une indirection supplémentaire. Dégrade l'audit (les commandes apparaissent avec un PID intermédiaire, la trace est moins propre) sans réduire les privilèges effectifs. Le pire des deux mondes.

- **Escalade dynamique de privilèges via polkit.** Rejeté pour cause de complexité et de portabilité fragile entre distributions. L'écosystème Go autour de polkit est également peu mature, ce qui obligerait à du code bas niveau spécifique pour un gain de sécurité modeste par rapport à sudoers whitelist (la solution de la v0.3).

- **Deux binaires : un daemon non-root + un helper SUID pour les opérations privilégiées.** Rejeté. Introduit un binaire SUID, qui est précisément la classe de vulnérabilités qu'on veut éviter quand on se préoccupe de l'exécution en root. Déplace le problème sans le résoudre, et ajoute une surface de compilation / packaging supplémentaire.
