# 🔒 Solution prof – Exercice 4 « deploy-with-ansible »

> Dépôt **privé**, réservé à l'enseignant. Ne pas rendre public ni le lier depuis un support étudiant.

Le scénario complet de l'énoncé a été **réellement joué** dans ce dépôt : chaque ligne ci-dessous renvoie au run GitHub Actions correspondant.

- Point de départ (`main`, 1er commit) = fin de l'exercice 3 (solution `automated-tests-solution`, étape 15).
- `etape-2-4-docker-ansible` = parties 2 à 4 (Dockerfile, `.dockerignore`, `ansible/`, `ci-cd.yml` avec `needs: [build, performance]`).
- `etape-5-4-autoriser-prod` = partie 5.4 (`maintenance_mode: false`).

## Partie 3 – Exécution locale d'Ansible (vérifiée avec ansible-core 2.22)

| Commande | Résultat |
|---|---|
| `ansible-playbook -i ansible/inventory/dev.ini ansible/playbook.yml` | ✅ `Env=development \| port=8080 \| debug=True` · ok=2 changed=1 skipped=1 |
| `ansible-playbook -i ansible/inventory/test.ini ansible/playbook.yml` | ✅ `Env=testing \| port=8081 \| debug=False` · ok=2 changed=1 skipped=1 |
| `ansible-playbook -i ansible/inventory/prod.ini ansible/playbook.yml` | ❌ `Deployment blocked: maintenance mode enabled on production` · failed=1 |

## Partie 5 – Le pipeline, événement par événement (vérifié le 12/11/2026)

| # | Étape de l'énoncé | Événement | Run | TEST | DEV | PROD |
|---|---|---|---|---|---|---|
| 1 | 5.1 Observer la PR | [PR 1](https://github.com/corentinbeuchet/deploy-with-ansible-solution/pull/1) `etape-2-4-docker-ansible` → `main` | [run](https://github.com/corentinbeuchet/deploy-with-ansible-solution/actions/runs/43222946862) | 🟢 | ⏭️ skipped | ⏭️ skipped |
| 2 | 5.2 Merge dans `main` | push `main` | [run](https://github.com/corentinbeuchet/deploy-with-ansible-solution/actions/runs/43223520874) | ⏭️ | ⏭️ | 🔴 **maintenance** (voulu) |
| 3 | 5.2 Création de `develop` | push `develop` | [run](https://github.com/corentinbeuchet/deploy-with-ansible-solution/actions/runs/43223555202) | ⏭️ | 🟢 | ⏭️ |
| 4 | 5.4 PR « autoriser la prod » | [PR 2](https://github.com/corentinbeuchet/deploy-with-ansible-solution/pull/2) → `develop` | [run](https://github.com/corentinbeuchet/deploy-with-ansible-solution/actions/runs/43223604290) | 🟢 | ⏭️ | ⏭️ |
| 5 | 5.4 Merge dans `develop` | push `develop` | [run](https://github.com/corentinbeuchet/deploy-with-ansible-solution/actions/runs/43224213731) | ⏭️ | 🟢 | ⏭️ |
| 6 | 5.4 PR `develop` → `main` | [PR 3](https://github.com/corentinbeuchet/deploy-with-ansible-solution/pull/3) → `main` | [run](https://github.com/corentinbeuchet/deploy-with-ansible-solution/actions/runs/43224222014) | 🟢 | ⏭️ | ⏭️ (pas encore mergé) |
| 7 | 5.4 Merge dans `main` | push `main` | [run](https://github.com/corentinbeuchet/deploy-with-ansible-solution/actions/runs/43224749832) | ⏭️ | ⏭️ | 🟢 `image_tag=3a30631…` |

Dans chaque run, `build` → `performance` → `docker` (build de l'image + test de fumée) sont verts avant le déploiement.

## Pièges et questions fréquentes
- **« La PR vers `main` n'a pas lancé Deploy PROD ? »** Normal (ligne 6) : sur une PR, `github.ref` vaut `refs/pull/<n>/merge`, donc seul `deploy-test` s'exécute. La prod ne part qu'au **merge** (push sur `main`), c'est-à-dire après relecture.
- **`image_tag` d'une PR ≠ dernier commit de la branche** : sur un `pull_request`, `github.sha` est le commit de merge temporaire calculé par GitHub (ex. PR 1 : `921e5e8…` alors que la branche est en `3372f5e…`). Sur un push, c'est bien le commit poussé.
- Le premier merge dans `main` **doit** échouer en PROD : c'est le garde-fou `maintenance_mode`, première tâche du playbook.

## Non reproduit ici (Settings)
- **Protection de `main` et `develop`** (5.3) et **bonus « Required reviewers »** : sur un dépôt **privé** GitHub Free, ni la protection de branche ni les règles d'environnement ne sont disponibles. Dans les dépôts publics des étudiants, ça fonctionne. Pour le bonus, ajouter au job `deploy-prod` :
  ```yaml
  deploy-prod:
    environment: production
  ```

## Versions
Spring Boot 4.1.2 · Gradle 9.9.1 · Java 25 · Checkstyle 14.4.0 · `eclipse-temurin:25-jre` · Ansible préinstallé sur `ubuntu-26.04` · `actions/checkout@v7`, `setup-java@v6`, `upload-artifact@v7`, `download-artifact@v8`, `grafana/setup-k6-action@v1`, `grafana/run-k6-action@v1`.
