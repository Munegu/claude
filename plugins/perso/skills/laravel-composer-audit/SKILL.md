---
name: laravel-composer-audit
description: "Lance un audit de sécurité Composer (CVE) sur un projet Laravel, met à jour les packages vulnérables (ou tous avec --all), vérifie que l'app tourne toujours (artisan about, route:list, view:cache, tests), puis produit un récap CVE/packages avec l'analyse d'impact (sans impact / à surveiller / action requise). Option --npm pour auditer aussi les dépendances JS. TRIGGER when: user invokes /laravel-composer-audit, demande un composer audit, de vérifier les CVE / failles des dépendances PHP, ou de mettre à jour les packages Composer d'un projet Laravel."
argument-hint: "[--all] [--npm]"
allowed-tools: [Bash, Read, Edit, Write, WebFetch, Grep, Glob, AskUserQuestion]
author: NV
---

# Laravel composer audit + update

Audite les dépendances Composer du projet Laravel, met à jour les packages, valide que rien n'est cassé et termine par un récap exploitable.

- Sans argument (mode **security-only**, par défaut) : mise à jour **uniquement** des packages vulnérables (et de leurs dépendances strictement nécessaires).
- `--all` : mise à jour de **tous** les packages dans le respect des contraintes de `composer.json`.
- `--npm` : ajoute un `npm audit` (rapport seulement, aucune mise à jour JS sans accord).

Ne jamais modifier les contraintes de `composer.json` (bump de major, `laravel/framework` compris) sans validation explicite. Ne pas commiter.

## Étapes

### 0. Préparation

1. Vérifier la présence de `artisan`, `composer.json`, `composer.lock` et de `laravel/framework` dans `composer.json`.
2. `git status --porcelain composer.json composer.lock package.json package-lock.json` : si ces fichiers sont déjà modifiés, prévenir et demander s'il faut continuer (sinon le diff final sera faussé).
3. **Choisir les commandes** (`$COMPOSER`, `$ARTISAN`), à réécrire en entier dans chaque appel Bash :
   - d'abord les consignes du projet (`CLAUDE.md`, `CLAUDE.local.md`, `.claude/CLAUDE.md`) : commande Docker (`docker exec <conteneur> composer`, `docker compose run --rm <service> composer`…), Sail (`./vendor/bin/sail composer`), cible `make` ;
   - sinon Sail si `vendor/bin/sail` existe et que les conteneurs tournent, sinon un service PHP Docker si le PHP local n'a pas la bonne version, sinon `composer` / `php artisan` locaux.
   - Composer doit tourner **avec la même version de PHP et les mêmes extensions** que l'app : si le PHP local diffère de celui du conteneur, passer par le conteneur (sinon le lock sera résolu pour la mauvaise plateforme).
4. Relever : version PHP effective (`$ARTISAN --version` + `php -v` dans le même contexte), `config.platform.php` de `composer.json`, version de `laravel/framework` installée.
5. Créer un répertoire de travail (scratchpad de la session si disponible, sinon `mktemp -d`) → `$WORK`, et snapshot :
   ```bash
   cp composer.lock "$WORK/composer.lock.before"
   $COMPOSER show --locked --format=json > "$WORK/before.json"
   $COMPOSER show --locked --direct --name-only > "$WORK/direct.txt"
   ```
   (si `$COMPOSER` tourne dans un conteneur, rediriger la sortie côté hôte comme ci-dessus — ne pas écrire dans un chemin que seul le conteneur voit.)

### 1. Audit initial

```bash
$COMPOSER audit --locked --format=json > "$WORK/audit-before.json"; echo "exit=$?"
```
(un exit code non nul est normal quand des failles existent.)

Parser le JSON (`advisories`, `abandoned`) et présenter un tableau :

| Package | Version installée | CVE / advisory | Sévérité | Versions affectées | Direct ? | Lien |

- **Direct ?** : présent dans `$WORK/direct.txt`, sinon transitif → indiquer qui le tire (`$COMPOSER why <package>`).
- Lister aussi les packages **abandonnés** avec leur remplaçant suggéré.
- Si aucune CVE : le dire et s'arrêter avec un récap (sauf `--all` → continuer vers l'update).

Pour chaque CVE, déterminer la version corrigée (advisory / GitHub Security Advisory via WebFetch si besoin) et vérifier si elle est atteignable avec les contraintes actuelles (`$COMPOSER why-not <package> <version-corrigée>`). Sinon (major à franchir, PHP trop vieux, `laravel/framework` bloqué…) → **bloquante** : remontée dans le récap avec ce qu'il faudrait changer, sans le faire.

### 2. Mise à jour

- Security-only :
  ```bash
  $COMPOSER update <pkg1> <pkg2> ... --with-dependencies --no-interaction
  ```
  Si la résolution échoue, réessayer avec `--with-all-dependencies` et le signaler (périmètre élargi). Ne jamais retomber sur un `update` sans liste de packages dans ce mode.
- `--all` :
  ```bash
  $COMPOSER update --with-all-dependencies --no-interaction
  ```

En cas d'échec de résolution : lire l'erreur, identifier le package/contrainte bloquant, réessayer en l'excluant si pertinent, le noter. `--ignore-platform-reqs` interdit sauf demande explicite.

Surveiller la sortie des scripts `post-autoload-dump` (`package:discover`) : une erreur ici casse souvent l'app (provider supprimé, classe renommée) → la traiter comme une vérification en échec.

Si des packages publient des assets/config (`vendor:publish` mentionné dans les release notes : Livewire, Horizon, Telescope, AdminLTE…), le signaler sans le lancer.

### 3. Diff des versions

```bash
$COMPOSER show --locked --format=json > "$WORK/after.json"
php -r '
$a = array_column(json_decode(file_get_contents($argv[1]), true)["locked"], "version", "name");
$b = array_column(json_decode(file_get_contents($argv[2]), true)["locked"], "version", "name");
foreach ($b as $n => $v) {
    if (!isset($a[$n])) echo "+ $n $v\n";
    elseif ($a[$n] !== $v) echo "~ $n {$a[$n]} -> $v\n";
}
foreach ($a as $n => $v) if (!isset($b[$n])) echo "- $n {$v}\n";
' -- "$WORK/before.json" "$WORK/after.json"
```

Qualifier chaque bump : **patch** / **minor** / **major** (en `0.x`, un minor est un major), direct / transitif.

### 4. Vérifications post-update

Lancer ce qui a du sens dans le projet (vérifier avant, ne pas inventer) :

```bash
$ARTISAN optimize:clear --no-interaction
$ARTISAN about --no-interaction            # boot complet de l'app + providers
$ARTISAN route:list --no-interaction > /dev/null && echo ROUTES_OK   # résout les contrôleurs / middlewares
$ARTISAN config:cache --no-interaction && $ARTISAN config:clear --no-interaction   # config sérialisable (pas de closure)
$ARTISAN view:cache --no-interaction && $ARTISAN view:clear --no-interaction       # compile toutes les vues Blade / composants
$ARTISAN event:list --no-interaction > /dev/null && echo EVENTS_OK
```
Si Livewire est présent, `$ARTISAN livewire:list` (ou équivalent selon la version) peut aider ; si Horizon, `$ARTISAN horizon:status` (info seulement).

**Tests** : seulement si `tests/` contient au moins un `*Test.php` (`find tests -name '*Test.php' -not -path '*/Browser/*' | head -1`). Utiliser la commande de test des consignes du projet si elle existe, sinon `$ARTISAN test`. Ne pas lancer Dusk / Cypress / PHPStan / Larastan (trop long, hors périmètre).
⚠️ Avant de lancer les tests, vérifier qu'ils ne videront pas la base de dev : si des tests utilisent `RefreshDatabase` / `DatabaseMigrations` / `LazilyRefreshDatabase` **et** que `phpunit.xml` / `.env.testing` ne pointent pas vers une base dédiée (sqlite `:memory:`, base `*_test`…), **ne pas les lancer** et le signaler.

**Dépréciations** : si `LOG_DEPRECATIONS_CHANNEL` est configuré, relever les nouvelles entrées du log correspondant après les vérifications ; sinon, chercher dans les UPGRADE/CHANGELOG (étape 5).

Relancer l'audit :
```bash
$COMPOSER audit --locked --format=json > "$WORK/audit-after.json"; echo "exit=$?"
```

Si une vérification casse : identifier le package responsable (message, stack trace, diff de version), proposer un correctif ou un retour arrière ciblé (`$COMPOSER update <pkg>:<ancienne-version>` / contrainte temporaire) — **demander avant d'appliquer**.

### 4bis. `--npm` (optionnel)

```bash
npm audit --json > "$WORK/npm-audit.json"; echo "exit=$?"
```
(ou via le conteneur Node si les consignes du projet l'indiquent.) Présenter les vulnérabilités par sévérité, en distinguant dépendances de **build** (webpack, laravel-mix, vite… → impact faible en prod) et dépendances **runtime** embarquées dans les bundles (→ à corriger). Proposer `npm audit fix` (sans `--force`) uniquement avec accord, puis `npm run dev` / `npm run build` pour vérifier que les assets compilent.

### 5. Analyse d'impact

Classer chaque package mis à jour :

- ✅ **Sans impact** : patch, ou minor d'un transitif non utilisé directement dans `app/`, vérifications OK.
- ⚠️ **Impact potentiel — à surveiller** : minor/major d'un package direct, ou package dont les classes/façades/config sont utilisées dans le projet.
- ❌ **Action requise** : vérification en échec, CVE non corrigée, package abandonné, `vendor:publish` nécessaire.

Pour les ⚠️ et ❌ :
1. Grep de l'usage réel : namespace du package dans `app/`, `Modules/`, `routes/`, `database/`, `config/<package>.php`, directives/composants Blade dans `resources/views`, providers dans `bootstrap/providers.php` / `config/app.php`.
2. Lire le CHANGELOG / UPGRADE / release notes entre les deux versions (`vendor/<pkg>/CHANGELOG.md`, `UPGRADE*.md`, sinon releases GitHub via WebFetch). Regrouper `laravel/*` et `illuminate/*` en une ligne, en se limitant aux composants utilisés.
3. En déduire des **points de surveillance concrets** (ex. « livewire/livewire 3.4 → 3.5 : changement du hydrate des propriétés typées → tester le formulaire de commande » ; « guzzlehttp/guzzle : changement de gestion des timeouts → vérifier les appels du service de paiement »). Pas de « tester l'application ».

### 6. Récap final

```markdown
## Récap composer audit + update

**Projet** : <nom> — PHP <x.y> — Laravel <x.y>
**Mode** : security-only (défaut) | complet (`--all`) [+ npm]

### CVE
| Package | CVE | Sévérité | Avant | Après | Statut |
|---|---|---|---|---|---|
| ... | CVE-XXXX-YYYY | high | 1.2.3 | 1.2.5 | ✅ corrigée / ❌ non corrigée (raison) |

### Packages mis à jour (N)
| Package | Avant → Après | Type | Direct | Impact |
|---|---|---|---|---|

(packages ajoutés / supprimés listés à part)

### Vérifications
about ✅ · route:list ✅ · config:cache ✅ · view:cache ✅ · tests ✅ (X tests) / non lancés (raison) · audit final : 0 CVE

### npm (si --npm)
X vulnérabilités (runtime : …, build : …)

### À surveiller
- **<package> <avant> → <après>** : <ce qui change> → <où dans le projet> → <quoi tester>

### Actions requises / non traitées
- CVE bloquantes et ce qu'il faudrait changer
- Packages abandonnés + remplaçant
- `vendor:publish` / migrations de packages à lancer

### Conclusion
Une phrase : **sans impact** / **impact potentiel limité (voir « À surveiller »)** / **à ne pas livrer en l'état**.
```

Terminer en rappelant que `composer.lock` (et `package-lock.json` si modifié) n'est pas commité, et proposer de générer le message de commit.
