# nv-claude

Marketplace Claude Code perso, partagé entre la machine du boulot et la machine perso.

## Contenu

| Plugin | Skill | Usage |
|---|---|---|
| `perso` | `/perso:commit-msg` | Message de commit Conventional Commits à partir des changements |
| `perso` | `/perso:feature-note` | Note Obsidian de fin de feature + ajout à la daily note (vault : `~/Sites/perso/obsidian`) |
| `perso` | `/perso:message` | Message Slack/Teams/mail avec mon ton, selon le destinataire |
| `perso` | `/perso:laravel-db-sync` | Synchronise la BDD locale d'un projet Laravel (rollback des migrations d'autres branches, migrate, diagnostic) |
| `perso` | `/perso:laravel-composer-audit` | Audit CVE Composer (+ npm en option) d'un projet Laravel, update, vérifications et analyse d'impact |
| `perso` | `/perso:laravel-write-tests` | Tests PHPUnit/Pest Laravel selon les conventions du projet, puis exécution |

## Installation sur une machine

```bash
git clone <url-du-repo> ~/Sites/perso/claude
```

Puis dans Claude Code :

```
/plugin marketplace add ~/Sites/perso/claude
/plugin install perso@nv-claude
```

## Mise à jour

```bash
cd ~/Sites/perso/claude && git pull
```

Puis `/plugin marketplace update nv-claude` dans Claude Code, et relancer la session.

## Ajouter un skill ou un agent

- Skill : `plugins/perso/skills/<nom>/SKILL.md` (frontmatter `name` + `description` avec les conditions de déclenchement).
- Agent : `plugins/perso/agents/<nom>.md`.
- Incrémenter `version` dans `plugins/perso/.claude-plugin/plugin.json`.
- Vérifier avec `claude plugin validate .`

Rien de confidentiel boulot (doc interne, clients, secrets) dans ce repo.
