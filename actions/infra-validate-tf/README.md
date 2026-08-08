# Script réutilisable pour valider le code Terraform

## Table des matières

- [Introduction](./README.md#introduction)
- [Fonctionnalités](./README.md#fonctionnalités)
- [Entrées (Inputs)](./README.md#entrées-inputs)
- [Sorties (Outputs)](./README.md#sorties-outputs)
- [Exemple d’utilisation](./README.md#exemple-dutilisation)
- [Détails du fonctionnement](./README.md#détails-du-fonctionnement)
- [Prérequis](./README.md#prérequis)
- [Limitations](./README.md#limitations)

## Introduction

Ce workflow GitHub Actions est un workflow composite réutilisable destiné à valider du code Terraform dans un environnement AWS.

Il effectue les opérations suivantes :
- Vérification du formatage Terraform (terraform fmt)
- Initialisation du backend Terraform (terraform init)
- Validation de la configuration (terraform validate)
- Analyse de sécurité via Checkov
- Configuration automatique d’AWS via un rôle IAM
- Exécution dans le même runner que le workflow appelant (pas besoin d’artefacts)

Ce workflow est conçu pour être appelé par d’autres workflows sans duplication de logique.

## Fonctionnalités

- 📦 Checkout automatique du repository
- 🔐 Configuration des identifiants AWS via aws-actions/configure-aws-credentials
- 🏗️ Formatage, init et validation Terraform
- 🛡️ Scan de sécurité Checkov (avec liste d’ignores personnalisable)
- 🔁 Compatible avec les environnements multiples (prod, preprod, etc.)
- 🧩 Workflow réutilisable simple à intégrer

## Entrées (Inputs)

| Nom de l’input | Description | Obligatoire |Valeur par défaut|
|------|--------|---------|---------|
|working_directory|Répertoire contenant les fichiers Terraform|✅|-|
|environment|Environnement ciblé (ex : preprod, prod)|✅|-|
|aws_region|Région AWS|✅|ca-central-1|
|bucket_tf_state|Nom du bucket S3 contenant l'état Terraform|✅|-|
|role_to_assume|Rôle IAM à assumer|✅|-|
|checkov_ignore_list|Liste des vérifications Checkov à ignorer (séparées par des virgules)|❗|BC_CROSS_1,BC_CROSS_2,BC_AWS_GENERAL_192,CKV_AWS_149|
|checkov_ca_bundle|Chemin facultatif vers un bundle CA du runner. Les chemins Linux standards sont détectés automatiquement si cette valeur est vide.|❌|Détection automatique|

## Sorties (Outputs)

| Output | Description |
|------|--------|
|terraform_validate|Résultat de terraform validate|
|terraform_fmt|Résultat de terraform fmt|
|terraform_init|Résultat de terraform init|
|checkov|Résultat de l’analyse Checkov|	

## Exemple d’utilisation

Vous pouvez appeler ce workflow réutilisable depuis un autre workflow comme suit :

```yaml
name: Validate Terraform

on:
  pull_request:
    paths:
      - "infra/**"

jobs:
  terraform-validation:
    uses: MCN-CQEN/ceai-cqen-commons/actions/infra-validate-tf@main
    with:
      working_directory: infra/terraform
      environment: preprod
      aws_region: ca-central-1
      bucket_tf_state: my-terraform-state-bucket
      role_to_assume: arn:aws:iam::123456789012:role/GitHubActionsTerraformRole
      checkov_ignore_list: "BC_AWS_GENERAL_192,CKV_AWS_149"
```

## Détails du fonctionnement

1. Checkout du code
    - Utilise actions/checkout@v7.0.0
    - Récupère tout l’historique (fetch-depth: 0) pour certains modules Terraform
2. Configuration AWS
    - Utilise aws-actions/configure-aws-credentials@v6.2.2
    - Assume le rôle IAM spécifié (role_to_assume)
    - Le runner peut ensuite effectuer des actions AWS sécurisées
3. Installation de Terraform
    - Utilise hashicorp/setup-terraform@v4.0.1
    - Permet d’utiliser la version définie dans .terraform-version ou dans le workflow appelant
4. Formatage Terraform
    
    Commande exécutée :

    ```bash
    terraform fmt -check -recursive -diff
    ```
5. Initialisation Terraform

    Avec le backend S3 fourni :
    ```bash
    terraform init -backend-config="bucket=<bucket>"
    ```
6. Validation Terraform
    ```bash
    terraform validate
    ```
7. Analyse de sécurité Checkov
    - Utilise bridgecrewio/checkov-action@master
    - Analyse les modules Terraform
    - Possibilité d’ignorer certains checks via checkov_ignore_list
    - Combine le bundle CA système et les certificats configurés sur le runner (`NODE_EXTRA_CA_CERTS`, `SSL_CERT_FILE`, `REQUESTS_CA_BUNDLE`, `CURL_CA_BUNDLE` ou Git) afin de prendre en charge les proxys d’entreprise
    - Convertit les identifiants Prisma `BC_*` connus en identifiants Checkov `CKV_*` et désactive les téléchargements Prisma lorsque le scan peut fonctionner entièrement hors ligne

## Prérequis
- Un bucket S3 existant pour stocker l’état Terraform
- Facultatif : un bundle CA système dans `/etc/ssl/certs/ca-certificates.crt` (Debian/Ubuntu), `/etc/pki/tls/certs/ca-bundle.crt` (RHEL), ou indiqué avec `checkov_ca_bundle`. Sans bundle détecté, Checkov utilise son magasin de certificats par défaut.
- Un rôle IAM permettant les actions suivantes :
    - sts:AssumeRole
    - Permissions nécessaires à l’init Terraform (S3, DynamoDB si lock)
- Le code Terraform doit être valide dans le répertoire fourni
- Le workflow appelant doit utiliser un runner compatible (Linux recommandé)

## Limitations
- Ne supporte pas encore la validation pour plusieurs workspaces Terraform (peut être ajouté sur demande)
- Utilise la branche master de Checkov, ce qui peut entraîner des changements non anticipés
- Les secrets ne sont pas transmis via des inputs, mais doivent être dans secrets.* du workflow appelant
