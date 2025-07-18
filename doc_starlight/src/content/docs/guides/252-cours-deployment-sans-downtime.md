---
title: "Cours - Déploiement sans downtime"
description: "Guide Cours - Déploiement sans downtime"
sidebar:
  order: 252
---



## Zero-Downtime 

**La logique Zero-Downtime consiste à s'assurer que l'application reste disponible durant un déploiement.**

Si par exemple on redéploie une flotte de webservers avec des Load Balancers, il faut éviter que des clients voient leurs requêtes web échouer.

Comment faire ? 

#### Utiliser lifecycle et create_before_destroy

**Le comportement par défault lorsqu'une ressource dite "immutable" est modifiée consiste à supprimer puis recréer la ressource avec la même configuration.**

Une ressource immutable est une ressource dont on ne peut pas changer les attributs sans en créer une nouvelle.

On peut utiliser un meta-argument nommé `lifecycle` pour changer le comportement par défaut.

L'ordre par défaut d'une configuration Terraform est Create - Delete - Update

1. Créer des ressources qui existent dans la configuration mais qui ne sont pas associées à un objet d'infrastructure réel dans l'état.
1. Détruire les ressources qui existent dans l'état mais qui n'existent plus dans la configuration.
1. Mettre à jour les ressources sur place dont les arguments ont changé.
1. Détruire et recréer les ressources dont les arguments ont changé mais qui ne peuvent pas être mis à jour sur place en raison des limitations de l'API distante.

Dans le dernier cas par défaut Terraform détruira l'objet existant, puis créera un nouvel objet de remplacement avec les nouveaux arguments configurés.

---

**Le méta-argument create_before_destroy modifie ce comportement afin que le nouvel objet de remplacement soit créé en premier et que l'objet précédent soit détruit après la création du remplacement.**

Dans l'exemple suivant un Launch Template est un modèle d'instance démarrable par un Auto Scaler.

```coffee

resource "aws_launch_template" "example" {
  image_id        = "var.image_id"
  instance_type   = "t2.micro"
  
  ...
  # Required when using a launch configuration with an auto scaling group.
  lifecycle {
    create_before_destroy = true
  }
}

```

Ainsi lorsqu'on modifie la variable `image_id`, les nouvelles instances seront créés _AVANT_ la destruction des précédentes, sans moment où il n'y aurait pas de serveur disponible pour traiter les requêtes.

---

#### Stratégies de déploiements 

**Parmi les différentes stratégies de déploiement existantes, en voici une qui marche bien pour Terraform.**

Les autres possibilités impliquent des opérations manuelles, et notons que c'est ici un avantage de Kubernetes.

---

###### Green / Blue Deployments  

**Le déploiement bleu/vert est un modèle de publication d'application qui permet de transférer progressivement le trafic utilisateur depuis la version antérieure d'une application ou d'un microservice vers une nouvelle version pratiquement identique, ces deux versions s'exécutant en même temps dans l'environnement de production.**

<!-- > Voir le TP-2.07-green-blue

Cette méthode utilise un module spécifique : 

> https://github.com/terraform-in-action/terraform-bluegreen-aws/tree/v0.1.3 -->

Cette méthode propose un remplacement complet des backends avant de basculer le load balancer.

Les autres méthodes de déploiement impliquent un remplacement progressif, voire un pilotage via deux load balancers (méthode canari).

Ces méthodes sont plus complexes et impliquent des lancements incrémentaux de l'infrastructure.

---

#### Déployer avec Docker ou Kubernetes 

**Pour déployer le code d'une application, il est toujours préférable de donner cette tâche à un  outil adapté.** 

Le déploiement d'infrastructures complexes sur la base de quelques variables est la force de Terraform. 

Kubernetes ou d'autres outils plus proches des développeurs sont plus adaptés pour les mises à jour d'application.

