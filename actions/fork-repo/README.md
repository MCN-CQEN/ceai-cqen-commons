# Fork repository (action GitHub)

Description
-----------
Action composite GitHub qui :
- fork un dépôt source vers un dépôt cible (owner/name),
- copie les variables d'environnement GitHub Actions (repository variables),
- recrée les environnements et leurs variables,
- recrée les noms de secrets (repository & environment) dans le dépôt cible en tant que placeholders à configurer manuellement.

Cette action est utile pour cloner la configuration d'un dépôt (variables / environnements / secrets) vers un nouveau dépôt forké.

Prérequis
---------
- GitHub CLI (`gh`) disponible dans le runner (recommandé : gh >= 2.x).
- `jq` installé dans le runner.
- Un token GitHub (PAT) avec les permissions nécessaires (voir section Permissions).
- Un runner GitHub Actions (ubuntu-latest fonctionne).

Permissions requises / Token
----------------------------
Le token fourni via l'input `github_token` doit avoir les droits suffisants pour :
- créer un fork dans l'organization target (si le fork doit être créé sous une org) ;
- lire les variables, environnements et secrets du dépôt source ;
- écrire les variables, environnements et secrets du dépôt cible.

Recommandation : utilisez un Personal Access Token (PAT) stocké en secret (par ex. `secrets.FORK_PAT`) avec scopes appropriés (repo, admin:org si nécessaire). Le GITHUB_TOKEN par défaut peut ne pas suffire pour forker dans une organisation ou pour modifier certaines ressources.

Inputs
------
- `source_repo` (required): dépôt source au format `owner/source-repo`.
- `target_repo` (required): dépôt cible au format `owner/target-repo`.
- `github_token` (required): token GitHub (PAT ou autre) avec permissions suffisantes.

Comportement détaillé
---------------------
1. Fork
   - La commande `gh api --method POST "/repos/${SOURCE_REPO}/forks"` est utilisée pour créer le fork.
   - Le fork est attendu (poll) jusqu'à 60 secondes (30 itérations de 2s). Si le fork n'apparaît pas, l'action échoue.
2. Variables de dépôt
   - Lit `/repos/{source}/actions/variables` et recrée chaque variable dans le dépôt cible avec sa valeur.
3. Environnements et variables d'environnement
   - Liste les environnements du dépôt source, crée chaque environnement dans le dépôt cible, puis copie les variables d'environnement via les endpoints correspondants.
4. Secrets (repository & environment)
   - L'API GitHub ne permet pas de lire la valeur des secrets chiffrés. L'action récupère les noms des secrets et crée dans le dépôt cible des secrets avec la valeur placeholder `"TO_BE_CONFIGURED"`. Il faudra configurer manuellement les valeurs dans le dépôt cible.

Exemple d'utilisation (workflow)
--------------------------------
```yaml
name: Fork and copy config

on:
  workflow_dispatch:

jobs:
  fork:
    runs-on: ubuntu-latest
    steps:
      - name: Fork and copy
        uses: MCN-CQEN/ceai-cqen-scripts-lib/actions/fork-repo@main
        with:
          source_repo: "org/source-repo"
          target_repo: "org/target-repo"
          github_token: ${{ secrets.FORK_PAT }}
```

Limitations & sécurité
-----------------------
- Les secrets ne sont pas copiés en clair : seuls les noms sont recréés avec la valeur "TO_BE_CONFIGURED". Tu dois reconfigurer les secrets dans le dépôt cible.
- Copier les variables de dépôt peut exposer des valeurs sensibles si ton token a accès en lecture : ne fournis jamais un token avec des permissions excessives sur des environnements non sûrs.
- Le fork est asynchrone : selon la charge et la politique GitHub, la création du fork peut prendre plus de temps que l'attente par défaut (60s). Si tu constates des erreurs de délai, augmente la boucle d'attente dans l'action.
- L'action repose sur gh api. Selon la version du CLI ou des changements d'API GitHub, certains endpoints peuvent évoluer.

Dépannage
----------
- Erreur "Fork did not appear after 60 seconds" :
   - Vérifie que le token a le droit de forker dans l'organisation cible.
   - Augmente le délai d'attente dans l'action (modifier le script).
- Erreurs 403 / 401 :
   - Vérifie le token (scope, validité).
   - Vérifie que le token n'est pas limité par des politiques d'organization (SSO, restrictions).
- Variables non copiées :
   - Vérifie que l'endpoint /actions/variables renvoie bien des valeurs pour l'utilisateur/token utilisé.
- Secrets non copiés en clair :
   - C'est attendu : c'est une protection. Les noms sont recréés et tu dois renseigner les valeurs dans le dépôt cible.

Bonnes pratiques
-----------------
- Utilise un PAT limité aux scopes nécessaires et stocke-le dans Secrets.
- Après la copie, supprime ou restreins le PAT si possible.
- Vérifie manuellement les environnements et secrets sensibles dans le dépôt cible avant d'exposer le dépôt.
