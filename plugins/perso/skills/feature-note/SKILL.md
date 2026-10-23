---
name: feature-note
description: "Crée (ou met à jour) une note Obsidian récapitulant la feature/le ticket qui vient d'être développé : brief, fonctionnement, config, pièges, tests, déploiement, TODO, et l'ajoute (lien + récap) à la daily note du jour. TRIGGER when: user invokes /feature-note, demande de créer/mettre à jour une note Obsidian sur ce qui a été fait, de documenter la feature terminée dans son Obsidian."
argument-hint: "[numéro-ticket] [titre court]"
allowed-tools: [Bash, Read, Write, Edit, Glob, Grep]
author: NV
---

# Note Obsidian de fin de feature

Produit une note dans le vault Obsidian, rangée au bon endroit, qui résume ce qui a été fait sur la feature en cours. On l'appelle à la fin d'une feature, comme `/commit-msg`.

Vault : `~/Sites/perso/obsidian`

## 1. Récupérer le contexte

```bash
git branch --show-current
git remote get-url origin
git status --short
git diff --stat master...HEAD
git log --oneline master..HEAD
```

- S'il n'y a aucun commit sur la branche, prendre aussi `git diff` / `git diff --staged` (travail non commité).
- Utiliser **la conversation en cours** comme source principale : décisions prises, questions/réponses avec l'utilisateur, pièges rencontrés, tests réalisés et résultats, choses à dire au client. C'est ce qui a le plus de valeur. Le diff ne sert qu'à compléter.
- Relire les fichiers clés modifiés si besoin pour décrire le fonctionnement avec justesse.
- Chercher une estimation/analyse liée dans `var/*.md` du projet et la référencer si elle existe.

## 2. Ticket et titre

- **Numéro de ticket** : `$ARGUMENTS`, sinon extrait de la branche (`issue-824` → `824`, `nv/824-xxx` → `824`). Si aucun n'est trouvé, la note n'a pas de numéro.
- **Lien du ticket** : construit depuis le remote GitLab : `git@git.adveris.fr:clients/cerfrance-2022.git` → `https://git.adveris.fr/clients/cerfrance-2022/-/issues/824`.
- **Titre court** : 2 à 5 mots, en français, qui décrivent la feature. Utiliser celui de `$ARGUMENTS` s'il est fourni.
- **Nom du fichier** : `<ticket> - <Titre court>.md`, par exemple `824 - Tracking bots IA Matomo.md`. Sans ticket : `<Titre court>.md`.

## 3. Trouver le bon dossier

Projets Adveris : `01 - Projects/Boulot/Adveris/Projets/<Client>/`

1. Lister les dossiers de `01 - Projects/Boulot/Adveris/Projets/`.
2. Les faire correspondre avec le nom du repo (dossier courant et remote). Par exemple, `cerfrance-solfege` et `clients/cerfrance-2022` correspondent à `Cerfrance`. Comparer sans tenir compte de la casse et sur des mots partiels.
3. En cas de doute (aucune correspondance, ou plusieurs), demander à l'utilisateur avec la liste des dossiers candidats. Ne jamais créer un nouveau dossier client sans validation.
4. Pour un projet perso ou un sujet non client, proposer `02 - Areas/Dev/Projets/` ou `00 - Inbox/`.

## 4. Note existante ?

Chercher dans le dossier cible une note du même ticket : `ls "<dossier>" | grep -E "^<ticket>([ -]|\.md)"`.
- **Si elle existe** : la lire et la **mettre à jour** avec Edit. Compléter les sections, cocher les TODO faits, ajouter les nouveaux points. Ne pas réécrire ni supprimer ce que l'utilisateur a écrit.
- **Sinon** : la créer.

## 5. Format de la note

S'aligner sur les notes existantes du dossier (en lire une ou deux pour reprendre leurs tags et leur ton). Structure par défaut :

```markdown
---
tags:
  - adveris
  - work
  - dev
  - <client en minuscules>
  - <techno/sujet : symfony, matomo, seo…>
---
### Brief
- [Ticket](<lien>) (branche `<branche>`)
- Le besoin en 1 à 3 puces
- Estimation : `var/xxx.md` (si elle existe)

### Fonctionnement
- Les classes/fichiers clés et leur rôle, en une ligne chacun
- Le flux principal

### Config / Variables d'env
(seulement s'il y en a, dans un bloc de code)

### À savoir
> [!warning]
> Le piège principal ou l'information à transmettre au client

- Les autres pièges et décisions non évidentes, avec leur raison

### Test en local
- Les URLs, commandes et données de test utilisées

### Préprod / Prod
- Les étapes de déploiement spécifiques (migrations, vars d'env, workers, crons…)
- Ce que le client doit faire de son côté

### TODO
- [x] ce qui est fait
- [ ] ce qui reste (relecture, retour client, déploiement…)
```

Règles :
- En français, concis, en puces. C'est une note de travail, pas une doc.
- Omettre les sections vides.
- Ne mettre **aucun secret** : tokens, mots de passe, clés. On peut garder les identifiants de test purement locaux s'ils sont déjà dans `CLAUDE.local.md`.
- Les chemins de fichiers en `code`, et les liens réels uniquement (aucun lien inventé).
- Utiliser les callouts Obsidian (`> [!warning]`, `> [!info]`) avec parcimonie.

## 6. Daily note

Ajouter la feature à la daily note du jour : `02 - Areas/Journal/<YYYY-MM-DD>.md` (date du jour via `date +%F`).

**Si la note n'existe pas**, la créer à partir du template `00 - Templates/Daily note.md` :
- remplacer `{{date:YYYY-MM-DD}}` par la date du jour ;
- ajouter le tag du client dans `tags`.

**Si elle existe**, la lire puis la compléter avec Edit, sans rien supprimer ni réordonner :
- ajouter le tag du client dans `tags` s'il n'y est pas ;
- sous `### Tâches du jour`, ajouter une ligne pour la feature ;
- sous `### Notes rapides`, ajouter le récap.

Si le client a déjà son bloc (`- [Client]` ou `- Client`), ajouter les sous-puces dans ce bloc au lieu d'en créer un second.

Format, aligné sur les daily notes existantes (blocs par client, sous-puces indentées avec une tabulation) :

```markdown
### Tâches du jour
- [Cerfrance]
	- [[824 - Tracking bots IA Matomo]]

### Notes rapides
- Cerfrance
	- [[824 - Tracking bots IA Matomo]] : tracking chatbots IA → Matomo (pilote national), testé en local
	- Reste : relecture, prévenir le client (crawlers non comptés), déploiement préprod
```

Règles :
- Le lien `[[nom de la note sans .md]]` est obligatoire. Le récap tient en 1 à 3 sous-puces : ce qui a été fait, puis ce qui reste ou est bloquant. Le détail reste dans la note de feature.
- Si la feature figure déjà dans la daily (lien présent), mettre à jour son récap plutôt que de le dupliquer.
- La ligne `- [ ]` vide du template peut être remplacée par l'entrée ajoutée.

## 7. Fin

Donner les chemins de la note de feature et de la daily note (créée ou mise à jour), et 2 ou 3 lignes sur leur contenu. Signaler les hypothèses faites, par exemple un ticket déduit de la branche ou un dossier choisi par correspondance.
