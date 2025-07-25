---
title: "Cours - validateurs"
description: "Guide Cours - validateurs"
sidebar:
  order: 242
---



## Les valideurs, Pre- et Post- Conditions Terraform

**Les trois manières de tester la validité des objets au sein de la recette Terraform sont assez simples d'usage, et relativement similaires.**

Documentation : 
* https://developer.hashicorp.com/terraform/language/expressions/custom-conditions


---

#### Les validateurs 

**Ils émettent des messages d'erreur si les variables fournies ne sont pas conformes.**

```coffee

variable "instance_type" {
  description = "The type of EC2 Instances to run (e.g. t2.micro)"
  type        = string

  validation {
    condition     = contains(["t2.micro", "t3.micro"], var.instance_type)
    error_message = "Only free tier is allowed: t2.micro | t3.micro."
  }
}

```

**La condition doit retourner un booléen pour être valide.**

```bash

$ terraform apply -var instance_type="m4.large"
│ Error: Invalid value for variable
│
│   on main.tf line 17:
│    1: variable "instance_type" {
│     ├────────────────
│     │ var.instance_type is "m4.large"
│
│ Only free tier is allowed: t2.micro | t3.micro.
│
│ This was checked by the validation rule at main.tf:21,3-13.

```
---
**Les validateurs sont limités à la seule validation interne de la variable, sans relation avec le reste de la recette.**

On ne peut pas utilser de sources de données externes comme d'autres variables, locals, data sources, et autres ressources par exemple.

---

#### Pre- et Post conditions

**Ces conditions permettent de faire des vérifications plus générales sur les ressources et les outputs.**

```bash
resource "aws_launch_configuration" "example" {
  image_id        = var.ami
  instance_type   = var.instance_type
  security_groups = [aws_security_group.instance.id]
  user_data       = var.user_data

  # Required when using a launch configuration with an auto scaling group.
  lifecycle {
    create_before_destroy = true
    precondition {
      condition     = data.aws_ec2_instance_type.instance.free_tier_eligible
      error_message = "${var.instance_type} is not part of the AWS Free Tier!"
    }
    postcondition {
      condition     = length(self.security_groups) > 1  # usage de self.attribute popssible
      error_message = "You must use more than one security groups!"
    }
  }
}

```

De la même manière  la condition intégrée dans un block de `lifeccyle` doit retourner un booléen.

```bash

$ terraform apply -var instance_type="m4.large"
│ Error: Resource precondition failed
│
│   on main.tf line 25, in resource "aws_launch_configuration" "example":
│   18:    condition = data.aws_ec2_instance_type.instance.free_tier_eligible
│     ├────────────────
│     │ data.aws_ec2_instance_type.instance.free_tier_eligible is false
│
│ m4.large is not part of the AWS Free Tier!

```

--- 

Comme pour les validations on peut avoir plusieurs conditions dans un même bloc   

---

**Les ressources et les Data acceptent  `precondition` et `postcondition`**

Les `precondition` sont vérifiées au moment où l'objet est construit, avec ses éventuels paramètres de `count` ou `for_each`.

Cela permet de s'assurer qu'on ne construit pas une donnée ou une ressource incorrecte.

Les `postcondition` sont vérifiées après la planification et l'obtention de l'objet.

Cela évite les dépendances envers un objet invalide pour d'autres objets qui auraient un effet de cascade.

---

**Les outputs n'acceptent que `precondition`**

Comme les validateurs vérifient que les entrées sont conformes, les conditions sur les outputs s'assurent que les sorties du module sont conformes afin d'éviter d'ajouter dans l'état Terraform des informations incorrectes.

---

**Pour construire de bons modules, il est pratique de s'assurer qu'on a bien intégré des processus de `condition`.** 

Ce sont des garde-barrières efficaces pour les situations inattendues que peut rencontrer le code en raison de mauvaise configuration notamment.

---

## Les tests Terraform

**La pratique des tests, comme la sécurité, est un sujet complexe sur lequel on pourrait passer beaucoup de temps.**

En effet, tester le code peut impliquer de nombreux types de tests utilsant de nombreuses approches différentes. 

- Tests unitaires 
- Tests d'intégration
- Tests End To End

Mais aussi 

- Tests statiques
- Tests de performance
- Tests de sécurité

---

**Le tester du code Terraform nécessite de lancer réellement des opérations de déploiement.**

Les tests principaux consistent donc à créer des scénarios de test consistant à

* créer une infrastructure de test avec des variables spécifiques
* faire des tests sur l'infrastructure 
* détruire l'infrastructure de test 

---

#### Écrire des tests fonctionnels pour  Terraform  

**Il existe de nombreux outils pour écrire des tests qui lancent Terraform.**

Les solutions standards sont écrites en Go : 

* Terratest, par Gruntwork : https://github.com/gruntwork-io/terratest 
* terraform-exec, par Hashicorp : https://github.com/hashicorp/terraform-exec

Mais il existe des librairies dans d'autres langages 

* Python https://pypi.org/project/pytest-terraform/
* Java : https://github.com/deliveredtechnologies/terraform-maven

--- 

Si vous êtes assez à l'aise avec Go on peut visiter ce dépôt qui donne des exemples de test Terraform 

> https://github.com/brikis98/terraform-up-and-running-code/tree/503ab1f5055917f2d0c715a6b1aa0b9dfb716354/code/terraform/10-terraform-team/test

--- 

**Quand on crée un module qu'on souhaite exporter, il est préférable d'avoir des tests.** 

Ces derniers sont placés dans un dossier `test` et on les lance automatiquement dans la CI / CD du dépôt du module.

Ceci dit, 
* il est particulièrement long et compliqué d'écrire des tests pour Terraform
* il est remarquable que les modules AWS pour Terraforme n'intègrent pas de tests publics alors que des modules [de moindre renommée](https://github.com/cloudposse/terraform-null-label) le fassent
* il est aussi remarquable que même Terraform se pose [la question de la testabilité des modules Terraform](https://developer.hashicorp.com/terraform/language/modules/testing-experiment) avec la commande `terraform test`
* il est au moins aussi important d'avoir une bonne page d'examples pour rendre son module compréhensible


---

## Rappel des objectifs 
- Savoir utiliser à bon escient les mécanismes de validation et les tests dans Terraform


