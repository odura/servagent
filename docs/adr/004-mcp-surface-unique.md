# ADR-004 — MCP comme surface unique d'exposition

- **Date** : 21 avril 2026
- **Statut** : Accepté

## Contexte

L'agent doit exposer deux familles de besoins dont la nature diffère :

- Les ~55 **outils atomiques** (observation système, actions métier sur Nginx, MySQL, TLS, etc.) s'expriment naturellement comme tools MCP. Claude les découvre via `tools/list` et les invoque via `tools/call`.
- Une dizaine d'**opérations de contrôle des missions** (`mission_submit`, `mission_approve`, `mission_status`, `mission_logs`, `mission_cancel`…) forment un control-plane qui ressemble davantage à une API de service. Un choix REST classique serait techniquement défendable à ce niveau.

La question n'est donc pas « MCP ou pas MCP » — les outils atomiques *doivent* être MCP pour être utilisables par Claude Desktop — mais « MCP seul, ou MCP + REST pour le contrôle ? ».

Contraintes qui pèsent sur la décision :

- **Un seul type de client identifié au MVP** : Claude Desktop. `servagentctl` tourne en local et peut parler le même protocole que Claude sans coût supplémentaire.
- **Porteur solo** : chaque canal supplémentaire signifie une surface d'auth, de rate-limiting, de logs d'audit et de code de parsing à maintenir en double.
- **Surface d'attaque** : l'agent tourne en root. Minimiser le nombre de points d'entrée est une contrainte de sécurité avant d'être une contrainte de design.
- **Cohérence avec ADR-001/002** : l'install ne pose rien d'autre que l'agent. Un second serveur HTTP interne (même dans le même binaire) ajoute de la surface pour un gain incertain.

## Décision

Tout le trafic applicatif passe par un **serveur MCP unique**, exposé sur `POST /mcp` en JSON-RPC 2.0 sur HTTPS (transport "Streamable HTTP" du spec MCP).

Deux familles de tools y sont publiées :

- **Famille A** — les outils atomiques (~55), utilisables à la fois en mode Investigation et comme briques de missions.
- **Famille B** — les outils de contrôle des missions (`mission_submit`, `mission_approve`, `mission_reject`, `mission_cancel`, `mission_status`, `mission_logs`, `mission_list`, `mission_dry_run`).

Les deux familles partagent le même canal, la même authentification bearer, le même rate-limiting, le même audit log.

**Exception explicite, assumée :** deux endpoints HTTP restent en dehors du canal MCP.

- `GET /healthz` — probe de vivacité, non authentifié, destinée aux outils de monitoring externe (uptime-kuma, watchdog systemd, superviseur tiers) qui ne parlent pas MCP et pour lesquels exiger un handshake JSON-RPC serait absurde.
- `GET /version` — identification du binaire, non authentifié, utile au diagnostic.

Ces deux endpoints sont strictement en lecture, sans état exposé au-delà de `status`, `version`, `uptime`. Ils ne constituent pas un second canal de contrôle : impossible d'y lire une mission ou d'y déclencher une action.

## Conséquences

**Positives.** Surface d'attaque unique : un seul code de parsing, un seul chemin d'authentification, un seul compteur de rate-limit, un seul format d'audit log. Découverte uniforme côté Claude — le contrôle des missions se trouve via le même `tools/list` que les outils atomiques, sans documentation spéciale. Configuration client minimale : un seul serveur MCP à déclarer dans `claude_desktop_config.json`, pas de second champ d'URL à documenter. `servagentctl` parle MCP en local et n'a pas à réimplémenter un client REST parallèle. Évolutions futures plus propres : ajouter des capacités (streams SSE pour les événements de mission, nouveaux tools de contrôle) se fait dans le même canal sans toucher au modèle d'exposition.

**Négatives.** Un intégrateur tiers non-MCP (webhook GitHub Actions, script Bash, Zapier) devra parler JSON-RPC 2.0 au lieu de REST idiomatique. Coût réel mais mineur — JSON-RPC reste du `POST` JSON et se scripte en trois lignes de `curl`. MCP est plus jeune que REST en avril 2026 : le tooling ambiant (Postman collections, proxies de debug, exemples copier-coller sur Stack Overflow) est moins fourni ; on parie sur le fait que cet écart se comble naturellement. L'argument "debug à la main avec `curl`" perd un peu en ergonomie : une requête JSON-RPC est plus verbeuse qu'un `curl -X POST /missions/abc/approve`, même si elle reste parfaitement faisable.

**Neutres.** Ajouter une API REST plus tard reste non-breaking, retirer REST plus tard serait breaking — le choix minimal préserve l'option. L'exception `/healthz` + `/version` est tracée ici pour qu'elle ne dérive pas : toute tentation future d'ajouter un troisième endpoint hors MCP devra passer par un nouvel ADR.

## Alternatives considérées

- **API REST parallèle pour le contrôle des missions, MCP uniquement pour les outils atomiques.** L'alternative la plus sérieuse, celle qu'il faut démolir en détail. Argumentation initiale séduisante : REST est mieux connu, mieux outillé, mieux documenté ; les opérations de contrôle (`submit`, `approve`, `status`) *ressemblent* à des ressources REST ; un intégrateur tiers aurait un chemin plus direct pour déclencher une mission depuis un webhook CI/CD. Raisons du rejet, dans l'ordre de gravité. **(1) Duplication de la surface de sécurité.** Deux canaux = deux chemins d'authentification à maintenir cohérents, deux rate-limiters à synchroniser, deux formats d'audit log à réconcilier quand on voudra répondre à « qui a déclenché cette mission ? ». Pour un agent qui tourne en root, chaque doublon est un risque. **(2) Drift sémantique.** Avec deux canaux, les règles métier (calcul des validations requises, gestion des états de mission, policy de retry) doivent vivre dans une couche commune strictement respectée par les deux handlers. L'expérience dit que ça dérive toujours : un bugfix appliqué côté REST et oublié côté MCP, une validation ajoutée d'un seul côté. Pour un porteur solo, c'est une dette qui s'accumule silencieusement. **(3) Rupture de la découverte unifiée côté Claude.** Si `mission_submit` est un endpoint REST et pas un tool MCP, Claude ne le découvre plus via `tools/list` — il faut l'injecter dans le system prompt, documenter à part, maintenir en parallèle du catalogue d'outils. Le modèle "tout est un tool" est ce qui rend le design lisible pour le LLM. **(4) YAGNI.** Aucun utilisateur du MVP n'est identifié comme incapable de parler MCP. Le scénario "webhook GitHub Actions qui déclenche une mission" est un cas hypothétique de v0.2+ (cf. roadmap Phase 5, "mission hooks"), qu'on adressera si le besoin se concrétise — probablement par un adaptateur léger entrant qui transforme le webhook en appel MCP, pas par une seconde API. **(5) L'argument "debug avec curl" s'effondre.** JSON-RPC 2.0 est intégralement debuggable au `curl` : c'est du `POST` JSON sur un endpoint unique. La différence ergonomique avec REST est réelle mais marginale et ne justifie pas le coût architectural.

- **gRPC.** Écarté : impose protobuf côté client, sort Claude Desktop du jeu (qui parle MCP, pas gRPC), complexifie l'install et le debug. Avantage de perf nul à ce niveau de charge.

- **WebSocket dédié au contrôle des missions.** Écarté : résout un problème (push temps réel des événements de mission) qu'on ne rencontre pas au MVP. Le polling de `mission_status` toutes les 2-5 secondes suffit largement. Si le besoin d'un vrai stream émerge, SSE sur le canal MCP existant est la réponse naturelle et n'exige pas de second transport.

- **SSH + commandes `servagentctl` distantes comme surface de contrôle.** Écarté frontalement. Ramène le modèle du sysadmin manuel (connexion SSH, exécution de commandes, parsing du stdout), qui est précisément ce que le projet cherche à remplacer (cf. brief §1.2). Dégrade aussi l'auditabilité : un shell distant ne produit pas naturellement le log structuré qu'on exige.
