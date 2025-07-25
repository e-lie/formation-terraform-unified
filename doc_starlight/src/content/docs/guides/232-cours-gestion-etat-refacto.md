---
title: "\"Cours - Gestion de l'état et refactorisation\""
description: "Guide \"Cours - Gestion de l'état et refactorisation\""
sidebar:
  order: 232
---


## Objectifs 
- Savoir gérer l'état de Terraform pour les pratiques DevOps
- Stocker et partager l'état dans une équipe

---

## Quand les problèmes commencent

**Dans la pratique DevOps, la capacité à agir librement avec des responsabilités au sein d'une équipe est la clé.** 

Mais l'état Terraform étant un fichier statique introduit plusieurs problème.

---

#### Travail en commun

**Si j'ai plusieurs partenaires dans le projet, il faut qu'on puisse travailler tous sur la même infrastructure.**

On a donc besoin de partager le fichier d'état.

**Je peux certes ajouter le fichier tfstate au dépôt git, mais c'est une mauvaise formule, car il est quasi certain qu'à un moment ça ne marchera pas.** 
* deux personnes vont lancer une commande en simultané (lock)
* quelqu'un va oublier de commit
* quelqu'un va utiliser un état ancien
* des secrets vont se retrouver accessibles 

---

#### Drift 

**Le dérapage (drift) de l'infrastructure survient lorsqu'il existe une différence entre ce qui se trouve dans le tfstate et ce qui est déployé.** 
* dans le fichier d'état 
* et dans les clouds des fournisseurs  

Le drift peut survenir
* à cause d'opérations manuelles 
* en utilisant un tfstate périmé
* en raison d'incidents techniques

---

#### Environnements 

**Le DevOps recommande évidemment d'avoir plusieurs environnements aussi similaires les uns des autres que possible.**

Chaque environnement représente une infrastructure spécifique, et donc un fichier d'état différent.


--- 
## Les solutions 

#### Les isolements du fichier d'état 

**L'isolement du fichier d'état dans Terraform fait référence à la pratique de séparer les fichiers d'état de Terraform selon les environnements.** 

L'isolement du fichier d'état permet de gérer les déploiements Terraform à grande échelle. 

Les différentes équipes travaillent sur des fichiers d'état isolés les uns des autres pour différents environnements et applications.

---

**Il y a deux façons courantes d'isoler le fichier d'état dans Terraform.**


###### Isolement via les espaces de travail 

**Les espaces de travail (workspaces en anglais) sont des environnements isolés dans Terraform.** 

Par défaut Terraform crée un workspace nommé `default`.

```bash

$ terraform workspace show

```

---

**On peut générer autant de workspaces que requis par le nombre d'environnements.**

```bash

$ terraform workspace new test1
$ terraform workspace select test1
```

Chaque espace de travail possède son propre fichier d'état géré séparément. 

Créer un nouveau workspace va déployer une nouvelle architecture d'une même configuration Terraform pour ce nouvel environnement.

--- 

**Cependant les workspaces ne sont pas des solutions idéales**

**Les fichiers d'état de tous vos espaces de travail sont stockés dans le même backend (par exemple, le même bucket S3).**

Cela signifie que vous utilisez les mêmes contrôles d'authentification et d'accès pour tous les espaces de travail, ce qui est l'une des principales raisons pour lesquelles les espaces de travail ne sont pas un mécanisme approprié pour isoler les environnements (par exemple, isoler la mise en scène de la production).

**Les espaces de travail ne sont pas visibles dans le code ou sur le terminal, sauf si vous exécutez les commandes de `terraform workspace`.**

Lorsque vous parcourez le code, un module qui a été déployé dans un espace de travail ressemble exactement à un module déployé dans 10 espaces de travail. Cela rend la maintenance plus difficile, car vous n'avez pas une bonne image de votre infrastructure.

**En réunissant les deux éléments précédents, le résultat est que les espaces de travail peuvent être assez sujets aux erreurs.**

Le manque de visibilité permet d'oublier facilement dans quel espace de travail vous vous trouvez et de déployer accidentellement des modifications dans le mauvais (par exemple, exécuter accidentellement `terraform destroy` dans un espace de travail de "production" plutôt qu'un espace de travail de "dev").

Parce que vous devez utiliser le même mécanisme d'authentification pour tous les espaces de travail, vous n'avez pas d'autres couches de défense pour vous protéger contre de telles erreurs.

On va voir plus loin avec Terragrunt comment trouver une solution à ce problème.

--- 

#### Les backends 

Documentation : 
- https://developer.hashicorp.com/terraform/language/settings/backends/configuration

**La meilleure façon de partager les fichiers d'état consiste à utiliser la prise en charge intégrée de Terraform pour les backends distants.**

Un backend Terraform détermine la façon dont Terraform charge et stocke l'état. Le backend par défaut, que vous avez utilisé tout ce temps, est le backend local, qui stocke le fichier d'état sur votre disque local.

Les backends distants vous permettent de stocker le fichier d'état chez Amazon S3, Azure Storage, Google Cloud Storage et Terraform Cloud / Enterprise de HashiCorp.

**La mise en oeuvre est simple (en apparence).**

```coffeescript

terraform {
  backend "<BACKEND_NAME>" {
    [CONFIG...]
  }
}

```

---

#### Les avantages des backends 

**Après avoir configuré un backend distant, Terraform chargera automatiquement le fichier d'état à partir de ce backend chaque fois que vous exécuterez un plan ou appliquerez.**

Il stockera automatiquement le fichier d'état dans ce backend après chaque application, il n'y a donc aucun risque d'erreur manuelle.

--- 

###### Verrouillage

**Un bon backend distant prend en charge nativement le verrouillage.**

Pour exécuter terraform apply, Terraform acquiert automatiquement un verrou ; si quelqu'un d'autre est déjà en cours d'exécution, il aura déjà le verrou et vous devrez attendre.

Vous pouvez exécuter apply avec le paramètre -lock-timeout=(TIME) pour demander à Terraform d'attendre jusqu'à TIME pour qu'un verrou soit libéré (par exemple, -lock-timeout=10m attendra 10 minutes).

---

###### Secrets

**Un bon backend distant prend en charge nativement le chiffrement en transit et le chiffrement au repos du fichier d'état.**

De plus, ces backends exposent généralement des moyens de configurer les autorisations d'accès (par exemple, en utilisant des politiques IAM avec un compartiment Amazon S3), afin que vous puissiez contrôler qui a accès à vos fichiers d'état et aux secrets qu'ils peuvent contenir.

Il serait encore préférable que Terraform prenne en charge nativement le chiffrement des secrets dans le fichier d'état, mais ces backends distants réduisent la plupart des problèmes de sécurité, étant donné qu'au moins le fichier d'état n'est stocké en texte brut sur le disque nulle part.


--- 

###### Prise en compte des workspaces

**Un bon backend distant prend en charge nativement la gestion des workspaces.**

Documentation : 
- https://developer.hashicorp.com/terraform/language/state/workspaces

Ces backends prennent en charge plusieurs espaces de travail nommés, ce qui permet d'associer plusieurs états à une seule configuration. 

La configuration n'a toujours qu'un seul backend, mais vous pouvez déployer plusieurs instances distinctes de cette configuration sans configurer un nouveau backend ni modifier les informations d'authentification.


Sans workspace, Terraform sauverait le fichier d'état dans 

    s3://bucket/key
 
Avec un workspace, cela devient automatiquement

    s3://bucket/env:/$workspace/key 



--- 


## Rappel des objectifs 
- Savoir gérer l'état de Terraform pour les pratiques DevOps
- Stocker et partager l'état dans une équipe


