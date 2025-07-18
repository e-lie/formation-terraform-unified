---
title: "Cours - Les entrées et sorties de Terraform"
description: "Guide Cours - Les entrées et sorties de Terraform"
sidebar:
  order: 144
---



## Terraform et le cycle des données 

**Terraform suit un cycle de données en plusieurs étapes pour gérer l'infrastructure.**

![](/144_cours_entrees_sorties/images/terraform-data-arguments-and-attributes.png)


Tout d'abord, Terraform doit connaître le contexte de l'utilisateur, tel que l'environnement, les mots de passe, les variables, et autres contraintes d'exécution.

**Les données en entrée sont utilisées sous formes d'`arguments` pour les ressources.**

---

Ensuite, le déploiement fournit des données telles que les data sources, les ID de machine, les adresses IP et d'autres informations d'infrastructure.

**Ce sont les `attributs` qui sont un mélange d'argument et d'attributs générés à l'instanciation.**

Lorsque la configuration est appliquée, Terraform utilise un mélange de variables fournies et produites via des sorties pour créer la recette de déploiement.

La force de Terraform est de planifier l'exécution en fonction des informations requises, c'est un ordonnancement par les variables. 

---

Les objets Terraform sont tous des fournisseurs de données potentielles via leurs attributs.

**Terraform sait aussi exporter certains attributs pour leur réutilisation : ce sont les `outputs`.**

Ces variables, outputs, attributs de ressources sont parfois sensibles (sensitive) car il s'agit de secrets.

---

## Les variables en entrées

Dans Terraform, les variables et les locals sont deux concepts importants pour définir des valeurs qui peuvent être réutilisées dans la configuration.

---

#### Les variables 

**Les variables sont des valeurs qui peuvent être utilisées dans une configuration Terraform.**

Elles peuvent être définies dans le fichier principal de configuration, ou dans un fichier séparé.

Les variables peuvent être de différents types, tels que des chaînes de caractères, des nombres, des booléens ou des listes.

Les variables peuvent également être utilisées pour des valeurs sensibles telles que les clés d'accès à des API.

Les variables peuvent être définies à l'aide de la syntaxe suivante :

```coffeescript

variable "nom_de_variable" {
  type = type_de_variable
  default = valeur_par_defaut
}

```
---

**Lorsque la configuration est appliquée, Terraform demande à l'utilisateur de fournir les valeurs pour les variables non définies.**

Attribuer des valeurs de variable avec l'argument par défaut n'est pas une bonne idée car cela ne facilite pas la réutilisation du code. 

Une meilleure façon de définir des valeurs de variable consiste à utiliser un fichier de définition de variables, qui est tout fichier `terraform.tfvars` ou `terraform.tfvars.json`.

Un fichier de définition de variables utilise la même syntaxe que le code de configuration Terraform, mais se compose exclusivement d'affectations de variables.

```coffeescript

## file: terraform.tfvars
default_cidr = 10.4.0.0/24

```

---

**On peut spécifier à Terraform des variables ou fichiers de variable spécifiques en ligne de commande.**

```bash

$ terraform apply -var="image_id=ami-abc123"
$ terraform apply -var-file="testing.tfvars"

```

---

**Terraform charge par défaut tous les fichiers nommés :**
* `terraform.tfvars` or `terraform.tfvars.json`
* `*.auto.tfvars` or `*.auto.tfvars.json`

---

**On peut également affecter les variables via des variables d'environnement.**

On utilise la syntaxe `TF_VAR_{ma variable}`.

```bash

$ export TF_VAR_image_id=ami-abc123
$ export TF_VAR_availability_zone_names='["us-west-1b","us-west-1d"]'

```

---

**Terraform charge les variables dans l'ordre suivant, les sources ultérieures ayant priorité sur les précédentes :**

* Variables d'environnement
* Le fichier `terraform.tfvars`, s'il existe.
* Le fichier `terraform.tfvars.json`, s'il est présent.
* Tous les fichiers `*.auto.tfvars` ou `*.auto.tfvars.json`, traités dans l'ordre lexical de leurs noms de fichiers.
* Toutes les options `-var` et `-var-file` sur la ligne de commande, dans l'ordre dans lequel elles sont fournies. (Cela inclut les variables définies par un espace de travail Terraform Cloud.)
---

**Les variables sont référencées dans le code en utilisant la syntaxe `var.nom_de_variable`.**

Il s'agit d'une expression comme une autre, on peut accéder à des items d'un objet complexe ou lui appliquer une fonction.

---

## Les locals

**Les locals sont des valeurs calculées dans une configuration Terraform.**

Ils sont similaires aux variables, mais sont utilisés pour éviter de répéter certaines variables dans la recette.


**Attention, les locals définis dans le fichier sont utilisés dans le même fichier et ne sont pas accessibles en dehors.**

---

**Les locals sont définis à l'aide de la syntaxe suivante**

```coffeescript

locals {
  nom_de_local = expression,
  [CONFIG ...]
}
```

L'expression peut utiliser des variables, d'autres locals et des fonctions Terraform.

```coffeescript

locals {
  # Ids for multiple sets of EC2 instances, merged together
  instance_ids = concat(aws_instance.blue.*.id, aws_instance.green.*.id)
}

```

---

**Les locals peuvent être référencés dans le code en utilisant la syntaxe `local.nom_de_local`.**

Comme d'habitude ce sont des expressions donc on peut appliquer des fonctions ou accéder à des éléments de leur structure.

---

## Les outputs 

**Les `outputs` (valeurs de sortie) rendent les informations sur votre infrastructure disponibles sur la ligne de commande et peuvent exposer des informations que d'autres configurations Terraform peuvent utiliser.**

Les valeurs de sortie ont plusieurs utilisations :

* Afficher certains résultats de la recette après `terraform apply`
* Exporter certaines variables pour les modules  


---

**Chaque valeur de sortie doit être déclarée à l'aide d'un bloc de sortie.**

```coffeescript

output "instance_ip_addr" {
   value = aws_instance.server.private_ip
}

```
Les sorties ne sont disponibles que lorsque Terraform applique votre plan.

---

## Les données sensibles 

**Les `variables` et les `outputs` peuvent être définis comme sensibles afin de masquer leur contenu.**

```coffeescript

variable "user_password" {
  type = string
  sensitive = true
}

output "db_password" {
  value       = aws_db_instance.db.password
  description = "The password for logging in to the database."
  sensitive   = true
}

```
Terraform masquera les valeurs marquées comme sensibles dans les messages de `terraform plan` et `terraform apply`. 

**Les données sensibles sont accessibles en clair dans l'état de Terraform.**

