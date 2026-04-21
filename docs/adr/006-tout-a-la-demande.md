# ADR-006 — Philosophie "tout à la demande" : rien de pré-installé hors l'agent

- **Date** : 21 avril 2026
- **Statut** : Accepté

## Contexte

À l'install, Servagent pourrait raisonnablement poser une stack applicative par défaut — Nginx, MySQL, PHP, certbot, Composer — pour que le premier déploiement soit immédiat. C'est le modèle des panels d'administration classiques (cPanel, Plesk, Webmin "LAMP bundle") et d'outils comme Easypanel ou CloudPanel. Raisonnable en apparence, incompatible avec plusieurs contraintes fondatrices du projet :

- **Neutralité technologique.** L'utilisateur cible (dev solo, freelance, agence d'une personne) a des stacks hétérogènes : Laravel/MySQL pour un client, FastAPI/Postgres pour un side-project, Node/Docker pour un outil interne. Imposer un choix à l'install, c'est soit le frustrer soit le forcer à désinstaller avant de réinstaller.
- **Auditabilité.** Le modèle du projet est : tout ce qui modifie le système passe par une mission YAML, validée selon son niveau de risque, tracée dans l'audit log. Une install qui pose Nginx + MySQL sans plan YAML crée une exception silencieuse — les premiers paquets installés sur le VPS n'auraient aucune trace dans l'historique des missions.
- **Surface d'attaque minimale.** L'agent tourne en root (cf. ADR-003). Chaque daemon installé par défaut = un service de plus à patcher, une CVE de plus à suivre, un port potentiel de plus exposé. Un VPS "équipé Servagent mais sans app" doit rester aussi nu que possible.
- **Cohérence avec ADR-001/002.** Un binaire statique unique et une DB embarquée. Tirer avec eux une stack applicative à l'install ruinerait la promesse d'installation en secondes et la maîtrise de l'empreinte.

Le §17 du design consigne déjà le raisonnement complet. Cet ADR le fige et le rend citable depuis les futurs PR reviews.

## Décision

L'installation de Servagent dépose **uniquement ce qui est nécessaire au fonctionnement de l'agent lui-même** :

- `/usr/local/bin/servagent` — binaire statique.
- `/etc/systemd/system/servagent.service` — unit systemd.
- `/etc/servagent/config.yaml` — configuration initiale en `600 root:root`.
- `/var/lib/servagent/`, `/var/log/servagent/`, `/var/backups/servagent/` — répertoires d'état.
- `ca-certificates` est installé s'il est absent, indispensable pour ACME et le TLS sortant.

**Rien d'autre.** Pas de webserver, pas de base de données, pas de runtime applicatif (PHP, Node, Python, Ruby, Go), pas de choix implicite entre équivalents (MySQL vs MariaDB, Nginx vs Apache vs Caddy, versions de langage).

Trois règles absolues en découlent, opposables à tout PR :

1. **Pas de pré-installation spéculative.** Aucun paquet applicatif posé par l'installeur ou par l'agent au démarrage.
2. **Chaque mission déclare ses dépendances.** Une mission de déploiement Laravel commence par `apt_update`, `install_package [php8.3-fpm, …]`, `install_package [mysql-server]`, etc., explicitement, visibles dans le YAML et soumis au flow de validation normal.
3. **Aucun tool ne présume de la présence d'un logiciel.** Chaque tool métier vérifie ses prérequis en début d'`Execute()` et retourne une erreur structurée plutôt que de tenter une installation implicite :

```json
{
  "error_code": "prerequisite_missing",
  "missing": ["nginx"],
  "hint": "ajouter un step 'install_package' avec packages=['nginx'] avant cette étape"
}
```

Pas de `apt install nginx` caché dans `create_nginx_vhost`. Pas de `systemctl enable mysql` implicite dans `create_database`.

## Conséquences

**Positives.** Installation en secondes, empreinte ~50 Mo RAM pour un agent au repos sur un VPS "équipé mais vide". Neutralité technologique préservée : l'utilisateur décide mission par mission. Auditabilité maximale — chaque paquet présent sur le système l'est parce qu'une mission l'a demandé et que l'utilisateur l'a validé (en mode `first_time`) ou autorisé via le niveau `confident`. Surface d'attaque minimale : pas de daemon MySQL ouvert sur localhost "au cas où", pas de PHP-FPM qui tourne sans raison. Cohérent avec ADR-003 : moins il y a de services root exposés par défaut, plus le pari "agent en root" est tenable.

**Négatives.** Première mission de déploiement sur un VPS fresh verbeuse — 4 à 5 steps d'installation purement setup avant la première étape métier. Explicite, mais long à lire. L'utilisateur voit défiler des validations `install_package` qui peuvent paraître redondantes les premières semaines — mitigé par le mode `confident` qui les auto-approuve passé la période d'observation (§7.1 du design). Un utilisateur qui arrive avec l'attente "installer le panel et avoir un Nginx qui tourne" sera désorienté la première fois — à documenter frontalement dans le getting-started.

**Neutres.** Justifie l'existence des missions de préparation réutilisables prévues en Phase 7 (`prepare-vps-for-laravel.yaml`, `prepare-vps-for-node.yaml`, `prepare-vps-for-docker.yaml`). Ces missions ne sont pas une feature spéciale ni un contournement de la règle — juste du YAML réutilisable qui respecte le même schéma que les autres et passe par le même flow de validation. L'utilisateur les exécute une fois sur un VPS fresh, puis ses missions applicatives suivantes présument que la stack de base est là (les tools continuent néanmoins de vérifier leurs prérequis — la règle 3 reste absolue, même si elle passe silencieusement).

Les dépendances transitives d'un paquet (`nginx` qui tire `nginx-common`, `libnginx-mod-*`) sont installées par `apt` nativement et ne sont pas considérées comme des actions séparées. Elles sont loggées dans l'`output_json` du step pour traçabilité.

## Alternatives considérées

- **Install "batteries included" (LAMP/LEMP posé par défaut).** Rejeté : impose des choix de stack, verrouille l'utilisateur, augmente la surface d'attaque, casse la cohérence du modèle "tout passe par une mission validée", reproduit les panels existants que Servagent cherche précisément à ne pas être.

- **Install interactive avec sélection de stack au moment du `curl | sudo bash`.** Rejeté : reproduit la friction des installers classiques, sort du modèle "plan YAML", complexité de UX disproportionnée pour un setup qui n'arrive qu'une fois. Si l'utilisateur veut une stack préparée, il exécute une mission de préparation juste après l'install — même workflow, cohérent, auditable.

- **Détection + installation implicite par les tools** (`create_nginx_vhost` qui fait `apt install nginx` en arrière-plan si le binaire est absent, `create_database` qui pose `mysql-server` si nécessaire). Rejeté : viole la règle 3, produit des side-effects invisibles dans le plan YAML. Un plan qui dit "créer un vhost" et qui pose silencieusement Nginx, UFW rules incluses, casse la promesse "le YAML dit tout ce qui se passe". Dégrade aussi la lisibilité de l'audit log : une entrée `create_nginx_vhost` qui a en réalité installé un daemon complet n'est plus atomique.

- **Catalogue d'images / templates de VPS pré-équipés** (un snapshot Hetzner "VPS Servagent + LAMP" prêt à cloner). Rejeté pour le MVP — scope hors projet, dépend d'un fournisseur cloud spécifique, et ne résout pas le fond du problème : l'utilisateur qui installe Servagent sur un VPS existant reste dans le cas nominal. Envisageable en v2+ comme option complémentaire, jamais comme défaut.

- **Deux profils d'install : "minimal" (actuel) vs "stack classique" (avec webserver + DB).** Rejeté : dédouble la surface à maintenir, crée une classe d'utilisateurs avec des systèmes aux paquets "fantômes" non tracés dans l'audit log. Si le besoin se confirme, il est mieux servi par des missions de préparation (Phase 7) que par un flag d'install.
