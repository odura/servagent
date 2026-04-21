# ADR-005 — Exécution sérielle des missions

- **Date** : 21 avril 2026
- **Statut** : Accepté

## Contexte

L'executor de missions doit orchestrer des étapes qui touchent à des ressources système partagées et fragiles. Plusieurs d'entre elles tolèrent mal la concurrence, soit par conception, soit parce que leur verrouillage est implicite et imparfait :

- **`dpkg` / `apt`** : lock fichier exclusif par design ; deux `apt install` concurrents se bloquent mutuellement et échouent avec un timeout peu informatif.
- **Nginx** : pas de verrou sur `/etc/nginx/sites-available/`. Deux étapes qui éditent le même vhost simultanément corrompent l'état en silence ; un `nginx -t` passe mais le `reload` peut mettre le serveur dans un état incohérent.
- **MySQL / MariaDB** : la séquence "créer la base → créer l'utilisateur → exécuter les migrations" n'est pas commutative ; lancer deux missions applicatives en parallèle qui visent la même instance MySQL expose aux conflits de noms, aux droits partiellement posés, aux migrations qui se marchent dessus.
- **systemd, cron, UFW, certbot** : chacun a ses propres verrous implicites, et plusieurs ont des effets globaux (reload d'un service, règle de firewall active immédiatement, challenge ACME qui monopolise le port 80).

En face de ces ressources, les contraintes humaines et produit :

- **Porteur solo** : tenir une roadmap exige de ne pas s'enliser dans du design system de concurrence.
- **Utilisateur cible** (dev solo, 1-5 VPS) : ne déploie pas deux apps en parallèle dans la même minute. La latence d'une file est acceptable dans son workflow.
- **Cohérence avec ADR-002** : SQLite WAL est dimensionné pour un single-writer. L'executor pousse des écritures continues (steps, executions, audit) pendant toute la durée d'une mission. Un modèle single-executor s'aligne naturellement avec un backend single-writer.

## Décision

Un seul `Execute()` de mission est actif à la fois sur l'agent, garanti par un `sync.Mutex` au niveau du runtime (cf. design §8.5). Les missions soumises pendant qu'une autre tourne sont acceptées normalement — `mission_submit` reste non-bloquant : il valide le plan, enregistre la mission en base avec l'état `pending_validation` ou `ready`, et retourne immédiatement le `mission_id`. Une goroutine dédiée dépile les missions `ready` en **FIFO stricte**, dans l'ordre de soumission, sans priorité.

Le handler MCP (qui enqueue) et l'executor (qui dépile) sont deux composants distincts, couplés uniquement via la base SQLite. Un `mission_cancel` sur la mission active propage un `context.Cancel` à l'étape en cours puis transitionne la mission en `aborted` ; les missions en file peuvent être `mission_reject` librement sans affecter l'active.

## Conséquences

**Positives.** Aucune classe de bugs de concurrence inter-missions à gérer : le lock `dpkg`, la cohérence de `/etc/nginx/sites-available/`, l'ordonnancement des opérations MySQL sont tous triviaux par construction. L'executor est drastiquement plus simple à écrire, tester et auditer — pas de logique de verrous par ressource, pas de matrice de conflits à maintenir. L'audit log est linéaire : pour une mission donnée, les événements sont strictement ordonnés, ce qui simplifie la lecture par un humain en cas d'incident et l'implémentation des golden files de test (§14.3 du design). Cohérence architecturale avec **ADR-002** : SQLite en mode WAL est pensé pour un writer unique, et l'executor single-threaded s'aligne naturellement sur cette garantie sans effort de coordination supplémentaire. `mission_cancel` n'a qu'une mission à interrompre, ce qui rend son implémentation non-ambiguë.

**Négatives.** Deux missions *indépendantes* (ex. "ajouter un cron de backup sur app A" et "créer un vhost pour un nouveau domaine B sans intersection avec A") sont sérialisées même quand elles ne se marchent pas dessus. Latence subie par l'utilisateur proportionnelle à la durée des missions en file. Un step long (ex. `composer install` qui prend 10 minutes, ou un `request_certificate` qui retry avec backoff) bloque toute la file derrière lui. Pas de parallélisation possible sans redesign significatif de l'executor — acceptable au MVP, à reconsidérer si le profil d'usage réel contredit l'hypothèse "un dev solo ne lance pas de missions parallèles".

**Neutres.** Exige une séparation claire entre le handler MCP de `mission_submit` (qui ne fait que valider + persister + retourner) et la goroutine executor (qui dépile + exécute). Cette séparation est déjà dans le design (§8.1) et ne crée pas de contrainte supplémentaire. La FIFO stricte est un choix explicite et non une contrainte — une politique de priorité (`urgent` vs `normal`) est ajoutable en v0.2 sans breaking change du schéma de mission si le besoin se manifeste.

## Alternatives considérées

- **Exécution concurrente avec locks par ressource.** Chaque outil déclarerait les ressources qu'il touche (`apt`, `nginx.config`, `mysql.global`, `vhost:monapp.fr`, etc.) et le runtime gérerait une matrice de locks pour autoriser la concurrence quand les ressources sont disjointes. Écarté : modéliser finement la granularité des ressources est un design system à part entière (quid des dépendances transitives ? quid des outils qui touchent "tout" implicitement ?). Coût d'implémentation et de tests disproportionné par rapport au gain — un utilisateur qui ne lance pas de missions parallèles n'en voit jamais le bénéfice. Ajoutable en v1.x si le profil d'usage change.

- **Exécution concurrente totale, en laissant `apt` / Nginx / MySQL gérer leurs propres verrous.** Tentant dans sa simplicité apparente. Écarté : le lock `apt` produit des timeouts et des échecs qu'il faudrait ré-encapsuler en retry côté executor, polluant la sémantique des résultats de step. Plus grave : **Nginx ne verrouille pas ses fichiers de config**, donc deux missions qui modifient le même vhost corrompent l'état sans qu'aucune erreur remonte. MySQL gère les verrous au niveau des lignes et des tables, pas au niveau "la base X est en cours de setup, ne la touche pas". Construire un agent de production sur ces verrous implicites est inacceptable.

- **Pool de N workers (concurrence bornée).** Exécute jusqu'à N missions en parallèle, avec N configurable. Écarté : partage tous les problèmes de l'option précédente (races silencieuses sur Nginx, conflits MySQL) sans les résoudre. Introduit un paramètre de tuning (`worker_count`) qui n'a pas de bonne valeur par défaut — 1 équivaut à la décision retenue, 2+ ouvre les races. Ajoute de la surface de configuration sans bénéfice net.

- **FIFO avec priorité (`urgent` / `normal`).** Une file à deux niveaux, les missions `urgent` passent devant. Écarté comme scope creep : aucun cas d'usage concret n'est identifié au MVP. Le FIFO simple suffit pour un dev solo qui, au pire, attend que son déploiement de 10 minutes finisse avant de lancer un cron d'une seconde. Ajoutable en v0.2 sans breaking change du schéma si un besoin réel émerge.

- **File distribuée / workers externes (Redis, NATS, Temporal).** Écarté d'emblée : contredit ADR-001/002 et la règle 1 du §17.2 du design (l'install ne pose rien d'autre que l'agent). Dépendance externe disproportionnée pour un agent mono-machine.
