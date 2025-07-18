---
title: "Cours - Provider, resource et data"
description: "Guide Cours - Provider, resource et data"
sidebar:
  order: 142
---



![](/142_cours_provider_resource_data/images/terraform-data-arguments-and-attributes.png)

---

## Les providers

**Terraform s'appuie sur des plugins appelés providers (fournisseurs) pour interagir avec les fournisseurs de cloud, les fournisseurs SaaS et d'autres API.**

Les configurations Terraform doivent déclarer les providers dont elles ont besoin pour que Terraform puisse les installer et les utiliser. 

De plus, certains fournisseurs nécessitent une configuration (comme les URL de point de terminaison ou les régions cloud) avant de pouvoir être utilisés.

---

**La liste des providers est disponible sur le Resistry Terraform.**

> https://registry.terraform.io/browse/providers

Terraform effectue des validations cryptographiques du contenu des providers téléchargés pour s'assurer qu'ils sont conformes.

Il existe des fournisseurs :
* Officiels : développés par hashicorp, namespace: hashicorp
* Partners : développés par des entreprises en relation étroite avec Hashicorp, namespace : "company"
* Community : développés par la communauté, namespace : "user"
* Archived : versions dépréciées 

---

**La documentation pour utiliser un provider est dans son code et sa page Terraform.** 

> https://registry.terraform.io/providers/hashicorp/boundary/latest/docs  
> https://registry.terraform.io/providers/hashicorp/vault/latest/docs

---

**Chaque fournisseur ajoute un ensemble de `resources types` (types de ressources) et/ou de `data sources` (sources de données) que Terraform peut gérer.**

Chaque type de ressource est implémenté par un fournisseur ; sans fournisseurs, Terraform ne peut gérer aucun type d'infrastructure.

La plupart des fournisseurs configurent une plate-forme d'infrastructure spécifique (cloud ou auto-hébergée). 

Les fournisseurs peuvent également proposer des utilitaires locaux pour des tâches telles que la génération de nombres aléatoires pour des noms de ressources uniques.

---


**Les providers sont codés en Go.**

> Ex: https://github.com/hashicorp/terraform-provider-boundary

Pour gérer le cycle de vie des ressources, ils prennent en compte les quatres méthodes CRUD

* Create
* Read
* Update 
* Delete

> ex: https://github.com/hashicorp/terraform-provider-boundary/blob/main/internal/provider/resource_group.go  

    > resourceGroup(...)   
    > resourceGroupCreate(...)   
    > resourceGroupRead(...)   
    > resourceGroupUpdate(...)   
    > resourceGroupDelete(...)    

---
**La déclaration d'un provider se fait au début de la recette.**

La déclaration suffit à déclencher l'ajout dans la liste des providers.

La configuration de chaque provider dépend de son implémentation.

Certains providers utilisent des variables d'environnement pour leur configuration afin d'éviter de stocker des secrets dans le code.

```coffeescript

## Une déclaration simple 
provider "aws" {
  region = "us-east-2"
}

## Une configuration qui utilise un alias 
provider "aws" {
  alias = "west"
  region = "us-west-2"
}

```

---

**Le lock file pour les providers permet de les versionner sous contrôle.**

```coffeescript
## This file is maintained automatically by "terraform init".
## Manual edits may be lost in future updates.

provider "registry.terraform.io/hashicorp/aws" {
  version = "4.57.0"
  hashes = [
    "h1:0bd5IKkEF1TGE4tgm0VuVMFQg2s6GOXJBU+/b/siYKw=",
    "zh:07d89ad94267b7d6285fd65fbd67f8680e111abf9bbcbcac2e30154262fbbe46",
    "zh:0eeee044e6fc285c20241d3de7f9b79450cab2df1452a9c18c0bed1090085a25",
    "zh:306ba8ac99a0d9f9eba0386cb11459323696e69dcb28bc5e55b6fb2de28640cd",
    "zh:40afc24b94e7cae387f22dd3045b09311a120e429aa4f06168d7498995a98f67",
    "zh:5a2c846a2cc463841ca2353fb734ba6f9502e662196c85fd3332a4e18acec72e",
    "zh:854fbf7d058e4e31ce4ed882e2085bd94c53be4b38b15f3b5d3d897a2c5102df",
    "zh:89a7a5e7de6400662804d5dc43251172e3f0522853dcab304d637a7bbb266654",
    "zh:89ef96a1b36396f555e80505f55fd29432be3dc518bd75b72a1aae29e8171b4a",
    "zh:9b12af85486a96aedd8d7984b0ff811a4b42e3d88dad1a3fb4c0b580d04fa425",
    "zh:b28516cc8e614fad40738ec73ce70528e2a817dae3118895333c1d63f1e22a89",
    "zh:c3f1c6a7d56b0838da2f880a74e19df65ca9006cb3ebdc34403cb9d6e4ee046d",
    "zh:d036a7355494792e2347b92d766431ba91cb399a4bd2bb719db3025542c0e674",
    "zh:d3299b9507085238aaf24f38faffb5d6226a31f916e47320b31d728c7062be16",
    "zh:d9f5c04f4648d593d91be2a66c6c61f6c52512c78ac6fb3077f0911a6b95fa2f",
    "zh:f84143ee0cff2ad0af8ad40074fb5dcd83bfb8a514c7f43ca23c7422caa42330",
  ]
}

```

Les hashs sont tirés des zips qui constituent l'archive 

* zh : "Zip Hash" (legacy) sha256 des fichiers zip
* h1 : "Hash Scheme 1" (prefered) sha256 des contenus 

**On peut mettre à jour les dépendances de providers avec la commande suivante.**

```bash

$ terraform init -upgrade

```

L'ajout à git du fichier de lock permet de gérer collaborativement le versionnage des providers. 

---

## Les data sources 

**Un fournisseur peut exposer des datas sources, soit des informations sur des types d'objets.**

La syntaxe HCL pour déclarer une data contient le nom du provider qui la fournit :

```coffeescript
data "<PROVIDER>_<DATA_SOURCE>" "<NAME>" {
    [CONFIG ...]
}

```
La plupart des éléments du corps d'un bloc de données sont définis par et spécifiques à la source de données sélectionnée, et ces arguments peuvent tirer pleinement parti des expressions et d'autres fonctionnalités dynamiques du langage Terraform.


```coffeescript

data "aws_ami" "ubuntu" {
  most_recent = true

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }

  owners = ["099720109477"]
}

```

Attention, l'argument `filter` dans ce bloc n'est pas générique : il dépend directement de l'implémentation fournie par le provider.

---

**Une fois l'appel effectué, la valeur demandée est disponible pour être utilisée dans des expressions HCL.**

```coffeescript

  id = data.aws_ami.ubuntu.id 

```
---

## Les resources 

**Chaque fournisseur peut exposer des ressources, soit des consommations types d'objets.**

Ces objets peuvent être des instances, des bases de données, des noms de domaines, ou toute autre structure utile au montage d'une infrastructure.

La syntaxe HCL pour déclarer une ressource contient le nom du provider qui la fournit :

```coffeescript
resource "<PROVIDER>_<TYPE>" "<NAME>" {
  [CONFIG ...]
}
```

---
**Chaque ressource a ses propres paramètres définis par le provider.**

```coffeescript
resource "aws_instance" "example" {
  ami           = "ami-0fb653ca2d3203ac1" # une référence d'Image AWS
  instance_type = "t2.micro"              # une référence de type de machine AWS
}
```

Il y a souvent beaucoup d'arguments dans une ressource.

```coffeescript
  # aws_instance.example will be created
  + resource "aws_instance" "example" {
      + ami                          = "ami-0fb653ca2d3203ac1"
      + arn                          = (known after apply)
      + associate_public_ip_address  = (known after apply)
      + availability_zone            = (known after apply)
      + cpu_core_count               = (known after apply)
      + cpu_threads_per_core         = (known after apply)
      + get_password_data            = false
      + host_id                      = (known after apply)
      + id                           = (known after apply)
      + instance_state               = (known after apply)
      + instance_type                = "t2.micro"
      + ipv6_address_count           = (known after apply)
      + ipv6_addresses               = (known after apply)
      + key_name                     = (known after apply)
      (...)

```

**Certains paramètres sont déclarés par l'utilisateur mais d'autres nécessitent une instanciation de la ressource demandée.**

Par exemple, l'adresse IP n'est connue que lorsque l'instance est effectivement commandée et disponible.

