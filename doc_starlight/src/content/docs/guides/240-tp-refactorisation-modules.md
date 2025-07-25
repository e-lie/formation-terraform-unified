---
title: "TP partie 9 - Modules et gestion d'état avancée"
description: "Guide TP partie 9 - Modules et gestion d'état avancée"
sidebar:
  order: 240
---


Dans cette neuvième partie, nous allons apprendre à refactoriser notre infrastructure en modules Terraform et maîtriser les techniques avancées de gestion d'état : `terraform state mv` et `terraform import`.

## Problématiques de la refactorisation

Quand on refactorise du code Terraform existant, on se heurte à un problème fondamental : **Terraform suit les ressources par leur adresse dans l'état**. Si vous avez une ressource déclarée directement puis que vous la déplacez dans un module, Terraform ne reconnaît pas que c'est la même ressource ! Il va vouloir détruire l'ancienne et créer une nouvelle, ce qui peut entraîner une interruption de service.

Prenons un exemple concret. Une ressource déclarée directement :
```coffee
resource "aws_instance" "web_server" {
  # configuration...
}
```

Devient, après refactorisation en module :
```coffee
module "webserver" {
  source = "./modules/webserver"
  # ...
}
```

La ressource prend alors l'adresse `module.webserver.aws_instance.web_server[0]` dans l'état. C'est exactement ce qu'on veut éviter en production : une destruction/recréation non planifiée.

## terraform state mv

La commande `terraform state mv` permet de déplacer des ressources dans l'état sans les détruire/recréer.

### Préparation de la refactorisation

Pour cette démonstration, nous allons partir d'un déploiement existant de la partie 8. Commençons par nous assurer que l'infrastructure est déployée :

```bash
# Assurez-vous d'avoir un état déployé depuis part8
cd part8_count_loadbalancer
terraform workspace select multi-server
terraform plan -var-file="multi-server.tfvars" -out=tfplan
terraform apply tfplan

# Listez les ressources actuelles
terraform state list
```

Cette commande vous montrera toutes les ressources déployées, organisées de manière monolithique. Vous devriez voir des ressources comme `aws_instance.web_server[0]`, `aws_lb.main[0]`, `aws_vpc.main`, etc.

### Préparation de la refactorisation

Nous allons effectuer la refactorisation directement dans le projet part8. Cette approche est plus réaliste car elle utilise le backend S3 configuré :

```bash
# Travaillons directement depuis la part8 (sur le résultat de tp précédent)
# faites un commit du résultat avant

# Créons d'abord une sauvegarde de sécurité
terraform state pull > terraform.tfstate.backup-$(date +%Y%m%d-%H%M%S)

# Copions les modules depuis part9
cp -r ../part9_refactorisation_modules/modules .

# Remplaçons directement les fichiers par leurs versions modulaires
cp ../part9_refactorisation_modules/main.tf .
cp ../part9_refactorisation_modules/outputs.tf .

# Supprimons les anciens fichiers maintenant obsolètes
rm vpc.tf webserver.tf loadbalancer.tf
```


### Commentons le code modularisé

Notre nouvelle configuration `main.tf` est maintenant beaucoup plus lisible et organisée. Au lieu d'avoir toutes les ressources déclarées au même endroit, nous utilisons trois modules distincts :

```coffee
# Module VPC
module "vpc" {
  source = "./modules/vpc"
  
  vpc_cidr             = var.vpc_cidr
  public_subnet_cidr   = var.public_subnet_cidr
  public_subnet_cidr_2 = var.public_subnet_cidr_2
  workspace            = terraform.workspace
  feature_name         = var.feature_name
  instance_count       = var.instance_count
}

# Module Webserver
module "webserver" {
  source = "./modules/webserver"
  
  instance_count      = var.instance_count
  instance_type       = var.instance_type
  subnet_id           = module.vpc.public_subnet_ids[0]
  security_group_id   = module.vpc.web_servers_security_group_id
  ssh_key_path        = var.ssh_key_path
  workspace           = terraform.workspace
  feature_name        = var.feature_name
}

# Module Load Balancer
module "loadbalancer" {
  source = "./modules/loadbalancer"
  
  instance_count       = var.instance_count
  workspace            = terraform.workspace
  feature_name         = var.feature_name
  vpc_id               = module.vpc.vpc_id
  subnet_ids           = module.vpc.public_subnet_ids
  security_group_ids   = module.vpc.alb_security_group_ids
  instance_ids         = module.webserver.instance_ids
}
```

**Points clés de cette approche modulaire :**

**Séparation des responsabilités :** Chaque module a une responsabilité claire - le VPC gère le réseau, webserver gère les instances, loadbalancer gère la répartition de charge.

**Communication entre modules :** Notez comment les modules communiquent entre eux via les outputs. Par exemple, `module.vpc.public_subnet_ids[0]` permet au module webserver d'utiliser le subnet créé par le module VPC.

**Réutilisabilité :** Ces modules peuvent maintenant être réutilisés dans d'autres projets en changeant simplement les variables d'entrée.

**Lisibilité :** Le fichier principal montre clairement l'architecture globale sans se perdre dans les détails de chaque ressource.


### Test avant migration

Nos modules sont déjà créés dans `modules/` et notre configuration utilise déjà ces modules. Voyons ce que Terraform pense de nos changements :

```bash
# Terraform inir est nécessaire quand on créé de nouveau modules
terraform init

# Voyons ce que Terraform pense de nos changements
terraform plan -var-file="multi-server.tfvars"
```

Terraform va montrer qu'il veut détruire toutes les ressources existantes et en créer de nouvelles. C'est exactement le problème qu'on veut résoudre ! La solution est d'utiliser `terraform state mv` pour déplacer les ressources vers leur nouvelle adresse dans les modules.

## Migration avec terraform state mv

Nous allons maintenant migrer chaque ressource de son ancienne adresse vers sa nouvelle adresse dans les modules. Cette opération doit être effectuée avec précaution et il est fortement recommandé de sauvegarder l'état avant de commencer.

### Migration étape par étape

La migration s'effectue en trois étapes principales : le module VPC, le module webserver, et le module loadbalancer.

**Migration du module VPC :**

```bash
# VPC principal
terraform state mv aws_vpc.main module.vpc.aws_vpc.main

# Internet Gateway
terraform state mv aws_internet_gateway.main module.vpc.aws_internet_gateway.main

# Subnets
terraform state mv aws_subnet.public module.vpc.aws_subnet.public
terraform state mv aws_subnet.public_2 module.vpc.aws_subnet.public_2

# Route table et associations
terraform state mv aws_route_table.public module.vpc.aws_route_table.public
terraform state mv aws_route_table_association.public module.vpc.aws_route_table_association.public
terraform state mv aws_route_table_association.public_2 module.vpc.aws_route_table_association.public_2

# Security groups
terraform state mv aws_security_group.web_servers module.vpc.aws_security_group.web_servers
terraform state mv 'aws_security_group.alb[0]' 'module.vpc.aws_security_group.alb[0]'

# Data source (sera recréé automatiquement)
terraform state mv data.aws_availability_zones.available module.vpc.data.aws_availability_zones.available
```

**Migration du module webserver :**

```bash
# Instances web
terraform state mv 'aws_instance.web_server[0]' 'module.webserver.aws_instance.web_server[0]'
terraform state mv 'aws_instance.web_server[1]' 'module.webserver.aws_instance.web_server[1]'
terraform state mv 'aws_instance.web_server[2]' 'module.webserver.aws_instance.web_server[2]'

# Data source AMI
terraform state mv data.aws_ami.custom_ubuntu module.webserver.data.aws_ami.custom_ubuntu
```

**Migration du module loadbalancer :**

```bash
# Load balancer
terraform state mv 'aws_lb.main[0]' 'module.loadbalancer.aws_lb.main[0]'

# Target group
terraform state mv 'aws_lb_target_group.web_servers[0]' 'module.loadbalancer.aws_lb_target_group.web_servers[0]'

# Target group attachments
terraform state mv 'aws_lb_target_group_attachment.web_servers[0]' 'module.loadbalancer.aws_lb_target_group_attachment.web_servers[0]'
terraform state mv 'aws_lb_target_group_attachment.web_servers[1]' 'module.loadbalancer.aws_lb_target_group_attachment.web_servers[1]'
terraform state mv 'aws_lb_target_group_attachment.web_servers[2]' 'module.loadbalancer.aws_lb_target_group_attachment.web_servers[2]'

# Listener
terraform state mv 'aws_lb_listener.web[0]' 'module.loadbalancer.aws_lb_listener.web[0]'
```

### Vérification de la migration

Une fois toutes les ressources migrées, vérifiez que l'opération s'est bien déroulée :

```bash
# Vérifiez que toutes les ressources ont été déplacées
terraform state list

# Testez le plan - il ne devrait plus y avoir de destruction/création
terraform plan -var-file="multi-server.tfvars"
```

Si tout s'est bien passé, le plan devrait montrer "No changes" ou seulement des modifications mineures ! Cela confirme que nos ressources sont maintenant correctement organisées en modules sans risque de destruction.

## terraform import

Parfois, vous avez des ressources AWS créées manuellement ou par d'autres outils que vous voulez intégrer à Terraform. C'est là qu'intervient `terraform import`.

### Exemple pratique : Importation d'un bucket S3

Supposons que vous ayez créé manuellement un bucket S3 pour stocker des logs ou des fichiers, et que vous vouliez maintenant le gérer avec Terraform.

**Création manuelle d'un bucket S3 :**

```bash
# Créons un bucket S3 avec un nom unique
aws s3 mb s3://terraform-demo-logs --region eu-west-3

# Vérifions que le bucket existe
aws s3 ls s3://terraform-demo-logs
# bucket vide -> par de sortie
```

**Déclaration dans Terraform :**

Ajoutons maintenant la déclaration de ce bucket dans notre configuration Terraform. Ajoutez cette section à votre `main.tf` :

```coffee
# Bucket S3 pour les logs (créé manuellement, à importer)
resource "aws_s3_bucket" "logs" {
  bucket = "terraform-demo-logs"  # Remplacez par votre nom de bucket
}

resource "aws_s3_bucket_versioning" "logs" {
  bucket = aws_s3_bucket.logs.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

**Importation dans Terraform :**

```bash
# Importez le bucket existant
terraform import aws_s3_bucket.logs terraform-demo-logs

# Vérifiez que l'import a fonctionné
terraform state list
terraform plan -var-file="multi-server.tfvars" -out=new-bucket.tfplan
```

Le plan devrait montrer que Terraform veut ajouter le versioning (qui n'existait pas sur le bucket créé manuellement) mais ne veut pas recréer le bucket lui-même.

**Finalisation de la configuration :**

```bash
# Appliquez les modifications pour ajouter le versioning
terraform apply new-bucket.tfplan

# Vérifiez que tout fonctionne
terraform state list | grep aws_s3_bucket.logs
```

Cette opération intègre une ressource existante dans l'état Terraform, permettant de la gérer ensuite via les fichiers de configuration. C'est particulièrement utile pour intégrer des ressources créées avant l'adoption de Terraform ou par d'autres équipes.

## Validation finale

Une fois la migration terminée, il est important de valider que tout fonctionne correctement :

```bash
# Plan final - devrait montrer "No changes"
terraform plan -var-file="multi-server.tfvars"

# Test de l'application
curl $(terraform output -raw web_url)

# Vérification des modules
terraform validate
```

## Bonnes pratiques pour la gestion d'état

Avant toute refactorisation ou manipulation d'état, il est essentiel de suivre certaines bonnes pratiques pour éviter toute perte de données ou interruption de service.

### Avant toute refactorisation

**Sauvegardez toujours l'état :**

```bash
# Sauvegarde manuelle
cp terraform.tfstate terraform.tfstate.backup-$(date +%Y%m%d-%H%M%S)

# Ou exportez l'état
terraform show -json > state-backup-$(date +%Y%m%d-%H%M%S).json
```

**Idéalement testez sur un workspace séparé :**

...donc sur une copie infra de test c'est l'avantage du cloud et de 'IaC de pouvoir faire des copies temporaire d'une infra. Mais ce n'est pas toujours possible selon les dépendances de votre système et les outils de réplication de son état s'il est stateful.

```bash
# Créez un workspace de test
terraform workspace new refactoring-test
terraform apply -var-file="multi-server.tfvars"

# Effectuez la migration sur ce workspace 
# Si ça marche, appliquez sur le workspace principal
```

**Planifiez la migration :**

```bash
# Listez toutes les ressources
terraform state list > resources-before.txt

# Après migration, comparez
terraform state list > resources-after.txt
diff resources-before.txt resources-after.txt
```

## Conclusion

Cette partie vous a montré comment refactoriser une infrastructure monolithique en modules sans interruption de service. Nous avons exploré deux techniques essentielles : `terraform state mv` pour déplacer des ressources dans l'état et `terraform import` pour intégrer des ressources existantes.

Les points clés à retenir sont la nécessité de toujours sauvegarder l'état avant une refactorisation, de tester sur un workspace séparé, et d'utiliser ces commandes pour éviter les destructions/recréations non désirées. La modularisation améliore considérablement la réutilisabilité et la maintenance des infrastructures Terraform.

Ces techniques sont essentielles pour maintenir et faire évoluer des infrastructures Terraform en production de manière sûre et contrôlée.