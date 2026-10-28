---
name: laravel-db-sync
description: "Synchronise la BDD locale d'un projet Laravel avec le code : status des migrations, détection et rollback des migrations inconnues (jouées sur une autre branche), preview SQL (--pretend), migrate, diagnostic des erreurs. TRIGGER when: user invokes /laravel-db-sync, demande de vérifier/lancer les migrations Laravel, synchroniser la base, lancer migrate / migrate:status / migrate:rollback, ou signale une base pas à jour après un changement de branche sur un projet Laravel."
allowed-tools: [Bash, Read, Grep, Glob, AskUserQuestion]
author: NV
---

# Laravel DB Sync

Remet la BDD locale en phase avec les migrations du projet Laravel courant.

**Règle d'or : aucune action destructive sans validation explicite de l'utilisateur** (`migrate:rollback`, `migrate:fresh`, `migrate:refresh`, `migrate:reset`, `db:wipe`, `DELETE` / `UPDATE` de données, `DROP`, insertion ou suppression manuelle dans la table `migrations`). Seul `migrate` (les migrations "up" en attente) est lancé directement : c'est le but du skill.

**Jamais** `migrate:fresh`, `migrate:refresh`, `migrate:reset` ni `db:wipe` de sa propre initiative : ils vident la base. Les proposer uniquement si l'utilisateur le demande, avec dump préalable.

Toujours passer `--no-interaction` aux commandes artisan. Ne **jamais** ajouter `--force` (il sert à passer outre la protection de l'env `production`) : si artisan refuse parce que `APP_ENV=production`, stop et le signaler.

---

## 1. Vérifier que c'est un projet Laravel

```bash
test -f artisan && test -f composer.json && grep -q '"laravel/framework"' composer.json && echo LARAVEL_OK
grep -o '"laravel/framework": *"[^"]*"' composer.json
```

- Pas Laravel → stop : « Ce dossier n'est pas un projet Laravel (pas de `artisan` / `laravel/framework`). »
- Noter la version majeure (certaines options en dépendent : `migrate:status --pending` ≥ 10, `db:show` natif ≥ 11, `migrate:rollback --batch` ≥ 11).

---

## 2. Choisir la commande artisan (`$ARTISAN`)

Dans la suite, `$ARTISAN` désigne la commande retenue (à réécrire en entier dans chaque appel Bash, le shell ne persiste pas les variables).

0. **Consignes du projet d'abord** (elles priment sur la détection) :
   ```bash
   ls CLAUDE.md CLAUDE.local.md .claude/CLAUDE.md 2>/dev/null
   ```
   Y chercher la commande artisan (`php artisan`, `docker exec <conteneur> php artisan`, `docker compose run --rm <service> php artisan`, `./vendor/bin/sail artisan`, cible `make`…), le service/conteneur BDD et toute info utile (dump, base de test…). Un alias shell n'est pas disponible en shell non interactif : utiliser la commande qu'il désigne.
   Si une commande y est indiquée → la retenir et passer directement au test (point 4).
1. **Laravel Sail** : `test -x vendor/bin/sail && grep -q 'laravel/sail' composer.json` → `./vendor/bin/sail artisan` (vérifier que les conteneurs tournent : `./vendor/bin/sail ps`).
2. **Docker** : `ls docker-compose*.y*ml compose*.y*ml Makefile 2>/dev/null` puis `grep -nE 'artisan|docker' Makefile | head -20`. Si un service PHP existe → préférer `docker compose exec <service> php artisan` (conteneur déjà lancé, sinon `docker compose run --rm <service> php artisan`). Vérifier avec `docker ps` que le conteneur est up. Penser que `mysqldump` (étape 6) passera alors par le conteneur BDD.
3. **PHP local** : déterminer la version attendue (`.php-version`, sinon `require.php` de `composer.json`) et la comparer à `php -v`.
   - Compatible → `php artisan`.
   - Sinon, chercher un binaire de la bonne version et le résoudre en **chemin complet** (les alias `.zshrc` ne marchent pas ici) :
     ```bash
     command -v herd && herd which-php 2>/dev/null                       # Laravel Herd
     compgen -c 2>/dev/null | grep -E '^php[0-9]+(\.[0-9]+)?$' | sort -u   # php8.3, php83…
     grep -hE '^\s*alias\s+php[0-9.]*=' ~/.zshrc ~/.bashrc ~/.bash_aliases 2>/dev/null
     command -v mise && mise where php@8.3 2>/dev/null
     ls /opt/homebrew/opt/php@*/bin/php /usr/local/opt/php@*/bin/php \
        ~/Library/Application\ Support/Herd/bin/php* 2>/dev/null           # macOS
     ls /usr/bin/php[0-9]* /usr/local/bin/php[0-9]* 2>/dev/null           # Linux
     ls /c/laragon/bin/php/*/php.exe /c/xampp/php/php.exe 2>/dev/null     # Windows (Git Bash)
     ```
     Vérifier chaque candidat avec `<chemin> -v` → ex. `/opt/homebrew/opt/php@8.3/bin/php artisan`.
   - Rien trouvé : ne pas deviner, demander à l'utilisateur (AskUserQuestion) : continuer avec `php`, ou indiquer le chemin (ex. `! which php8.3`).
4. Annoncer la commande retenue et sa source, puis tester :
   ```bash
   $ARTISAN --version --no-interaction
   $ARTISAN migrate:status --no-interaction | head -5
   ```
   Si la connexion échoue (serveur arrêté, mauvais credentials, base inexistante, `getaddrinfo` sur un host Docker lancé depuis le PHP local…) → afficher l'erreur brute, proposer la piste (`docker compose up -d`, `brew services start mysql`, `sudo systemctl start mysql`, créer la base, lancer artisan depuis le conteneur si `DB_HOST` est un nom de service Docker) et **stop**.

   Si `Migration table not found` : la base est vierge. Le signaler ; si un `database/schema/<connexion>-schema.sql` existe, prévenir que `migrate` le chargera d'abord (schema dump). Puis passer à 3.4 (migrate) avec accord.

---

## 3. Migrations

### 3.1 État des lieux

```bash
$ARTISAN migrate:status --no-interaction
```

Puis lister les **migrations exécutées inconnues** (présentes dans la table `migrations` mais sans fichier dans aucun chemin enregistré — y compris ceux des packages/modules chargés via `loadMigrationsFrom`) :

```bash
$ARTISAN tinker --no-interaction --execute='$m = app("migrator"); $files = $m->getMigrationFiles(array_merge([database_path("migrations")], $m->paths())); $ran = $m->getRepository()->getMigrations(PHP_INT_MAX); foreach ($ran as $r) { if (!isset($files[$r->migration])) echo $r->batch."\t".$r->migration.PHP_EOL; }'
```

(Si le projet utilise plusieurs connexions avec leurs propres migrations, répéter avec `--database=<connexion>` sur les commandes artisan et le `setConnection()` du repository.)

Présenter un résumé court : nombre de migrations en attente (avec leurs noms), migrations inconnues (avec leur batch). Si tout est à jour et rien d'inconnu → récap (étape 7).

### 3.2 Migrations exécutées inconnues → retrouver l'origine et proposer le rollback

Traiter **avant** les migrations en attente, dans l'ordre **inverse** (batch le plus élevé puis nom décroissant).

**D'abord, classer toutes les inconnues en bloc** (il peut y en avoir des dizaines sur un vieux projet ou une base importée d'un autre environnement) :

```bash
for n in <NAME1> <NAME2> ...; do
  all=$(git log --all --format='%h' -- "*$n.php" | head -1)
  del=$(git log --format='%h' --diff-filter=D -- "*$n.php" | head -1)
  renamed=$(find . -path ./vendor -prune -o -path ./node_modules -prune -o -name "*_${n#????_??_??_??????_}.php" -print 2>/dev/null | head -1)
  echo "$n | all=$all | deleted_on_HEAD=$del | similar=$renamed"
done
```

| Catégorie | Critère | Traitement |
|---|---|---|
| **Autre branche** | trouvée dans `--all` mais pas supprimée dans l'historique de HEAD | Traitement individuel ci-dessous (rollback proposé) |
| **Supprimée sur la branche courante** | `deleted_on_HEAD` renseigné | Obsolète, volontairement retirée du code : **ne rien faire** (le down n'a plus de sens) |
| **Hors git** | introuvable partout (`all` vide) | Base importée d'un autre env (dump prod…), fichier renommé (voir `similar`) ou jamais commité : **ne rien faire** par défaut |

Présenter le tableau de classement, puis **une seule** AskUserQuestion pour les catégories « supprimée » + « hors git » : laisser tel quel (recommandé, aucun impact sur `migrate`) / supprimer ces entrées de la table `migrations` (cosmétique, avec accord). Ne poser des questions individuelles que pour la catégorie **Autre branche**.

Pour chaque migration inconnue `<NAME>` de la catégorie **Autre branche** :

1. Retrouver le fichier, le commit et la branche (le dossier peut varier : `database/migrations`, `Modules/*/Database/Migrations`, sous-dossiers…) :
   ```bash
   git log --all --format='%h %ad %an %s' --date=short --name-only -- "*<NAME>.php"
   git branch -a --contains <commit>
   ```
   Si introuvable : chercher aussi `git log --all -S '<NAME>'` et `git stash list`. Si toujours rien → rollback impossible ; proposer seulement la suppression de l'entrée (option b) ou de ne rien faire.
2. Afficher le contenu du `down()` :
   ```bash
   git show <commit>:<PATH>/<NAME>.php
   ```
   Résumer en une phrase ce que le down fera (tables/colonnes supprimées → **perte de données** à signaler). Si `down()` est vide ou absent, le dire : le rollback ne retirerait que l'entrée.
3. Demander via AskUserQuestion :
   - **Rollback de la migration** (recommandé si on revient durablement sur cette branche) ;
   - **Supprimer seulement l'entrée de la table `migrations`** (le schéma garde les modifs — si le down est cassé ou qu'on repassera sur l'autre branche) ;
   - **Ne rien faire**.
4. Si **rollback** :
   - proposer le dump de sécurité (étape 6) ;
   - restaurer temporairement le fichier **dans un dossier dédié** (pour que `--path` ne cible que lui), prévisualiser, exécuter, puis nettoyer :
     ```bash
     mkdir -p storage/db-sync-tmp
     git show <commit>:<PATH>/<NAME>.php > storage/db-sync-tmp/<NAME>.php
     $ARTISAN migrate:rollback --path=storage/db-sync-tmp --step=1000000 --pretend --no-interaction
     $ARTISAN migrate:rollback --path=storage/db-sync-tmp --step=1000000 --no-interaction
     rm -rf storage/db-sync-tmp
     git status --short
     ```
     `--path` limite le rollback aux fichiers du dossier (les autres migrations ran sont ignorées « Migration not found ») et `--step` large garantit que l'entrée est dans la sélection. Le `--pretend` doit afficher **uniquement** le SQL du down de `<NAME>` : sinon, stop.
     Si le conteneur Docker ne voit pas le dossier (volume partiel), utiliser un chemin monté (vérifier avec `$ARTISAN tinker --execute='echo base_path();'`).
   - Le dossier temporaire ne doit **jamais** rester sur le disque, même si le rollback échoue : toujours faire le `rm -rf`, puis vérifier `git status`.
   - Une migration anonyme (`return new class extends Migration`) ou nommée fonctionne pareil ; si une classe nommée entre en conflit avec une autre déjà chargée, le signaler.
5. Si **suppression de l'entrée** :
   ```bash
   $ARTISAN tinker --no-interaction --execute='echo app("migrator")->getRepository()->delete((object) ["migration" => "<NAME>"]) ?? "ok";'
   ```

### 3.3 Prévisualiser les migrations en attente

```bash
$ARTISAN migrate --pretend --no-interaction
```
Montrer le SQL en résumé et **signaler** toute instruction destructive (`DROP TABLE`, `DROP COLUMN`, `TRUNCATE`, `DELETE`, réduction de taille de colonne…). Si destructif → demander confirmation (AskUserQuestion) et proposer le dump (étape 6) avant de lancer. Note : `--pretend` n'exécute pas le code PHP hors schéma (requêtes Eloquent / `DB::table()` de reprise de données) de façon fiable ; le signaler si la migration en contient.

### 3.4 Exécuter les migrations en attente

```bash
$ARTISAN migrate --no-interaction
```

### 3.5 Gestion des erreurs

En cas d'échec, **toujours afficher l'erreur SQL brute**, identifier la migration fautive, puis diagnostiquer :

| Cas | Diagnostic | Proposition |
|---|---|---|
| `Base table or view already exists`, `Duplicate column name`, `Duplicate key name` | Le schéma existe déjà (autre branche, import de dump, création manuelle) | Comparer avec le `up()` via `SHOW CREATE TABLE`, puis proposer de marquer la migration comme exécutée (avec accord) : `$ARTISAN tinker --no-interaction --execute='$r = app("migrator")->getRepository(); $r->log("<NAME>", $r->getNextBatchNumber());'` |
| `Column not found`, `Can't DROP ...; check that it exists` | Schéma en avance/retard sur l'historique | Expliquer l'écart, proposer d'adapter manuellement ou de marquer la migration exécutée — avec accord |
| `Integrity constraint violation`, `Cannot add or update a child row` (FK), `Invalid use of NULL value`, `Data truncated`, `Duplicate entry` | Des données existantes bloquent la migration | Lancer un `SELECT` (lecture seule, OK sans accord) pour montrer les lignes fautives + leur nombre : `$ARTISAN tinker --no-interaction --execute='dump(DB::select("SELECT ..."));'`. Proposer la correction (`DELETE` / `UPDATE` / valeur par défaut) **sans l'exécuter**. Si validé : dump (étape 6), exécution via `DB::statement("...")`, puis relancer le migrate |
| Erreur PHP (classe introuvable, méthode inexistante, `Class "..." not found` d'un model supprimé), SQL invalide, erreur de connexion | Pas réglable simplement | Expliquer la cause, indiquer le fichier et la ligne, **stop** — ne pas modifier la migration sans demande |

Note MySQL/MariaDB : les DDL font un commit implicite, une migration qui échoue à mi-chemin peut laisser un schéma partiellement appliqué **sans** entrée dans `migrations`. Le signaler et identifier quelles instructions du `up()` sont déjà passées (`SHOW CREATE TABLE`). Sur PostgreSQL / SQLite, Laravel encapsule chaque migration dans une transaction (`$withinTransaction`), le rollback est réel.

Après toute correction validée → relancer 3.4, puis 3.1 pour vérifier.

---

## 4. Seeders (information seulement)

Si des migrations ont été jouées, regarder si des seeders ont changé sur la branche récemment :
```bash
git log -5 --format='%h %ad %s' --date=short -- database/seeders
```
Si une migration crée une table de référence que seul un seeder remplit, le **mentionner** et proposer `$ARTISAN db:seed --class=<Seeder> --no-interaction` — sans le lancer sans accord (un seeder peut dupliquer des données).

---

## 5. Check final

```bash
$ARTISAN migrate:status --no-interaction
```
Plus rien en `Pending` et plus de migration inconnue (relancer la commande tinker de 3.1) → « Base à jour ✅ ». Sinon lister ce qui reste et pourquoi (choix de l'utilisateur, erreur non résolue).

Laravel n'a pas d'équivalent à `doctrine:schema:update --dump-sql` : on ne peut pas comparer les models au schéma. Ne pas inventer de diff.

---

## 6. Dump de sécurité (toujours proposé, jamais imposé)

Avant chaque action destructive (rollback, `DELETE`/`UPDATE` de données, migration avec `DROP`), demander via AskUserQuestion : « Faire un dump de la base avant ? ».

Si oui :
1. Récupérer la connexion **sans lire `.env`** (souvent bloqué en lecture par les permissions) :
   ```bash
   $ARTISAN tinker --no-interaction --execute='$c = config("database.connections.".config("database.default")); echo json_encode(array_intersect_key($c, array_flip(["driver","host","port","database","username","password"])));'
   ```
2. Si les infos sont obtenues → lancer selon le driver, depuis le bon endroit (conteneur BDD si dockerisé, ex. `docker compose exec -T <db> mysqldump …` ; le host est alors `localhost` côté conteneur) :
   ```bash
   mysqldump -h <HOST> -P <PORT> -u <USER> -p'<PASSWORD>' --single-transaction <DB> > storage/db-sync-backup-$(date +%Y%m%d-%H%M%S).sql
   # PostgreSQL : pg_dump -h <HOST> -p <PORT> -U <USER> <DB> > storage/db-sync-backup-....sql
   # SQLite    : cp database/database.sqlite storage/db-sync-backup-$(date +%Y%m%d-%H%M%S).sqlite
   ```
   puis vérifier que le fichier n'est pas vide (`ls -lh`).
3. Sinon → **ne pas insister** ni contourner les permissions : donner la commande avec placeholders à lancer via le préfixe `!`, et **attendre la confirmation** avant de continuer :
   ```bash
   ! mysqldump -h <HOST> -P <PORT> -u <USER> -p <DB> > storage/db-sync-backup-<DATE>.sql
   ```

---

## 7. Récap final

Terminer par un tableau court :

| Élément | Résultat |
|---|---|
| Artisan utilisé | `php artisan` / `docker exec <conteneur> php artisan` / `./vendor/bin/sail artisan` |
| Migrations rollback | liste (+ branche d'origine) ou — |
| Entrées supprimées/ajoutées dans `migrations` | liste ou — |
| Migrations jouées | liste ou — |
| Erreurs | résolues / restantes (avec la cause) |
| Seeders à lancer | liste ou — |
| Dump | chemin du fichier ou — |
| Base | à jour ✅ / X en attente ⚠️ |
