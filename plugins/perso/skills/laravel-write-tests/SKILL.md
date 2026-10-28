---
name: laravel-write-tests
description: "Génère des tests PHPUnit ou Pest pour un fichier source ou une feature d'un projet Laravel, en suivant les conventions des tests existants (stratégie BDD, factories, rôles, fakes Queue/Mail/Http…), puis les lance. Couvre contrôleurs, policies, FormRequests, jobs, commandes, composants Livewire, services, listeners. TRIGGER when: user invokes /laravel-write-tests, asks to write or generate tests for a file or feature in a Laravel project."
argument-hint: "[chemin/vers/fichier.php ...] [--type=unit|feature]"
allowed-tools: [Bash, Read, Edit, Write, Grep, Glob, AskUserQuestion]
author: NV
---

# Laravel Write Tests

Génère les tests en respectant les conventions **du projet** (les tests existants font foi, pas les habitudes génériques). Seul le code PHP backend est concerné (pas de JS, CSS, Blade seul, Dusk ni Cypress).

## Étape 1 — Identifier les fichiers cibles

- Chemins dans `$ARGUMENTS` → les utiliser.
- Sinon :
  ```bash
  git diff HEAD --name-only; git diff main...HEAD --name-only 2>/dev/null
  ```
  Garder les fichiers PHP de `app/`, `Modules/*/` (hors `Models/` simples, migrations, config, vues — sauf demande). Si plusieurs, demander lesquels prioriser (AskUserQuestion).

Lire chaque fichier cible : méthodes publiques, dépendances injectées, routes associées (`grep -rn '<Controller>' routes/ Modules/*/Routes/`), autorisations (`authorize()`, `Gate::`, `can:` / `role:` middleware, policies), événements/jobs/mails déclenchés, appels HTTP externes.

## Étape 2 — Détecter les conventions du projet

```bash
grep -E '"(pestphp/pest|phpunit/phpunit|mockery/mockery|livewire/livewire|spatie/laravel-permission)"' composer.json
find tests -name '*Test.php' -not -path '*/Browser/*' | head -30
cat phpunit.xml | grep -E '<env|testsuite name'
ls .env.testing 2>/dev/null
```

Lire **1 à 2 tests existants les plus proches** du type à écrire (même dossier / même type de classe) et `tests/TestCase.php`, puis relever :
- **Framework** : Pest (`tests/Pest.php`, `it()` / `test()`) ou PHPUnit ; style de nommage (`test_snake_case`, `#[Test]`, `/** @test */`, camelCase) ;
- **Stratégie BDD** — point critique :
  - `RefreshDatabase` / `LazilyRefreshDatabase` / `DatabaseMigrations` → la base de test est vidée/recréée ;
  - `DatabaseTransactions` → tests sur une base existante, rollback à la fin ;
  - base utilisée : `DB_CONNECTION` / `DB_DATABASE` dans `phpunit.xml` / `.env.testing` (sqlite `:memory:`, base `*_test`, ou… la base de dev).
- **Données de référence** : seeders appelés dans le `TestCase` (`$seed`, `$seeder`), rôles/permissions supposés existants, brand/tenant par défaut ;
- **Helpers maison** dans `TestCase` (login par rôle, création d'utilisateur…) → les réutiliser ;
- **Factories** disponibles : `ls database/factories Modules/*/Database/[Ff]actories 2>/dev/null` ; lire celles des modèles manipulés (états, relations).

**Règle de sécurité** : utiliser **la même stratégie BDD que les tests existants**. Ne jamais introduire `RefreshDatabase` / `DatabaseMigrations` dans un projet qui n'a pas de base de test dédiée : ça viderait la base de dev. En l'absence de tests existants et de base de test dédiée, demander (AskUserQuestion) : `DatabaseTransactions` sur la base locale / configurer une base de test (sqlite ou `*_test`) d'abord.

## Étape 3 — Choisir le type de test

`--type=unit|feature` force le choix. Sinon :

| Classe source | Type | Ce qu'on teste |
|---|---|---|
| Contrôleur (HTTP) | Feature | Pour chaque route : invité → redirection login / 401 ; utilisateur sans droit → 403 ; utilisateur autorisé → 200/302 + effet en base (`assertDatabaseHas`) + vue/JSON (`assertViewHas`, `assertJson`, `assertJsonStructure`) ; validation (`assertInvalid([...])` / `assertSessionHasErrors`) |
| Policy / Gate | Feature (modèles en base) ou Unit | Chaque ability × chaque rôle pertinent : `assertTrue($user->can('update', $model))`, propriétaire vs non-propriétaire, `before()` admin |
| FormRequest | Feature (via la route) | Cas valide + un cas invalide par règle importante (`@dataProvider` / `with()` en Pest) |
| Job | Feature | Appel direct `(new Job(...))->handle(...)` avec dépendances mockées ou résolues ; effets en base ; et côté déclencheur : `Queue::fake()` / `Bus::fake()` + `assertPushed` / `assertPushedOn('<queue>')` |
| Commande artisan | Feature | `$this->artisan('cmd', [...])->expectsOutputToContain(...)->assertExitCode(0)` (ou `assertSuccessful()`), + effets en base |
| Composant Livewire | Feature | `Livewire::test(Component::class, [...])->set(...)->call(...)->assertSet(...)->assertHasErrors([...])->assertDispatched(...)` ; `actingAs` avant |
| Service / Action | Unit si pur (Mockery pour les dépendances), Feature s'il touche la base | Chaque méthode publique, cas nominal + cas limites + exceptions (`expectException`) |
| Listener / Observer | Feature | Déclencher l'événement / l'action sur le modèle, vérifier l'effet ; `Event::fake([...])` pour vérifier l'émission sans exécuter |
| Mailable / Notification | Feature | `Mail::fake()` / `Notification::fake()` + `assertSent` / `assertSentTo` avec closure sur le contenu ; rendu : `(new Mailable(...))->assertSeeInHtml(...)` |
| Middleware | Feature | Route de test ou route existante, avec/sans la condition |
| Scope / accessor / cast de modèle | Unit ou Feature | Valeurs limites, `null` |

## Étape 4 — Générer les tests

- Créer les données avec les **factories** (états et relations existants plutôt que des tableaux en dur) ; ne pas dépendre de données présentes en base sauf si les tests existants le font (rôles seedés, brand par défaut…), et alors utiliser les mêmes accès.
- Authentification : `actingAs($user)` (ou le helper du `TestCase`) ; rôles via `assignRole()` si spatie/permission ; ne pas oublier le guard si plusieurs.
- **Aucun appel externe réel** : `Http::fake()` (+ `Http::preventStrayRequests()`), `Queue::fake()`, `Mail::fake()`, `Notification::fake()`, `Storage::fake('<disk>')`, `Event::fake([...])` ciblé (pas global si des observers doivent tourner). Services tiers (paiement, vidéo, CRM, OAuth…) → mock via `$this->mock(Service::class, fn ($m) => ...)` ou `$this->instance()`.
- Temps : `$this->travelTo(...)` / `Carbon::setTestNow()` si la logique dépend de la date.
- Multi-tenant / multi-brand : reproduire la manière dont les tests existants fixent le tenant (config, host de la requête, middleware). Si aucun test n'en donne l'exemple et que le code en dépend, le signaler plutôt que de deviner.

## Étape 5 — Règles de qualité

- Un scénario = un test, nom qui décrit le comportement (`test_student_cannot_submit_homework_after_deadline`).
- Assertions précises : `assertSame` pour les scalaires, `assertFalse()` plutôt que `assertTrue(!...)`, `assertCount`, `assertDatabaseHas` / `assertDatabaseMissing` / `assertModelExists` / `assertSoftDeleted`.
- Status HTTP explicites : `assertOk()`, `assertForbidden()`, `assertRedirect(route('...'))`, `assertNotFound()`, `assertUnprocessable()`.
- Pas de `Model::first()` sans critère : toujours cibler l'enregistrement créé dans le test.
- Pas de `sleep`, pas de dépendance à l'ordre d'exécution, pas d'IDs en dur.

## Étape 6 — Écrire le fichier

Emplacement selon les conventions existantes (en général `tests/Feature/<arbo d'app>/XxxTest.php` ou `tests/Unit/...` ; `Modules/<Mod>/Tests/...` si les modules ont leurs propres tests). Si le fichier existe déjà : le lire et n'ajouter que les tests manquants.

## Étape 7 — Lancer les tests

Commande : celle des consignes du projet (`CLAUDE.md` : Docker, Sail, `make tests`…), sinon `php artisan test`. Cibler le fichier :
```bash
<commande-test> --filter=<NomDuFichierTest>
# ou: <commande-test> tests/Feature/.../XxxTest.php
```
⚠️ Ne pas lancer si l'étape 2 a montré un risque de vider la base de dev (trait de refresh sans base dédiée) : le signaler et s'arrêter.

En cas d'échec :
- **erreur dans le test** (factory incomplète, mauvaise route, donnée manquante) → corriger le test et relancer ;
- **le test révèle un bug du code** → ne pas modifier le code source ni « adapter » l'assertion pour faire passer : marquer le test (`$this->markTestIncomplete('…')` ou le laisser en échec selon la préférence de l'utilisateur) et le **signaler** clairement.

Conclure avec : le(s) fichier(s) créé(s), le résultat de l'exécution (X passés / Y échecs + cause), les cas non couverts, et la commande pour relancer.
