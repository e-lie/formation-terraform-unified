---
title: "Cours - Qu'est ce que larchitecture en modules de Terraform"
description: "Guide Cours - Qu'est ce que larchitecture en modules de Terraform"
sidebar:
  order: 234
---



## Les différentes formes de structuration du code 

**Avant d'aborder les modules, voyons quels sont les structures utilisées pour organiser le code.**

#### La structure "monolithique"

Tout le code est dans un seul fichier, sans séparation entre responsabilités.

```coffee
prod
├─ outputs.tf 
├─ main.tf 
└─ variables.tf 
```

Rapidement elle peut devenir insuffisante.

---

#### La structure "flat"

Tout le code est dans un seul dossier, avec chaque partie "logique" de l'infrastructure dans un fichier distinct.

```coffeescript
prod
├─ database.tf 
├─ load-balancer.tf 
├─ outputs.tf 
├─ variables.tf 
└─ webserver.tf 
```

Elle résulte d'une analyse des composants logiques de la recette.

---

#### La structure en recettes séparées

**Les différents composants sont séparés dans des dossiers différents.**

```coffeescript
└── prod
    ├── webserver
    │   ├── main.tf
    │   ├── outputs.tf
    │   └── variables.tf
    └── database
        ├── main.tf
        ├── outputs.tf
        └── variables.tf
```

Elle permet de séparer les problèmes, mais elle implique une duplication des ressources avec par exemple des dossiers `.terraform` distincts.

---

#### La structure "liens symboliques"

**On déploie le code dans plusieurs dossiers, avec les fichiers communs définis comme des liens symboliques.**

```coffeescript
├── dev
│   ├── infra_override.tf
│   ├── infra.tf -> ../common/infra.tf
│   ├── outputs.tf -> ../common/outputs.tf
│   ├── state.tf
│   ├── terraform.tfvars
│   └── variables.tf -> ../common/variables.tf
└── common
    ├── infra.tf
    ├── outputs.tf
    └── variables.tf
```

Cette structure fonctionne en chargeant les variables locales et en surchargeant des parties de la recette avec des fichiers via `_override.tf`

_À noter également, le système de fichier de Windows ne supporte pas historiquement les liens symboliques, et il faut ajouter le mode developpeur pour y accéder._



---
        
## Les modules dans Terraform 

Documentation:
- https://developer.hashicorp.com/terraform/language/modules

**Les modules sont des packages de code autonomes qui vous permettent de créer des composants réutilisables en regroupant des ressources associées.**

C'est un compossant essentielle de toute architecture d'infrastructure logicielle Terraform mature, car les modules offrent de nombreux avantages

- Lisibilité 
- Réusabilité
- Testabilité

Vous n'avez pas besoin de savoir comment fonctionne un module pour pouvoir l'utiliser ; il suffit de savoir paramétrer les entrées et les sorties. Les modules sont des outils utiles pour promouvoir l'abstraction logicielle et la réutilisation du code.

--- 

**On va utiliser un exemple standard d'infrastructure modulaire.** 

> TP2.05-modules

Dans le premier `main.tf`, notez l'usage de 

```coffeescript

module "<NAME>" {
  source = "<SOURCE>"

  [CONFIG ...]
}

```

On observe que ce module est principalement composé d'autres modules. 

Ce modèle est connu sous le nom de composition logicielle : la pratique consistant à décomposer un code volumineux et complexe en sous-systèmes plus petits.

---

**HashiCorp recommande fortement que chaque module suive certaines conventions de code connues sous le nom de [structure de module standard](https://www.terraform.io/docs/modules/index.html#standard-module-structure).** 

Au minimum, cela signifie avoir trois configurations Terraform fichiers de figuration par module.

* main.tf—le point d'entrée principal
* outputs.tf—déclarations pour toutes les valeurs de sortie
* variables.tf—déclarations pour toutes les variables d'entrée

--- 

**Le module racine est le module de niveau supérieur.** 

C'est là que les variables d'entrée fournies par l'utilisateur sont configurées et que les commandes Terraform telles que `terraform init` et `terraform apply` sont exécutées.

Le module racine ne fait pas grand-chose à part déclarer des modules de composants et leur permettre de transmettre des données entre eux.

**Du point de vue du module racine, les modules sont des fonctions avec des effets secondaires (c'est-à-dire des fonctions non pures), qui sont les ressources provisionnées à la suite de l'application de terraform.**

--- 

**Nous pouvons voir que les sorties de chaque module "s'écoulent" via le module racine qui fait appel à leur outputs.**

```coffeescript

module.<MODULE_NAME>.<OUTPUT_NAME>

```

Le module avec le moins de dépendances est appelé en premier, et ses sorties sont transmises au module avec le plus de dépendances (l'équilibreur de charge) par le module racine.

---


#### Bonnes pratiques pour le module racine  

Il est conseillé d'intégrer certains fichiers sont dans le module racine : 
* versions.tf 

```coffeescript

terraform {
  required_version = ">= 0.14"
  required_providers {
    aws = "= 3.28"
  }
}

```

* providers.tf
```coffeescript

provider "google" {
  project = var.project_id
  region = var.region
}

```
* README.md

Ces fichiers définissent la documentation et configuration de base de votre recette.

Cependant rien n'interdit un module d'avoir sa propre documentation ou de surcharger un provider (par exemple pour utiliser une version différente).

---

## Sources des modules 

**On peut utiliser les modules disponibles sur le registre Terraform, créer les siennes et utiliser un mélange des deux.**

--- 
#### Les modules du Registry 

**Il existe de très nombreux modules qui couvrent à peu près tous les usages sur le Registry.**

> https://registry.terraform.io/browse/modules

On les appelle en mentionnant une source "absolue" et terraform les télécharge avec les providers quand on initie le projet.

```coffeescript

module "alb" {
  source = "terraform-aws-modules/alb/aws"
  ...
}
```

De la même manière qu'avec les providers, on peut bloquer les versions des modules.

Souvent, les modules du Registry utiliseront eux-mêmes des modules : c'est ce qu'on appelle des modules imbriqués (nested).

---

#### Créer ses propres modules 

**La plupart du temps on crée ses modules au sein de son projet, la syntaxe dans ce cas fait appel à un dossier relatif.**

```coffee

module "webservers" {
  source = "./modules/webservers"
}

```

On peut ensuite ajouter ce module dans le Registry Terraform, et en réalité on peut aussi charger des modules depuis Github, des buckets S3 et encore d'autres.

---

**Créer un module Terraform est très simple : tout ensemble de fichiers de configuration Terraform dans un dossier est un module.**

Toutes les configurations que vous avez écrites jusqu'à présent étaient techniquement des modules, bien que pas particulièrement intéressants, puisque vous les avez déployés directement : si vous exécutez apply directement sur un module, il est appelé module racine.

Pour voir de quoi les modules sont réellement capables, vous devez créer un module réutilisable, c'est-à-dire un module destiné à être utilisé dans d'autres modules.

--- 

#### Structurer le module 

**Comme on l'a vu la norme est de mettre** 

* La génération de ressources dans le fichier `main.tf`
* Les inputs dans le fichier `variables.tf`
* Les outputs dans le fichier `outputs.tf`

--- 

#### Rendre son module customisable

**La bonne gestion des variables du module est essentielle, car la capacité à pouvoir être réutilisable dépend d'elle.**

Il faut donc essayer de multiplier intelligemment les variables, avec des défauts intelligents.

--- 

#### Éviter les pièges

**Le premier piège apparaît quand on fait appel à des fonctions utilisant les fichiers locaux, comme `template`.**

Terraform utilise le dossier courant comme référence par défaut des chemins relatifs, mais ça ne fonctionnera plus quand vous serez dans un module.

On utilise donc des variables spéciales `path.*` qui permettent à Terraform de charger les fichiers correctement.

```coffeescript

  user_data = templatefile("${path.module}/user-data.sh", {
    server_port = var.server_port
    db_address  = data.terraform_remote_state.db.outputs.address
    db_port     = data.terraform_remote_state.db.outputs.port
  })

```

---

**Le deuxième piège survient avec les déclarations de bloc dans les ressources.**

Avec la déclaration suivante dans un module, il est impossible de surcharger le `ingress` déclaré dans le module, car le bloc est déterminé.

```coffeescript

resource "aws_security_group" "alb" {
  name = "${var.cluster_name}-alb"

  ingress {
    from_port   = local.http_port
    to_port     = local.http_port
    protocol    = local.tcp_protocol
    cidr_blocks = local.all_ips
  }
  ...
}

```

On utillise donc des ressources séparées, qui seront aggrégées au moment de l'exécution. 

C'est pourquoi on trouve dans les listes d'objets mis à disposition par les providers une myriade de sous-éléments.

```coffee
resource "aws_security_group" "alb" {
  name = "${var.cluster_name}-alb"
}

resource "aws_security_group_rule" "allow_http_inbound" {
  type              = "ingress"
  security_group_id = aws_security_group.alb.id

  from_port   = local.http_port
  to_port     = local.http_port
  protocol    = local.tcp_protocol
  cidr_blocks = local.all_ips
}
```

En exportant le Security Group...

```coffee
output "alb_security_group_id" {
  value       = aws_security_group.alb.id
  description = "The ID of the Security Group attached to the load balancer"
}

```

On peut le réutiliser pour ajouter de nouvelles règles hors module

```coffee
module "webservers" {
  source = "./modules/webservers"
  ...
}

resource "aws_security_group_rule" "allow_testing_inbound" {
  type              = "ingress"
  security_group_id = module.webservers.alb_security_group_id

  from_port   = 12345
  to_port     = 12345
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}
```

--- 

## Rappel des objectifs 
- Comprendre les modules dans Terraform
- Savoir créer ses propres modules


